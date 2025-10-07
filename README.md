```java
package com.yourcompany.yourapp.exception;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.validation.BindingResult;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import org.springframework.web.server.ResponseStatusException;

import java.time.Instant;
import java.util.List;
import java.util.stream.Collectors;

/**
 * Global exception handler for REST controllers.
 */
@RestControllerAdvice
public class RestExceptionHandler {

    // --- 404: Resource not found ---
    @ExceptionHandler(NotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(final NotFoundException exception) {
        return buildErrorResponse(exception, HttpStatus.NOT_FOUND);
    }

    // --- 400: Validation errors ---
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(final MethodArgumentNotValidException exception) {
        BindingResult bindingResult = exception.getBindingResult();

        List<FieldError> fieldErrors = bindingResult.getFieldErrors().stream()
                .map(err -> new FieldError(err.getCode(), err.getField()))
                .collect(Collectors.toList());

        ErrorResponse errorResponse = new ErrorResponse();
        errorResponse.setHttpStatus(HttpStatus.BAD_REQUEST.value());
        errorResponse.setException(exception.getClass().getSimpleName());
        errorResponse.setMessage("Validation failed for one or more fields.");
        errorResponse.setFieldErrors(fieldErrors);

        return ResponseEntity.badRequest().body(errorResponse);
    }

    // --- 409: Business rule conflicts (duplicate names, configs, etc.) ---
    @ExceptionHandler({
        DuplicateDashboardNameException.class,
        DashboardWidgetConfigAlreadyExistsException.class
    })
    public ResponseEntity<ErrorResponse> handleConflict(final RuntimeException exception) {
        return buildErrorResponse(exception, HttpStatus.CONFLICT);
    }

    // --- ResponseStatusException from Spring ---
    @ExceptionHandler(ResponseStatusException.class)
    public ResponseEntity<ErrorResponse> handleResponseStatus(final ResponseStatusException exception) {
        return buildErrorResponse(exception, exception.getStatusCode());
    }

    // --- Invalid pagination / custom response exception ---
    @ExceptionHandler(InvalidResponseException.class)
    public ResponseEntity<ErrorPaginationResponse> handleInvalidPagination(final InvalidResponseException ex) {
        ErrorPaginationResponse errorResponse = new ErrorPaginationResponse(
                ex.getErrorCode(),
                ex.getMessage(),
                ex.getErrorDetails(),
                Instant.now()
        );
        return ResponseEntity.badRequest().body(errorResponse);
    }

    // --- 500: Fallback for unexpected errors ---
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleUnexpected(final Exception exception) {
        exception.printStackTrace(); // Ideally use logger.warn/error
        return buildErrorResponse(exception, HttpStatus.INTERNAL_SERVER_ERROR);
    }

    // --- Helper method to build standard error responses ---
    private ResponseEntity<ErrorResponse> buildErrorResponse(Throwable exception, HttpStatus status) {
        ErrorResponse errorResponse = new ErrorResponse();
        errorResponse.setHttpStatus(status.value());
        errorResponse.setException(exception.getClass().getSimpleName());
        errorResponse.setMessage(exception.getMessage());
        errorResponse.setTimestamp(Instant.now());
        return ResponseEntity.status(status).body(errorResponse);
    }
}
```
