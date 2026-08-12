---
name: integration-testing
description: Guidance for writing Spring Boot integration tests in this Kafka producer app using @SpringBootTest, MockMvc, and embedded Kafka.
---

# Integration testing for the library events producer

Use this skill when creating or updating integration tests for this repository.

## Goal

Write true integration tests for the REST-to-Kafka producer flow:

- load the full Spring application context with `@SpringBootTest`
- call HTTP endpoints with `MockMvc`
- use embedded Kafka infrastructure with `@EmbeddedKafka`
- verify the published Kafka record directly from a test consumer

This service is a Kafka **producer** application, so integration tests should validate both sides of the flow:

1. the HTTP response returned by the controller
2. the record published to Kafka

## Required test setup

Make sure the Kafka test support dependency is available:

```groovy
testImplementation 'org.springframework.boot:spring-boot-starter-kafka-test'
```

For controller integration tests in this repo, prefer this annotation stack:

```java
@SpringBootTest(properties = {
		"spring.profiles.active=test",
		"spring.kafka.bootstrap-servers=${spring.embedded.kafka.brokers}",
		"spring.kafka.topic=" + AppConstants.DEFAULT_LIBRARY_EVENTS_TOPIC
})
@AutoConfigureMockMvc
@EmbeddedKafka(partitions = 1, topics = AppConstants.DEFAULT_LIBRARY_EVENTS_TOPIC)
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_CLASS)
class LibraryEventsControllerIntegrationTest {
}
```

### Why these annotations matter

- `@SpringBootTest`
  - required for integration tests here because it loads the **whole Spring context**
  - ensures controller, service, producer, Kafka config, validation, and advice are wired together
- `@AutoConfigureMockMvc`
  - injects `MockMvc` so endpoint calls can be exercised without starting a real server port
- `@EmbeddedKafka`
  - required because this is a Kafka producer app and tests must use embedded Kafka infrastructure instead of an external broker
  - define the topic with `AppConstants.DEFAULT_LIBRARY_EVENTS_TOPIC`
- `@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_CLASS)`
  - prevents Kafka test state from leaking across tests in the class

## Endpoint invocation pattern

All controller methods return `CompletableFuture<ResponseEntity<?>>`, so endpoint assertions must use asynchronous MVC dispatch.

Use `MockMvc` like this:

```java
MvcResult mvcResult = mockMvc.perform(post(AppConstants.API_BASE_PATH)
				.contentType(MediaType.APPLICATION_JSON)
				.content(objectMapper.writeValueAsString(requestEvent)))
		.andExpect(request().asyncStarted())
		.andReturn();

mockMvc.perform(asyncDispatch(mvcResult))
		.andExpect(status().isCreated())
		.andExpect(jsonPath("$.eventType").value("ADD"));
```

For `PUT`, keep the same async pattern and assert `200 OK`.

## Core imports to use

When generating a new integration test in this repo, these are the most important imports to include:

```java
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.webmvc.test.autoconfigure.AutoConfigureMockMvc;
import org.springframework.kafka.test.context.EmbeddedKafka;
import org.springframework.kafka.test.EmbeddedKafkaBroker;
import org.springframework.test.annotation.DirtiesContext;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.MvcResult;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.asyncDispatch;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.put;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.request;
```

Also include the usual JSON/status/assertion imports required by the specific test body.

## Kafka verification pattern

Create a real Kafka consumer against the embedded broker and consume from the topic under test.

Recommended setup:

```java
@Autowired
private EmbeddedKafkaBroker embeddedKafkaBroker;

private Consumer<Long, String> consumer;

@BeforeEach
void setUp() {
	var consumerProps = new HashMap<String, Object>();
	consumerProps.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, embeddedKafkaBroker.getBrokersAsString());
	consumerProps.put(ConsumerConfig.GROUP_ID_CONFIG, "library-events-controller-int-" + UUID.randomUUID());
	consumerProps.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
	consumerProps.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, LongDeserializer.class);
	consumerProps.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);

	consumer = new DefaultKafkaConsumerFactory<Long, String>(consumerProps)
			.createConsumer();

	embeddedKafkaBroker.consumeFromAnEmbeddedTopic(consumer, AppConstants.DEFAULT_LIBRARY_EVENTS_TOPIC);
}

@AfterEach
void tearDown() {
	if (consumer != null) {
		consumer.close();
	}
}
```

Poll until the expected event is found, then deserialize the payload with `ObjectMapper` and assert both key and body.

## Assertions to preserve

Match the project’s API semantics exactly:

- `POST` must publish `ADD`
- `PUT` must publish `UPDATE`
- use `AppConstants.API_BASE_PATH` instead of hardcoding the path
- verify the response payload and the Kafka payload are consistent

Typical assertions:

- `POST`
  - HTTP status is `201 Created`
  - Kafka record key is `null`
  - published event equals the request payload
- `PUT`
  - HTTP status is `200 OK`
  - Kafka record key equals `libraryEventId`
  - published event equals the request payload

## Project conventions

- Prefer `@SpringBootTest` for integration coverage; do **not** replace it with `@WebMvcTest` here.
- Use embedded Kafka; do **not** depend on a running local Kafka cluster for automated tests.
- Use `MockMvc` for endpoint calls.
- Use `asyncDispatch(...)` for controller methods returning `CompletableFuture`.
- Reuse `AppConstants.API_BASE_PATH` and `AppConstants.DEFAULT_LIBRARY_EVENTS_TOPIC`.
- Keep assertions compatible with the controller’s exact request/response behavior.

## Reference implementation in this repo

Base your changes on:

- `src/test/java/com/learnjava/controller/LibraryEventsControllerIntegrationTest.java`

That test class is the canonical example for this repository’s integration testing style.

## Quick template

```java
@SpringBootTest(properties = {
		"spring.profiles.active=test",
		"spring.kafka.bootstrap-servers=${spring.embedded.kafka.brokers}",
		"spring.kafka.topic=" + AppConstants.DEFAULT_LIBRARY_EVENTS_TOPIC
})
@AutoConfigureMockMvc
@EmbeddedKafka(partitions = 1, topics = AppConstants.DEFAULT_LIBRARY_EVENTS_TOPIC)
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_CLASS)
class SomeIntegrationTest {

	@Autowired
	private MockMvc mockMvc;

	@Autowired
	private ObjectMapper objectMapper;

	@Autowired
	private EmbeddedKafkaBroker embeddedKafkaBroker;

	// create consumer in @BeforeEach
	// call endpoint with MockMvc
	// assert async dispatch result
	// poll embedded Kafka and verify the published record
}
```



