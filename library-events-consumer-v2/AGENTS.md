# AGENTS.md

## Project Snapshot
- Spring Boot Kafka consumer service: consumes `library-events` and persists to PostgreSQL via JPA.
- Runtime path is currently **Kafka -> Service -> Mapper -> JPA Repositories -> Postgres**.
- `docs/` contains broader course plans (DLT, scheduler, REST controllers), but those parts are **not** present in current `src/main` code.

## Code Map (Start Here)
- Kafka entrypoint: `src/main/java/com/learnkafka/consumer/LibraryEventsConsumer.java`
- Kafka listener config: `src/main/java/com/learnkafka/config/LibraryEventsConsumerConfig.java`
- Business logic and validation: `src/main/java/com/learnkafka/service/LibraryEventService.java`
- DTO -> entity mapping: `src/main/java/com/learnkafka/mapper/LibraryEventMapper.java`
- Persistence model: `src/main/java/com/learnkafka/entity/LibraryEvent.java`, `src/main/java/com/learnkafka/entity/Book.java`
- DB schema source of truth: `src/main/resources/db/migration/V1__init_schema.sql`, `src/main/resources/db/migration/V2__add_audit_columns.sql`

## Architecture and Data Flow Details
- Consumer method `onMessage(...)` receives `ConsumerRecord<Integer, LibraryEventDTO>` from topic `library-events`.
- `LibraryEventService.processEvent(...)` does two validation layers before persistence:
  - Bean validation with `jakarta.validation.Validator` (`@NotNull`, `@NotBlank`, nested `@Valid`).
  - Conditional rules (`book != null`; `UPDATE` requires `libraryEventId`).
- Event branching is explicit:
  - `ADD` -> `libraryEventMapper.toEntity(dto)` -> `libraryEventRepository.save(...)`
  - `UPDATE` -> load existing entity by ID -> `libraryEventMapper.updateEntity(dto, existing)` -> save
- IDs in DTO are `Long` but entities use `Integer`; conversion uses `Math.toIntExact(...)` and can throw on overflow.
- Relationship is bidirectional one-to-one; always set via `LibraryEvent.setBook(...)` so both sides stay synchronized.

## Persistence and Migration Conventions
- Flyway is enabled (`spring.flyway.enabled=true`), JPA DDL is disabled (`ddl-auto: none`) in `application.yml`.
- Do not edit old migrations (`V1`, `V2`); add new `V{N}__*.sql` files for schema changes.
- Audit columns (`created_at`, `updated_at`) are maintained by entity lifecycle hooks (`@PrePersist`, `@PreUpdate`), not service logic.

## Kafka and Serialization Conventions
- Consumer value deserializer is `JsonDeserializer` configured in `src/main/resources/application.yml`.
- Trusted packages currently include both `com.learnkafka.domain` and legacy `com.learnjava.domain`; keep compatibility unless intentionally removing.
- `spring.json.use.type.headers=false` and default type `com.learnkafka.domain.LibraryEventDTO` mean payloads are interpreted as this DTO even without type headers.

## Developer Workflows (Verified)
- Run tests: `./gradlew test --no-daemon` (verified passing locally; integration test uses Embedded Kafka + Testcontainers Postgres).
- Run app: `./gradlew bootRun` (expects reachable Kafka at `spring.kafka.consumer.bootstrap-servers`, default `localhost:9092`).
- Local Postgres helper (for app runtime, not required for tests): `docker compose up -d postgres` using `compose.yaml`.

## Testing Patterns to Follow
- Main integration suite: `src/test/java/com/learnkafka/consumer/LibraryEventsConsumerIntegrationTest.java`.
- Tests produce Kafka messages with `KafkaTemplate<Integer, LibraryEventDTO>` and assert persisted DB state via repositories.
- Cleanup order matters because FK constraints exist: delete `book` records before `library_event` records.
- Async assertions use polling helpers (`waitForCondition`, `assertConditionRemainsTrue`) instead of fixed long sleeps.

## Practical Agent Guardrails
- Prefer extending existing ADD/UPDATE flow in `LibraryEventService` instead of adding logic in listener.
- Keep DTO (`src/main/java/com/learnkafka/domain`) and JPA entities (`src/main/java/com/learnkafka/entity`) separate.
- If adding error-handling features, align with current implementation first; docs in `docs/9_KAFKA_ERROR_HANDLING.md` describe patterns not yet wired in code.
- Preserve Jackson version alignment note in `build.gradle` (explicit `jackson-databind:2.20.2`) to avoid serializer/deserializer mismatches.

