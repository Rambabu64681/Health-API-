// File: src/main/java/com/ram/healthcare/HealthcareApiApplication.java
// A realistic, GitHub-ready Spring Boot mini-project: Patient + Appointment APIs (in-memory),
// basic validation, audit logging, simple search, and health endpoint.
// Create a Spring Boot project (Spring Web + Validation) and drop this file in.

package com.ram.healthcare;

import jakarta.validation.Valid;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Pattern;
import jakarta.validation.constraints.Size;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.http.ResponseEntity;
import org.springframework.util.StringUtils;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.*;

import java.time.Instant;
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.stream.Collectors;

@SpringBootApplication
public class HealthcareApiApplication {
  public static void main(String[] args) {
    SpringApplication.run(HealthcareApiApplication.class, args);
  }

  // ----------------------------
  // Domain models (records)
  // ----------------------------
  public record Patient(
      UUID id,
      @NotBlank @Size(max = 80) String firstName,
      @NotBlank @Size(max = 80) String lastName,
      @NotNull LocalDate dateOfBirth,
      @Pattern(regexp = "^\\+?[0-9(). -]{7,20}$", message = "invalid phone format") String phone,
      @Size(max = 120) String email,
      @NotBlank @Size(max = 40) String mrn,   // Medical Record Number
      @NotBlank @Size(max = 40) String status // ACTIVE / INACTIVE
  ) {}

  public record Appointment(
      UUID id,
      @NotNull UUID patientId,
      @NotNull LocalDateTime scheduledAt,
      @NotBlank @Size(max = 80) String department,
      @NotBlank @Size(max = 80) String provider,
      @NotBlank @Size(max = 30) String status // SCHEDULED / COMPLETED / CANCELED
  ) {}

  public record AuditEvent(
      UUID id,
      Instant at,
      String actor,
      String action,
      String entityType,
      String entityId,
      Map<String, Object> meta
  ) {}

  // ----------------------------
  // In-memory repositories
  // ----------------------------
  static final class PatientRepo {
    private final Map<UUID, Patient> byId = new ConcurrentHashMap<>();
    private final Map<String, UUID> mrnIndex = new ConcurrentHashMap<>();

    Patient save(Patient p) {
      mrnIndex.put(p.mrn(), p.id());
      byId.put(p.id(), p);
      return p;
    }

    Optional<Patient> findById(UUID id) { return Optional.ofNullable(byId.get(id)); }

    Optional<Patient> findByMrn(String mrn) {
      UUID id = mrnIndex.get(mrn);
      return id == null ? Optional.empty() : findById(id);
    }

    List<Patient> search(String q, String status) {
      String needle = (q == null) ? "" : q.trim().toLowerCase();
      return byId.values().stream()
          .filter(p -> !StringUtils.hasText(status) || p.status().equalsIgnoreCase(status))
          .filter(p -> needle.isEmpty()
              || (p.firstName() + " " + p.lastName()).toLowerCase().contains(needle)
              || p.mrn().toLowerCase().contains(needle))
          .sorted(Comparator.comparing(Patient::lastName).thenComparing(Patient::firstName))
          .collect(Collectors.toList());
    }

    void delete(UUID id) {
      Patient removed = byId.remove(id);
      if (removed != null) mrnIndex.remove(removed.mrn());
    }
  }

  static final class ApptRepo {
    private final Map<UUID, Appointment> byId = new ConcurrentHashMap<>();
    private final Map<UUID, Set<UUID>> byPatient = new ConcurrentHashMap<>();

    Appointment save(Appointment a) {
      byId.put(a.id(), a);
      byPatient.computeIfAbsent(a.patientId(), _k -> ConcurrentHashMap.newKeySet()).add(a.id());
      return a;
    }

    List<Appointment> listForPatient(UUID patientId) {
      Set<UUID> ids = byPatient.getOrDefault(patientId, Collections.emptySet());
      return ids.stream()
          .map(byId::get)
          .filter(Objects::nonNull)
          .sorted(Comparator.comparing(Appointment::scheduledAt))
          .collect(Collectors.toList());
    }

    Optional<Appointment> findById(UUID id) { return Optional.ofNullable(byId.get(id)); }

    void delete(UUID id) {
      Appointment a = byId.remove(id);
      if (a != null) {
        Set<UUID> ids = byPatient.get(a.patientId());
        if (ids != null) ids.remove(id);
      }
    }
  }

  static final class AuditRepo {
    private final Deque<AuditEvent> events = new ArrayDeque<>(500);

    synchronized void add(AuditEvent e) {
      if (events.size() >= 500) events.removeFirst();
      events.addLast(e);
    }

    synchronized List<AuditEvent> latest(int limit) {
      return events.stream()
          .sorted(Comparator.comparing(AuditEvent::at).reversed())
          .limit(Math.max(1, limit))
          .collect(Collectors.toList());
    }
  }

  // ----------------------------
  // API (Controllers)
  // ----------------------------
  @RestController
  @RequestMapping("/api")
  static final class HealthcareController {
    private static final Logger log = LoggerFactory.getLogger(HealthcareController.class);

    private final PatientRepo patients = new PatientRepo();
    private final ApptRepo appts = new ApptRepo();
    private final AuditRepo audit = new AuditRepo();

    // ---- Health / readiness
    @GetMapping("/health")
    public Map<String, Object> health() {
      return Map.of(
          "status", "UP",
          "service", "healthcare-api",
          "time", Instant.now().toString()
      );
    }

    // ---- Patients
    public record CreatePatientRequest(
        @NotBlank @Size(max = 80) String firstName,
        @NotBlank @Size(max = 80) String lastName,
        @NotNull LocalDate dateOfBirth,
        @Pattern(regexp = "^\\+?[0-9(). -]{7,20}$", message = "invalid phone format") String phone,
        @Size(max = 120) String email,
        @NotBlank @Size(max = 40) String mrn
    ) {}

    @PostMapping("/patients")
    public ResponseEntity<Patient> createPatient(
        @RequestHeader(value = "X-Actor", required = false) String actor,
        @Valid @RequestBody CreatePatientRequest req
    ) {
      // enforce unique MRN
      patients.findByMrn(req.mrn()).ifPresent(_p -> {
        throw new ApiException(HttpStatus.CONFLICT, "MRN_ALREADY_EXISTS", "MRN already exists: " + req.mrn());
      });

      Patient p = new Patient(
          UUID.randomUUID(),
          req.firstName().trim(),
          req.lastName().trim(),
          req.dateOfBirth(),
          req.phone(),
          req.email(),
          req.mrn().trim(),
          "ACTIVE"
      );
      patients.save(p);

      audit("CREATE", "PATIENT", p.id().toString(), actor, Map.of("mrn", p.mrn()));
      log.info("Patient created id={} mrn={}", p.id(), p.mrn());
      return ResponseEntity.status(HttpStatus.CREATED).body(p);
    }

    @GetMapping("/patients/{id}")
    public Patient getPatient(@PathVariable UUID id) {
      return patients.findById(id).orElseThrow(() ->
          new ApiException(HttpStatus.NOT_FOUND, "PATIENT_NOT_FOUND", "Patient not found: " + id));
    }

    @GetMapping("/patients")
    public List<Patient> searchPatients(
        @RequestParam(value = "q", required = false) String q,
        @RequestParam(value = "status", required = false) String status
    ) {
      return patients.search(q, status);
    }

    public record UpdatePatientStatusRequest(@NotBlank String status) {}

    @PatchMapping("/patients/{id}/status")
    public Patient updatePatientStatus(
        @RequestHeader(value = "X-Actor", required = false) String actor,
        @PathVariable UUID id,
        @Valid @RequestBody UpdatePatientStatusRequest req
    ) {
      Patient existing = getPatient(id);
      String newStatus = req.status().trim().toUpperCase();
      if (!Set.of("ACTIVE", "INACTIVE").contains(newStatus)) {
        throw new ApiException(HttpStatus.BAD_REQUEST, "INVALID_STATUS", "Allowed: ACTIVE, INACTIVE");
      }
      Patient updated = new Patient(
          existing.id(),
          existing.firstName(),
          existing.lastName(),
          existing.dateOfBirth(),
          existing.phone(),
          existing.email(),
          existing.mrn(),
          newStatus
      );
      patients.save(updated);

      audit("UPDATE_STATUS", "PATIENT", id.toString(), actor, Map.of("status", newStatus));
      return updated;
    }

    @DeleteMapping("/patients/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void deletePatient(
        @RequestHeader(value = "X-Actor", required = false) String actor,
        @PathVariable UUID id
    ) {
      // also remove appointments
      appts.listForPatient(id).forEach(a -> appts.delete(a.id()));
      patients.delete(id);
      audit("DELETE", "PATIENT", id.toString(), actor, Map.of());
    }

    // ---- Appointments
    public record CreateAppointmentRequest(
        @NotNull UUID patientId,
        @NotNull LocalDateTime scheduledAt,
        @NotBlank @Size(max = 80) String department,
        @NotBlank @Size(max = 80) String provider
    ) {}

    @PostMapping("/appointments")
    public ResponseEntity<Appointment> createAppointment(
        @RequestHeader(value = "X-Actor", required = false) String actor,
        @Valid @RequestBody CreateAppointmentRequest req
    ) {
      // ensure patient exists
      getPatient(req.patientId());

      Appointment a = new Appointment(
          UUID.randomUUID(),
          req.patientId(),
          req.scheduledAt(),
          req.department().trim(),
          req.provider().trim(),
          "SCHEDULED"
      );
      appts.save(a);

      audit("CREATE", "APPOINTMENT", a.id().toString(), actor, Map.of("patientId", a.patientId().toString()));
      return ResponseEntity.status(HttpStatus.CREATED).body(a);
    }

    @GetMapping("/patients/{patientId}/appointments")
    public List<Appointment> listAppointments(@PathVariable UUID patientId) {
      // ensure patient exists
      getPatient(patientId);
      return appts.listForPatient(patientId);
    }

    public record UpdateAppointmentStatusRequest(@NotBlank String status) {}

    @PatchMapping("/appointments/{id}/status")
    public Appointment updateAppointmentStatus(
        @RequestHeader(value = "X-Actor", required = false) String actor,
        @PathVariable UUID id,
        @Valid @RequestBody UpdateAppointmentStatusRequest req
    ) {
      Appointment existing = appts.findById(id).orElseThrow(() ->
          new ApiException(HttpStatus.NOT_FOUND, "APPOINTMENT_NOT_FOUND", "Appointment not found: " + id));

      String newStatus = req.status().trim().toUpperCase();
      if (!Set.of("SCHEDULED", "COMPLETED", "CANCELED").contains(newStatus)) {
        throw new ApiException(HttpStatus.BAD_REQUEST, "INVALID_STATUS", "Allowed: SCHEDULED, COMPLETED, CANCELED");
      }

      Appointment updated = new Appointment(
          existing.id(),
          existing.patientId(),
          existing.scheduledAt(),
          existing.department(),
          existing.provider(),
          newStatus
      );
      appts.save(updated);

      audit("UPDATE_STATUS", "APPOINTMENT", id.toString(), actor, Map.of("status", newStatus));
      return updated;
    }

    // ---- Audit trail (nice for healthcare compliance demos)
    @GetMapping("/audit")
    public List<AuditEvent> audit(@RequestParam(value = "limit", defaultValue = "50") int limit) {
      return audit.latest(Math.min(200, Math.max(1, limit)));
    }

    // ---- Utilities
    private void audit(String action, String entityType, String entityId, String actor, Map<String, Object> meta) {
      String who = StringUtils.hasText(actor) ? actor : "system";
      audit.add(new AuditEvent(UUID.randomUUID(), Instant.now(), who, action, entityType, entityId, meta));
    }
  }

  // ----------------------------
  // Error handling
  // ----------------------------
  static final class ApiException extends RuntimeException {
    final HttpStatus status;
    final String code;

    ApiException(HttpStatus status, String code, String message) {
      super(message);
      this.status = status;
      this.code = code;
    }
  }

  @RestControllerAdvice
  static final class ApiExceptionHandler {

    @ExceptionHandler(ApiException.class)
    ResponseEntity<ProblemDetail> handleApi(ApiException ex) {
      ProblemDetail pd = ProblemDetail.forStatus(ex.status);
      pd.setTitle(ex.code);
      pd.setDetail(ex.getMessage());
      pd.setProperty("timestamp", Instant.now().toString());
      return ResponseEntity.status(ex.status).body(pd);
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    ResponseEntity<ProblemDetail> handleValidation(MethodArgumentNotValidException ex) {
      ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
      pd.setTitle("VALIDATION_ERROR");
      pd.setDetail("Request validation failed");
      pd.setProperty("timestamp", Instant.now().toString());
      Map<String, String> errors = new LinkedHashMap<>();
      for (FieldError fe : ex.getBindingResult().getFieldErrors()) {
        errors.put(fe.getField(), fe.getDefaultMessage());
      }
      pd.setProperty("errors", errors);
      return ResponseEntity.badRequest().body(pd);
    }

    @ExceptionHandler(Exception.class)
    ResponseEntity<ProblemDetail> handleOther(Exception ex) {
      ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.INTERNAL_SERVER_ERROR);
      pd.setTitle("INTERNAL_ERROR");
      pd.setDetail("Unexpected error occurred");
      pd.setProperty("timestamp", Instant.now().toString());
      return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(pd);
    }
  }
}
