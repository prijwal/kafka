**`@KafkaListener` is the "What" and the "Where":**
*   It marks a method as a target for receiving Kafka messages.
*   It tells Spring *what* topic to listen to (`topics = "payments"`) and *what* consumer group to use (`groupId = "..."`).
*   It's the **job assignment**.

**`ConcurrentKafkaListenerContainerFactory` is the "How":**
*   It's a factory that builds and configures the underlying message listener container (the "worker").
*   It defines *how* the consumer should behave: how to connect (bootstrap servers), how to deserialize messages, how to handle errors, how many threads to use, etc.
*   It's the **worker's blueprint and toolset**.

You **always** need both. When you use `@KafkaListener`, Spring looks for a `ConcurrentKafkaListenerContainerFactory` bean to create the listener for that specific job.
*   **If you configure via `application.properties`**, Spring Boot's auto-configuration creates a default factory bean for you behind the scenes.
*   **If you configure via Java `@Configuration`**, you are creating a *custom* factory bean, overriding the default one. `@KafkaListener` will then use your custom factory instead.

---

## 1. Spring Boot: Configuration via `application.properties`

This is the most common approach, relying on Spring Boot's auto-configuration. Spring creates the necessary factory beans based on your properties.

### `application.yml`

```yaml
spring:
  kafka:
    # ===================================================================================
    # == BASIC CONSUMER CONFIGURATION (Commented Out) ===================================
    # ===================================================================================
    # This basic setup is simple but lacks robust error handling. A single malformed
    # message (a "poison pill") could cause a deserialization error and block the
    # consumer partition indefinitely. It is superseded by the advanced DLT
    # configuration below.
    # ===================================================================================
    # consumer:
    #   bootstrap-servers: localhost:9092
    #   group-id: payments-consumer-group
    #   key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
    #   value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer # Prone to poison pills
    #   properties:
    #     spring.json.trusted.packages: "*"


    # ===================================================================================
    # == ADVANCED CONSUMER CONFIGURATION (With Dead Letter Topic) =======================
    # ===================================================================================
    # This configuration is production-ready. It uses the ErrorHandlingDeserializer
    # and a DefaultErrorHandler to automatically route failed messages to a DLT.
    # ===================================================================================
    consumer:
      bootstrap-servers: localhost:9092
      group-id: payments-consumer-group
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      # 1. Use ErrorHandlingDeserializer to wrap the actual deserializer.
      # This prevents deserialization errors from stopping the consumer.
      value-deserializer: org.springframework.kafka.support.serializer.ErrorHandlingDeserializer
      properties:
        # 2. Tell the ErrorHandlingDeserializer which class to delegate to.
        spring.deserializer.value.delegate.class: org.springframework.kafka.support.serializer.JsonDeserializer
        spring.json.trusted.packages: "*"
    
    # 3. Configure a producer for the KafkaTemplate, which the error handler will use
    # to send messages to the Dead Letter Topic.
    producer:
      bootstrap-servers: localhost:9092
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer

```

### Consumer Code

The `@KafkaListener` annotation tells Spring's auto-configured factory to create a listener for this method.

```java
@Component
public class PaymentConsumer {

    @KafkaListener(topics = "payments", groupId = "payments-consumer-group")
    public void consume(PaymentEvent event) {
        System.out.println("Consumed: " + event);
        // If this method throws an exception, Spring's default error handler
        // will catch it and send the message to the DLT.
        if (event.getAmount() < 0) {
            throw new IllegalArgumentException("Payment amount cannot be negative.");
        }
    }
}
```

---

## 2. Spring Boot: Configuration via Java `@Configuration`

This approach gives you maximum control by defining the factory beans yourself. It is used when the `application.properties` approach is not flexible enough.

### `application.yml`

You still need basic properties for the Java config to read.

```yaml
spring:
  kafka:
    consumer:
      bootstrap-servers: localhost:9092
      group-id: payments-consumer-group
      properties:
        spring.json.trusted.packages: "*"
```

### Java Configuration (`@Configuration`)

Here, you explicitly define all the beans that Spring Boot would normally auto-configure.

```java
@Configuration
public class KafkaConsumerConfiguration {

    @Autowired
    private Environment environment;

    /*
    // ===================================================================================
    // == BASIC CONSUMER CONFIGURATION (Commented Out) ===================================
    // ===================================================================================
    // This basic configuration is commented out because it lacks a proper error
    // handling strategy. The advanced configuration below includes an ErrorHandlingDeserializer
    // and a DefaultErrorHandler to manage poison pills and processing failures by
    // sending them to a Dead Letter Topic (DLT).
    // ===================================================================================

    @Bean
    public ConsumerFactory<String, Object> basicConsumerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, environment.getProperty("spring.kafka.consumer.bootstrap-servers"));
        config.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        config.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JsonDeserializer.class);
        config.put(JsonDeserializer.TRUSTED_PACKAGES, environment.getProperty("spring.kafka.consumer.properties.spring.json.trusted.packages"));
        config.put(ConsumerConfig.GROUP_ID_CONFIG, environment.getProperty("spring.kafka.consumer.group-id"));
        return new DefaultKafkaConsumerFactory<>(config);
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Object> basicKafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, Object> factory = new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(basicConsumerFactory());
        return factory;
    }
    */

    // ===================================================================================
    // == ADVANCED CONSUMER CONFIGURATION (With Dead Letter Topic) =======================
    // ===================================================================================
    // This is the recommended production-ready configuration.
    // ===================================================================================

    @Bean
    public ConsumerFactory<String, Object> consumerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, environment.getProperty("spring.kafka.consumer.bootstrap-servers"));
        config.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        config.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, ErrorHandlingDeserializer.class);
        config.put(ErrorHandlingDeserializer.VALUE_DESERIALIZER_CLASS, JsonDeserializer.class);
        config.put(JsonDeserializer.TRUSTED_PACKAGES, environment.getProperty("spring.kafka.consumer.properties.spring.json.trusted.packages"));
        config.put(ConsumerConfig.GROUP_ID_CONFIG, environment.getProperty("spring.kafka.consumer.group-id"));
        return new DefaultKafkaConsumerFactory<>(config);
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Object> kafkaListenerContainerFactory(
            ConsumerFactory<String, Object> consumerFactory,
            KafkaTemplate<String, Object> kafkaTemplate) {
        DefaultErrorHandler errorHandler = new DefaultErrorHandler(new DeadLetterPublishingRecoverer(kafkaTemplate));
        ConcurrentKafkaListenerContainerFactory<String, Object> factory = new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory);
        factory.setCommonErrorHandler(errorHandler);
        return factory;
    }

    @Bean
    public KafkaTemplate<String, Object> kafkaTemplate(ProducerFactory<String, Object> producerFactory) {
        return new KafkaTemplate<>(producerFactory);
    }

    @Bean
    public ProducerFactory<String, Object> producerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, environment.getProperty("spring.kafka.consumer.bootstrap-servers"));
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        return new DefaultKafkaProducerFactory<>(config);
    }
}
```

### Consumer Code

The consumer code is **identical**. The `@KafkaListener` annotation now finds and uses the custom `kafkaListenerContainerFactory` bean you defined above instead of the auto-configured one.

```java
@Component
public class PaymentConsumer {

    @KafkaListener(topics = "payments", groupId = "payments-consumer-group")
    public void consume(PaymentEvent event) {
        System.out.println("Consumed: " + event);
        if (event.getAmount() < 0) {
            throw new IllegalArgumentException("Payment amount cannot be negative.");
        }
    }
}
```

---

## 3. Quarkus: Configuration via `application.properties`

This is the standard, idiomatic way to configure consumers in Quarkus using SmallRye Reactive Messaging.

### `application.properties`

```properties
kafka.bootstrap.servers=localhost:9092

# ===================================================================================
# == BASIC CONSUMER CONFIGURATION (Commented Out) ===================================
# ===================================================================================
# This basic setup is commented out because its default failure-strategy is 'fail',
# which stops the entire application on an error. This is not suitable for production.
# The advanced configuration below uses 'dead-letter-queue' for resilience.
# ===================================================================================
# mp.messaging.incoming.payments-in.connector=smallrye-kafka
# mp.messaging.incoming.payments-in.topic=payments
# mp.messaging.incoming.payments-in.group.id=payments-consumer-group
# mp.messaging.incoming.payments-in.value.deserializer=org.apache.kafka.common.serialization.StringDeserializer


# ===================================================================================
# == ADVANCED CONSUMER CONFIGURATION (With Dead Letter Queue) =======================
# ===================================================================================
# This is the recommended production-ready configuration for Quarkus.
# ===================================================================================
mp.messaging.incoming.payments-in.connector=smallrye-kafka
mp.messaging.incoming.payments-in.topic=payments
mp.messaging.incoming.payments-in.group.id=payments-consumer-group
mp.messaging.incoming.payments-in.value.deserializer=org.apache.kafka.common.serialization.StringDeserializer

# 1. Set the failure strategy to 'dead-letter-queue'.
# If the consumer method throws an exception, the message is automatically sent to a DLQ topic.
mp.messaging.incoming.payments-in.failure-strategy=dead-letter-queue
```

### Consumer Code

The `@Incoming` annotation links this method to the `payments-in` channel defined in the properties.

```java
import org.eclipse.microprofile.reactive.messaging.Incoming;
import jakarta.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class PaymentConsumer {

    @Incoming("payments-in")
    public void consume(String message) {
        System.out.println("Consumed: " + message);
        if (message.contains("fail")) {
            throw new RuntimeException("This is a poison pill message!");
        }
    }
}
```

---

## 4. Quarkus: Programmatic Configuration (Java)

Quarkus does **not** have a direct equivalent to Spring's `@Configuration` beans for defining listener containers. The framework is designed to be configured via `application.properties`. The "programmatic" approach in Quarkus involves building a reactive stream pipeline, which is a very different and more advanced pattern, typically used for stream processing rather than simple message consumption.

This method is **not recommended** for standard consumer use cases. The `application.properties` approach is the correct and idiomatic way. This is included for completeness.

### Explanation

You would not typically define bootstrap servers or group IDs in Java. Instead, you would build a reactive pipeline that connects to a channel already configured in `application.properties`. Error handling is done within the stream itself using Mutiny operators like `.onFailure()`.

### Programmatic Consumer Code

```java
import io.smallrye.reactive.messaging.kafka.KafkaRecord;
import io.smallrye.mutiny.Multi;
import jakarta.enterprise.context.ApplicationScoped;
import org.eclipse.microprofile.reactive.messaging.Incoming;
import org.eclipse.microprofile.reactive.messaging.Outgoing;

@ApplicationScoped
public class AdvancedPaymentProcessor {

    // This is not a typical consumer. It's a stream processor.
    // It consumes from one channel and must produce to another.
    @Incoming("payments-in-stream") // Consumes from a configured channel
    @Outgoing("processed-payments-out") // Must produce to an output channel
    public Multi<KafkaRecord<String, String>> process(Multi<KafkaRecord<String, String>> records) {
        return records
            .onItem().invoke(record -> {
                System.out.println("Processing record: " + record.getPayload());
                if (record.getPayload().contains("fail")) {
                    // Error handling is done inside the stream
                    throw new RuntimeException("Failed to process record!");
                }
            })
            // In a real reactive pipeline, you would handle errors here
            .onFailure().invoke(failure -> {
                System.err.println("Stream processing failed: " + failure.getMessage());
                // You could use .recoverWithItem() or other operators here.
                // Sending to a DLQ would require injecting a Kafka producer and manually sending.
            });
    }
}
```
