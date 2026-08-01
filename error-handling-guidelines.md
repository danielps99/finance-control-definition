# REST API Error Handling Guidelines

This document defines the error handling architecture, standardized response payloads, and exception mapping blueprint for the **Finance Control System MVP** built with Quarkus 3.33 (RESTEasy Reactive).

---

## 1. Architectural Pattern: `@ServerExceptionMapper`

The system uses Quarkus RESTEasy Reactive's native **`@ServerExceptionMapper`** annotation inside a single, centralized `@ApplicationScoped` CDI bean (`br.com.bdws.financecontrol.exception.GlobalExceptionHandler`).

### Core Rules:
* **Centralized Mappers**: All global REST exception handlers reside in `GlobalExceptionHandler.java`. Do not create multiple `ExceptionMapper` provider classes.
* **Flexible Parameters**: Handler methods inject contextual request objects (e.g., `UriInfo`) directly to construct error metadata.
* **Build-Time Optimized**: Leverages Quarkus RESTEasy Reactive native routing for minimal reflection overhead and fast startup.

---

## 2. Standardized Error Payload (`ErrorResponseDTO`)

All API error responses return a uniform JSON body. HTTP status code and reason phrase are excluded as they are conveyed via the HTTP response status line. 

The payload optionally includes `type` (`EnErrorType`) to categorize the error and `params` (`Object`) to pass arbitrary contextual data (e.g., a map, object, or string list of suggested values for the frontend):

```json
{
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "timestamp": "2026-07-24T14:30:00Z",
  "message": "The name 'Ana' already exists.",

  "path": "/api/v1/workspaces",
  "type": "DUPLICATE_ENTITY",
  "params": {
    "name": "Ana",
    "suggestedNames": [
      "Ana 1",
      "José"
    ]
  }
}
```

---

### DTO Implementation (`ErrorResponseDTO.java`)

```java
package br.com.bdws.financecontrol.dto;

import com.fasterxml.jackson.annotation.JsonInclude;
import io.opentelemetry.api.trace.Span;
import java.time.Instant;
import java.util.List;
import java.util.UUID;

@JsonInclude(JsonInclude.Include.NON_NULL)
public record ErrorResponseDTO(
    String traceId,
    Instant timestamp,
    String message,
    String path,
    List<FieldErrorDTO> details,
    EnErrorType type,
    Object params
) {
    public enum EnErrorType {
        RESOURCE_NOT_FOUND,
        INVALID_ARGUMENT,
        VALIDATION_FAILED,
        DUPLICATE_ENTITY,
        BUSINESS_RULE_VIOLATION,
        INTERNAL_SERVER_ERROR
    }

    public record FieldErrorDTO(String field, String message) {}

    private static String resolveTraceId() {
        var spanContext = Span.current().getSpanContext();
        return spanContext.isValid() ? spanContext.getTraceId() : UUID.randomUUID().toString();
    }

    public static ErrorResponseDTO of(String message, String path) {
        return new ErrorResponseDTO(resolveTraceId(), Instant.now(), message, path, null, null, null);
    }

    public static ErrorResponseDTO of(EnErrorType type, String message, String path) {
        return new ErrorResponseDTO(resolveTraceId(), Instant.now(), message, path, null, type, null);
    }

    public static ErrorResponseDTO of(EnErrorType type, String message, String path, Object params) {
        return new ErrorResponseDTO(resolveTraceId(), Instant.now(), message, path, null, type, params);
    }

    public static ErrorResponseDTO of(EnErrorType type, String message, String path, List<FieldErrorDTO> details) {
        return new ErrorResponseDTO(resolveTraceId(), Instant.now(), message, path, details, type, null);
    }

    public static ErrorResponseDTO of(String traceId, String message, String path, List<FieldErrorDTO> details, EnErrorType type, Object params) {
        return new ErrorResponseDTO(traceId != null ? traceId : resolveTraceId(), Instant.now(), message, path, details, type, params);
    }
}

```

---

## 3. Implementation Blueprint (`GlobalExceptionHandler.java`)

```java
package br.com.bdws.financecontrol.exception;

import br.com.bdws.financecontrol.dto.ErrorResponseDTO;
import br.com.bdws.financecontrol.dto.ErrorResponseDTO.EnErrorType;
import br.com.bdws.financecontrol.dto.ErrorResponseDTO.FieldErrorDTO;
import io.quarkus.resteasy.reactive.server.ServerExceptionMapper;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.persistence.EntityNotFoundException;
import jakarta.validation.ConstraintViolationException;
import jakarta.ws.rs.core.Response;
import jakarta.ws.rs.core.UriInfo;
import org.jboss.logging.Logger;

import java.util.List;

@ApplicationScoped
public class GlobalExceptionHandler {

    private static final Logger LOG = Logger.getLogger(GlobalExceptionHandler.class);

    // 1. Resource Not Found (404)
    @ServerExceptionMapper
    public Response handleNotFound(EntityNotFoundException ex, UriInfo uriInfo) {
        LOG.warnf("Resource not found at %s: %s", uriInfo.getPath(), ex.getMessage());
        ErrorResponseDTO payload = ErrorResponseDTO.of(
            EnErrorType.RESOURCE_NOT_FOUND,
            ex.getMessage(),
            uriInfo.getPath()
        );
        return Response.status(Response.Status.NOT_FOUND).entity(payload).build();
    }

    // 2. Business Rule / Invalid Arguments (400)
    @ServerExceptionMapper
    public Response handleIllegalArgument(IllegalArgumentException ex, UriInfo uriInfo) {
        LOG.warnf("Invalid argument at %s: %s", uriInfo.getPath(), ex.getMessage());
        ErrorResponseDTO payload = ErrorResponseDTO.of(
            EnErrorType.INVALID_ARGUMENT,
            ex.getMessage(),
            uriInfo.getPath()
        );
        return Response.status(Response.Status.BAD_REQUEST).entity(payload).build();
    }

    // 3. Bean Validation Failures (400 Bad Request with field errors)
    @ServerExceptionMapper
    public Response handleConstraintViolation(ConstraintViolationException ex, UriInfo uriInfo) {
        List<FieldErrorDTO> fieldErrors = ex.getConstraintViolations().stream()
            .map(v -> new FieldErrorDTO(v.getPropertyPath().toString(), v.getMessage()))
            .toList();

        ErrorResponseDTO payload = ErrorResponseDTO.of(
            EnErrorType.VALIDATION_FAILED,
            "Validation failed for one or more fields",
            uriInfo.getPath(),
            fieldErrors
        );
        return Response.status(Response.Status.BAD_REQUEST).entity(payload).build();
    }

    // 4. Catch-all Internal Server Errors (500)
    @ServerExceptionMapper
    public Response handleUncaughtThrowable(Throwable ex, UriInfo uriInfo) {
        ErrorResponseDTO payload = ErrorResponseDTO.of(
            EnErrorType.INTERNAL_SERVER_ERROR,
            "An unexpected error occurred. Please contact system support.",
            uriInfo.getPath()
        );
        LOG.errorf(ex, "[TraceID: %s] Unhandled exception occurred at %s", payload.traceId(), uriInfo.getPath());
        return Response.status(Response.Status.INTERNAL_SERVER_ERROR).entity(payload).build();
    }
}
```

---

## 4. Summary Rules for AI & Developers

1. **Use `@ServerExceptionMapper`**: Consolidate exception handling inside `GlobalExceptionHandler`. Do not create separate JAX-RS `ExceptionMapper<T>` provider files.
2. **Uniform Payload**: Always return `ErrorResponseDTO` for consistent API client integration.
3. **Log Boundaries**: Use `LOG.warnf` for client errors (4xx) and `LOG.errorf` with full tracebacks for server errors (5xx).
4. **Sanitize 5xx Messages**: Never expose internal database stack traces or SQL exceptions in the 500 error response message.
5. **Traceability & Classification**: `ErrorResponseDTO` includes `traceId` (resolved via OpenTelemetry `Span.current().getSpanContext().getTraceId()` following Grafana's 32-character W3C trace ID pattern, with random fallback) for seamless Grafana (Loki & Tempo) log and trace correlation, `type` (`EnErrorType`) for error categorization, and `params` (`Object`) to pass contextual objects to the frontend.




