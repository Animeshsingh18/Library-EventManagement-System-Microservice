---
name: controller-skill
description: Guidance for building Spring Boot controllers in this repository with async Kafka integrations, sync DB integrations, payload validation, and centralized error handling.
---

# Controller skill for this repository

Use this skill when creating or updating Spring controllers.

## Goal

Build controllers that are consistent with this codebase:

- KafkaTemplate-backed integrations are asynchronous and return `CompletableFuture`
- DB-backed integrations return regular types or `ResponseEntity` wrappers
- error handling is centralized in a class annotated with `@RestControllerAdvice`
- request payload validation mirrors the current `LibraryEventsController` style

## Core rules

1. **Kafka integration must be async**
   - If the request path publishes to Kafka (directly or via service/producer), controller methods should return:
     - `CompletableFuture<ResponseEntity<?>>`, or
     - `CompletableFuture<SomeType>` when response wrapping is not needed
   - Do not block on future completion in controller code (`join`, `get`, etc.).

2. **DB integration can be synchronous**
   - If the request path is DB-only, return regular types (for example, DTO/entity) or `ResponseEntity<...>`.
   - Use `ResponseEntity` where status/header control is needed.

3. **Centralize exception mapping**
   - Do not scatter try/catch blocks across controller endpoints for normal API failures.
   - Use `@RestControllerAdvice` with `@ExceptionHandler` methods.
   - Keep a stable error envelope shape:
     - `{ "errors": ["..."] }`

4. **Validate request payloads at boundary**
   - Use `@Valid` on `@RequestBody` in controller methods.
   - Use Bean Validation annotations in payload models (`@NotNull`, `@NotBlank`, nested `@Valid`).
   - Keep business-rule checks in controller when method-specific (for example, PUT-only id requirements).

## Kafka controller pattern

Use this pattern when endpoint behavior includes Kafka publishing:

```java
@PostMapping
public CompletableFuture<ResponseEntity<?>> create(@RequestBody @Valid SomeRequest request) {
    // Optional method-specific business guard
    if (request.type() != SomeType.CREATE) {
        return CompletableFuture.completedFuture(
                ResponseEntity.badRequest().body(new ErrorResponse(List.of("only CREATE type is supported"))));
    }

    return someService.publishToKafka(request)
            .thenApply(saved -> ResponseEntity.status(HttpStatus.CREATED).body(saved));
}
```

Notes:
- The service/producer chain should stay async when Kafka is involved.
- Controller tests should assert async behavior (`request().asyncStarted()` + `asyncDispatch(...)`).

## DB controller pattern

Use this pattern when endpoint behavior is DB-only:

```java
@GetMapping("/{id}")
public ResponseEntity<SomeDto> findById(@PathVariable Long id) {
    SomeDto body = someDbService.findById(id);
    return ResponseEntity.ok(body);
}
```

Also acceptable when no explicit status control is needed:

```java
@GetMapping("/all")
public List<SomeDto> findAll() {
    return someDbService.findAll();
}
```

## Error handling pattern (`@RestControllerAdvice`)

Keep exception handling centralized:

```java
@RestControllerAdvice
public class ApiControllerAdvice {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        List<String> errors = ex.getBindingResult().getFieldErrors().stream()
                .map(FieldError::getDefaultMessage)
                .filter(msg -> msg != null && !msg.isBlank())
                .sorted()
                .toList();

        return ResponseEntity.badRequest().body(new ErrorResponse(errors));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneric(Exception ex) {
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(new ErrorResponse(List.of("An unexpected error occurred. Please try again later.")));
    }

    public record ErrorResponse(List<String> errors) {}
}
```

## Validation conventions to mirror `LibraryEventsController`

- Keep field-level validation messages human-readable and deterministic for tests.
- Prefer exact message strings when tests assert them.
- For enum/body parse failures, map to a clear bad-request message in advice.
- For method-specific rules (for example, `libraryEventId` required for `PUT`), return explicit `400` with the same error envelope.

## Testing guidance for controllers

1. **Async Kafka endpoints**
   - Use `MockMvc` async assertions:
     - `request().asyncStarted()`
     - `asyncDispatch(...)`

2. **Validation and advice assertions**
   - Assert `$.errors` is an array.
   - Assert exact messages where behavior is contract-sensitive.

3. **Service interaction boundaries**
   - For invalid requests, verify downstream service is not called.

## Apply this skill in this repo

When editing this project:

- follow existing patterns in `src/main/java/com/learnjava/controller/LibraryEventsController.java`
- keep global handler behavior aligned with `src/main/java/com/learnjava/controller/LibraryEventsControllerAdvice.java`
- preserve contract-sensitive error strings and response shapes used by tests

