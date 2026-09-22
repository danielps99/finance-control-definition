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

The `message` property carries the error translation key (`EnErrorMessage`), which the frontend uses to translate and render localized error messages. The payload optionally includes `params` (`Object`) to pass arbitrary contextual data for dynamic placeholder interpolation (e.g., entity name, suggested values, constraints) and `details` for field-level validation failures:

```json
{
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "timestamp": "2026-07-24T14:30:00Z",
  "message": "DUPLICATE_ENTITY",
  "path": "/api/v1/workspaces",
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
    Object params
) {
    public record FieldErrorDTO(String field, String message) {}

    private static String resolveTraceId() {
        var spanContext = Span.current().getSpanContext();
        return spanContext.isValid() ? spanContext.getTraceId() : UUID.randomUUID().toString();
    }

    public static ErrorResponseDTO of(String message, String path) {
        return new ErrorResponseDTO(resolveTraceId(), Instant.now(), message, path, null, null);
    }

    public static ErrorResponseDTO of(String message, String path, Object params) {
        return new ErrorResponseDTO(resolveTraceId(), Instant.now(), message, path, null, params);
    }

    public static ErrorResponseDTO of(String message, String path, List<FieldErrorDTO> details) {
        return new ErrorResponseDTO(resolveTraceId(), Instant.now(), message, path, details, null);
    }

    public static ErrorResponseDTO of(String traceId, String message, String path, List<FieldErrorDTO> details, Object params) {
        return new ErrorResponseDTO(traceId != null ? traceId : resolveTraceId(), Instant.now(), message, path, details, params);
    }
}
```

---

## 3. Error Message Enum (`EnErrorMessage.java`) & Custom Exception Hierarchy

The system defines specific semantic exception types that map directly to standard HTTP status codes without complex switch logic. All business exceptions inherit from a common `BaseException` and accept an `EnErrorMessage` enum value representing the localization key, along with optional contextual `params`.

### `EnErrorMessage.java`

```java
package br.com.bdws.financecontrol.enums;

public enum EnErrorMessage {
    RESOURCE_NOT_FOUND,
    INVALID_ARGUMENT,
    VALIDATION_FAILED,
    DUPLICATE_ENTITY,
    BUSINESS_RULE_VIOLATION,
    INTERNAL_SERVER_ERROR
}
```

### Exception Hierarchy Implementation

```java
package br.com.bdws.financecontrol.exception;

import br.com.bdws.financecontrol.enums.EnErrorMessage;

// Base Exception containing contextual parameters and error message key
public abstract class BaseException extends RuntimeException {
    private final Object params;

    public BaseException(EnErrorMessage errorMessage) {
        super(errorMessage.name());
        this.params = null;
    }

    public BaseException(EnErrorMessage errorMessage, Object params) {
        super(errorMessage.name());
        this.params = params;
    }

    public Object getParams() { return params; }
}

// 404 Not Found
public class NotFoundException extends BaseException {
    public NotFoundException(EnErrorMessage errorMessage) { super(errorMessage); }
    public NotFoundException(EnErrorMessage errorMessage, Object params) { super(errorMessage, params); }
}

// 409 Conflict
public class ConflictException extends BaseException {
    public ConflictException(EnErrorMessage errorMessage) { super(errorMessage); }
    public ConflictException(EnErrorMessage errorMessage, Object params) { super(errorMessage, params); }
}

// 422 Unprocessable Entity (Business Rule Violation)
public class BusinessException extends BaseException {
    public BusinessException(EnErrorMessage errorMessage) { super(errorMessage); }
    public BusinessException(EnErrorMessage errorMessage, Object params) { super(errorMessage, params); }
}

// 400 Bad Request
public class BadRequestException extends BaseException {
    public BadRequestException(EnErrorMessage errorMessage) { super(errorMessage); }
    public BadRequestException(EnErrorMessage errorMessage, Object params) { super(errorMessage, params); }
}

// 500 Internal Server Error
public class InternalServerErrorException extends BaseException {
    public InternalServerErrorException(EnErrorMessage errorMessage) { super(errorMessage); }
    public InternalServerErrorException(EnErrorMessage errorMessage, Object params) { super(errorMessage, params); }
}
```

### Usage Examples in Service Layer

```java
// 409 Conflict: Duplicate Entity with contextual parameters
throw new ConflictException(EnErrorMessage.DUPLICATE_ENTITY, Map.of("name", name));

// 422 Unprocessable Entity: Business Rule Violation
throw new BusinessException(EnErrorMessage.BUSINESS_RULE_VIOLATION);

// 404 Not Found: Resource Not Found
throw new NotFoundException(EnErrorMessage.RESOURCE_NOT_FOUND);

// 400 Bad Request: Invalid Argument
throw new BadRequestException(EnErrorMessage.INVALID_ARGUMENT);
```

---

## 4. Implementation Blueprint (`GlobalExceptionHandler.java`)

```java
package br.com.bdws.financecontrol.exception;

import br.com.bdws.financecontrol.dto.ErrorResponseDTO;
import br.com.bdws.financecontrol.dto.ErrorResponseDTO.FieldErrorDTO;
import br.com.bdws.financecontrol.enums.EnErrorMessage;
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
    public Response handleNotFound(NotFoundException ex, UriInfo uriInfo) {
        LOG.warnf("Resource not found at %s: [%s]", uriInfo.getPath(), ex.getMessage());
        ErrorResponseDTO payload = ErrorResponseDTO.of(ex.getMessage(), uriInfo.getPath(), ex.getParams());
        return Response.status(Response.Status.NOT_FOUND).entity(payload).build();
    }

    // 2. Conflict / Duplicate Entity (409)
    @ServerExceptionMapper
    public Response handleConflict(ConflictException ex, UriInfo uriInfo) {
        LOG.warnf("Conflict at %s: [%s]", uriInfo.getPath(), ex.getMessage());
        ErrorResponseDTO payload = ErrorResponseDTO.of(ex.getMessage(), uriInfo.getPath(), ex.getParams());
        return Response.status(Response.Status.CONFLICT).entity(payload).build();
    }

    // 3. Business Rule Violation (422 Unprocessable Entity)
    @ServerExceptionMapper
    public Response handleBusinessException(BusinessException ex, UriInfo uriInfo) {
        LOG.warnf("Business rule violation at %s: [%s]", uriInfo.getPath(), ex.getMessage());
        ErrorResponseDTO payload = ErrorResponseDTO.of(ex.getMessage(), uriInfo.getPath(), ex.getParams());
        return Response.status(Response.Status.UNPROCESSABLE_ENTITY).entity(payload).build();
    }

    // 4. Bad Request / Invalid Arguments (400)
    @ServerExceptionMapper
    public Response handleBadRequest(BadRequestException ex, UriInfo uriInfo) {
        LOG.warnf("Bad request at %s: [%s]", uriInfo.getPath(), ex.getMessage());
        ErrorResponseDTO payload = ErrorResponseDTO.of(ex.getMessage(), uriInfo.getPath(), ex.getParams());
        return Response.status(Response.Status.BAD_REQUEST).entity(payload).build();
    }

    // 5. Bean Validation Failures (400 Bad Request with field errors)
    @ServerExceptionMapper
    public Response handleConstraintViolation(ConstraintViolationException ex, UriInfo uriInfo) {
        List<FieldErrorDTO> fieldErrors = ex.getConstraintViolations().stream()
            .map(v -> new FieldErrorDTO(v.getPropertyPath().toString(), v.getMessage()))
            .toList();

        ErrorResponseDTO payload = ErrorResponseDTO.of(
            EnErrorMessage.VALIDATION_FAILED.name(),
            uriInfo.getPath(),
            fieldErrors
        );
        return Response.status(Response.Status.BAD_REQUEST).entity(payload).build();
    }

    // 6. JPA Entity Not Found Fallback (404)
    @ServerExceptionMapper
    public Response handleEntityNotFound(EntityNotFoundException ex, UriInfo uriInfo) {
        LOG.warnf("Resource not found at %s: %s", uriInfo.getPath(), ex.getMessage());
        ErrorResponseDTO payload = ErrorResponseDTO.of(
            EnErrorMessage.RESOURCE_NOT_FOUND.name(),
            uriInfo.getPath()
        );
        return Response.status(Response.Status.NOT_FOUND).entity(payload).build();
    }

    // 7. Explicit Internal Server Error (500)
    @ServerExceptionMapper
    public Response handleInternalServerError(InternalServerErrorException ex, UriInfo uriInfo) {
        ErrorResponseDTO payload = ErrorResponseDTO.of(
            ex.getMessage(),
            uriInfo.getPath(),
            ex.getParams()
        );
        LOG.errorf(ex, "[TraceID: %s] Internal server error at %s", payload.traceId(), uriInfo.getPath());
        return Response.status(Response.Status.INTERNAL_SERVER_ERROR).entity(payload).build();
    }

    // 8. Catch-all Internal Server Errors (500)
    @ServerExceptionMapper
    public Response handleUncaughtThrowable(Throwable ex, UriInfo uriInfo) {
        ErrorResponseDTO payload = ErrorResponseDTO.of(
            EnErrorMessage.INTERNAL_SERVER_ERROR.name(),
            uriInfo.getPath()
        );
        LOG.errorf(ex, "[TraceID: %s] Unhandled exception occurred at %s", payload.traceId(), uriInfo.getPath());
        return Response.status(Response.Status.INTERNAL_SERVER_ERROR).entity(payload).build();
    }
}
```

---

## 5. Summary Rules for AI & Developers

1. **Use `@ServerExceptionMapper`**: Consolidate exception handling inside `GlobalExceptionHandler`. Do not create separate JAX-RS `ExceptionMapper<T>` provider files.
2. **Explicit Semantic Exceptions**: Throw dedicated exceptions (`NotFoundException`, `ConflictException`, `BusinessException`, `BadRequestException`, `InternalServerErrorException`) matching HTTP status codes instead of generic runtime exceptions.
3. **Uniform Payload**: Always return `ErrorResponseDTO` for consistent API client integration.
4. **Log Boundaries**: Use `LOG.warnf` for client errors (4xx) and `LOG.errorf` with full tracebacks for server errors (5xx).
5. **Sanitize 5xx Messages**: Never expose internal database stack traces or SQL exceptions in the 500 error response message.
6. **Frontend Localization & Traceability**: The `message` field contains the `EnErrorMessage` enum key used by the frontend for localization/i18n translation. `params` provides interpolation values, and `traceId` (resolved via OpenTelemetry W3C trace ID) enables log-to-trace correlation in Grafana (Loki & Tempo).






