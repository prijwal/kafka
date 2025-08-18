### End-to-End Kafka Consumer Setup: Spring Boot and Quarkus

#### Spring Boot: Declarative and Factory-Driven

*   **`@KafkaListener` is the "What" and the "Where":** It explicitly tells the Spring container *what* topic(s) to listen to and *which* `groupId` to use.
*   **`ConcurrentKafkaListenerContainerFactory` is the "How":** A factory bean (either custom-defined or auto-configured) that defines *how* the consumer should behave: connection details, deserialization, and advanced error handling strategies like retries and Dead Letter Topics (DLT).

#### Quarkus: Channel-Based and Configuration-Driven

*   **`@Incoming` is the "What" and the "Where":** It links a method to an abstract "channel" defined in the configuration.
*   **`application.properties` is the "How":** This configuration file defines everything: it links the channel to a Kafka topic and configures the consumer's behavior, including the `failure-strategy`, retries, and Dead Letter Queues (DLQ).

---

### 1. Spring Boot: End-to-End Setup

Spring Boot offers two powerful ways to achieve the same outcome: explicit Java configuration for maximum control, or pure property-based configuration for simplicity and speed.

---

### Approach 1: Java `@Configuration` (Maximum Control)

This approach is best when you need to programmatically define beans, share complex configurations, or have logic that cannot be expressed in properties files.

#### `application.yml`

This file provides the basic properties that the Java `@Configuration` class will use.

```yaml
spring:
  kafka:
    consumer:
      bootstrap-servers: localhost:992
      group-id: payments-consumer-group
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      # Use the ErrorHandlingDeserializer to gracefully handle poison pills
      value-deserializer: org.springframework.kafka.support.serializer.ErrorHandlingDeserializer
      properties:
        # Delegate to the standard JsonDeserializer for valid messages
        spring.deserializer.value.delegate.class: org.springframework.kafka.support.serializer.JsonDeserializer
        # Trust packages for deserialization
        spring.json.trusted.packages: "com.example"
    
    # Producer config is needed for the DeadLetterPublishingRecoverer to send messages to the DLT
    producer:
      bootstrap-servers: localhost:9092
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
```

#### Java Configuration (`@Configuration`) with Retries - Fully Implemented

This class explicitly defines every bean, giving you full control over the error handling and retry logic.

```java
import com.example.InvalidPaymentDataException;
import com.example.NonRetryableException;
import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.common.serialization.StringDeserializer;
import org.apache.kafka.common.serialization.StringSerializer;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.env.Environment;
import org.springframework.kafka.config.ConcurrentKafkaListenerContainerFactory;
import org.springframework.kafka.core.*;
import org.springframework.kafka.listener.DeadLetterPublishingRecoverer;
import org.springframework.kafka.listener.DefaultErrorHandler;
import org.springframework.kafka.support.serializer.ErrorHandlingDeserializer;
import org.springframework.kafka.support.serializer.JsonDeserializer;
import org.springframework.kafka.support.serializer.JsonSerializer;
import org.springframework.util.backoff.FixedBackOff;
import java.util.HashMap;
import java.util.Map;

@Configuration
public class KafkaConsumerConfiguration {

    @Autowired
    private Environment environment;

    @Bean
    public ConsumerFactory<String, Object> consumerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, environment.getProperty("spring.kafka.consumer.bootstrap-servers"));
        config.put(ConsumerConfig.GROUP_ID_CONFIG, environment.getProperty("spring.kafka.consumer.group-id"));
        config.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        config.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, ErrorHandlingDeserializer.class);
        config.put(ErrorHandlingDeserializer.VALUE_DESERIALIZER_CLASS, JsonDeserializer.class);
        config.put(JsonDeserializer.TRUSTED_PACKAGES, environment.getProperty("spring.kafka.consumer.properties.spring.json.trusted.packages"));
        return new DefaultKafkaConsumerFactory<>(config);
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Object> kafkaListenerContainerFactory(
            ConsumerFactory<String, Object> consumerFactory,
            KafkaTemplate<String, Object> kafkaTemplate) {

        // 1. Configure the DeadLetterPublishingRecoverer to send failed messages to a DLT.
        DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(kafkaTemplate);

        // 2. Configure a FixedBackOff for retries: 3 attempts with a 5-second delay.
        FixedBackOff backOff = new FixedBackOff(5000L, 3L);

        // 3. Create the error handler with the recoverer and backoff policy.
        DefaultErrorHandler errorHandler = new DefaultErrorHandler(recoverer, backOff);

        // 4. Classify exceptions: tell the handler which errors should NOT be retried.
        // If these exceptions occur, we skip retries and go straight to the DLT.
        errorHandler.addNotRetryableExceptions(NonRetryableException.class, InvalidPaymentDataException.class);

        ConcurrentKafkaListenerContainerFactory<String, Object> factory = new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory);
        factory.setCommonErrorHandler(errorHandler); // Attach the configured error handler.
        return factory;
    }

    @Bean
    public KafkaTemplate<String, Object> kafkaTemplate(ProducerFactory<String, Object> producerFactory) {
        return new KafkaTemplate<>(producerFactory);
    }

    @Bean
    public ProducerFactory<String, Object> producerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, environment.getProperty("spring.kafka.producer.bootstrap-servers"));
        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        return new DefaultKafkaProducerFactory<>(config);
    }
}
```

---

### Approach 2: `application.yml` Only (Auto-Configuration)

This is the idiomatic Spring Boot approach for most use cases. You declare your intent in the properties file, and Spring Boot builds the necessary beans (like the `DefaultErrorHandler`) for you. **No `@Configuration` class is needed for this to work.**

#### `application.yml` (Complete Configuration)

```yaml
spring:
  kafka:
    consumer:
      bootstrap-servers: localhost:9092
      group-id: payments-consumer-group
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.ErrorHandlingDeserializer
      properties:
        spring.deserializer.value.delegate.class: org.springframework.kafka.support.serializer.JsonDeserializer
        spring.json.trusted.packages: "com.example"

    producer:
      bootstrap-servers: localhost:9092
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
    
    listener:
      # Configure the error handler for all @KafkaListener methods
      common-error-handler:
        # Enable the auto-configured DefaultErrorHandler
        enabled: true
      
      # Configure the Dead Letter Topic (DLT) mechanism
      dead-letter-publishing:
        # Enable the auto-configured DeadLetterPublishingRecoverer
        enabled: true
      
      # Configure the retry mechanism (BackOff policy)
      default-error-handler:
        # Set a fixed 5-second delay between retries
        back-off:
          interval: 5000
        # Set the maximum number of retry attempts to 3 (for a total of 4 attempts)
        max-attempts: 4
        
        # A list of exceptions that should NOT be retried and should go straight to the DLT
        not-retryable-exceptions:
          - com.example.NonRetryableException
          - com.example.InvalidPaymentDataException
```

#### Exception Classes, POJO, and Consumer Code (For Both Spring Approaches)

The business logic is completely decoupled from the configuration method.

```java
// Package: com.example
public class RetryableException extends RuntimeException { /* ... */ }
public class NonRetryableException extends RuntimeException { /* ... */ }
public class InvalidPaymentDataException extends RuntimeException { /* ... */ }
public class PaymentEvent { /* ... private fields, getters, setters ... */ }

// Consumer Code
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Component;

@Component
public class PaymentConsumer {

    @KafkaListener(topics = "payments", groupId = "payments-consumer-group")
    public void consume(PaymentEvent event) {
        System.out.println("Consumed: " + event);

        if ("RETRY".equals(event.getType())) {
            System.out.println("Simulating a temporary failure...");
            throw new RetryableException("Service temporarily unavailable.");
        }

        if ("FAIL_FAST".equals(event.getType())) {
            System.out.println("Simulating a permanent data error...");
            throw new NonRetryableException("Invalid event structure.");
        }

        if (event.getAmount() < 0) {
            System.out.println("Simulating invalid business data...");
            throw new InvalidPaymentDataException("Payment amount cannot be negative.");
        }
        
        System.out.println("Payment processed successfully.");
    }
}
```

#### Spring Boot Error Handling Flow (For Both Approaches)

1.  **Exception Interception**: The `DefaultErrorHandler` intercepts any exception from the `@KafkaListener`.
2.  **Exception Classification**: It checks if the exception is in the `not-retryable-exceptions` list.
3.  **Non-Retryable Path**: If matched, the handler skips retries and the `DeadLetterPublishingRecoverer` sends the message to the DLT (`payments.DLT`).
4.  **Retryable Path**: If not matched, the retry policy is activated (3 retries, 5s delay).
5.  **Retry Exhaustion**: After all retries fail, the message is sent to the DLT.

---

### 2. Quarkus: End-to-End Setup

Quarkus uses a single, streamlined properties-based approach.

#### `application.properties`

```properties

kafka.bootstrap.servers=localhost:9092

# 1. Connector and topic configuration // here 'payments-in' is the channel name
mp.messaging.incoming.payments-in.connector=smallrye-kafka
mp.messaging.incoming.payments-in.topic=payments
mp.messaging.incoming.payments-in.group.id=payments-consumer-group

# 2. Deserializers
mp.messaging.incoming.payments-in.key.deserializer=org.apache.kafka.common.serialization.StringDeserializer
mp.messaging.incoming.payments-in.value.deserializer=io.quarkus.kafka.client.serialization.JsonbDeserializer
mp.messaging.incoming.payments-in.value.jsonb.type=com.example.PaymentEvent

# 3. Failure strategy
mp.messaging.incoming.payments-in.failure-strategy=dead-letter-queue

# 4. Retry Configuration
mp.messaging.incoming.payments-in.retries=3
mp.messaging.incoming.payments-in.retry-delay=5000 # Delay in milliseconds

# 5. Non-Retryable Exceptions
mp.messaging.incoming.payments-in.dead-letter-queue.non-retryable-exceptions=com.example.NonRetryableException,com.example.InvalidPaymentDataException
```

#### Exception Classes, POJO, and Quarkus Consumer Code

```java
// Exception classes and PaymentEvent POJO are identical to the Spring examples

// Quarkus Consumer Code
import com.example.InvalidPaymentDataException;
import com.example.NonRetryableException;
import com.example.PaymentEvent;
import com.example.RetryableException;
import jakarta.enterprise.context.ApplicationScoped;
import org.eclipse.microprofile.reactive.messaging.Incoming;

@ApplicationScoped
public class PaymentConsumer {

    @Incoming("payments-in")
    public void consume(PaymentEvent event) {
        System.out.println("Consumed record: " + event);

        if ("RETRY".equals(event.getType())) {
            System.out.println("Simulating a temporary, retryable failure...");
            throw new RetryableException("Downstream service is temporarily unavailable.");
        }

        if ("FAIL_FAST".equals(event.getType())) {
            System.out.println("Simulating a permanent, non-retryable data error...");
            throw new NonRetryableException("The event structure is invalid.");
        }

        if (event.getAmount() < 0) {
            System.out.println("Simulating a permanent, non-retryable business rule violation...");
            throw new InvalidPaymentDataException("Payment amount cannot be negative.");
        }
        
        System.out.println("Payment for transactionId '" + event.getTransactionId() + "' processed successfully.");
    }
}
```

#### Quarkus Error Handling Flow

1.  **Exception Interception**: The SmallRye framework catches exceptions from the `@Incoming` method.
2.  **Exception Classification**: It checks the `dead-letter-queue.non-retryable-exceptions` list.
3.  **Non-Retryable Path**: If the exception is in the list, the message goes directly to the DLQ.
4.  **Retryable Path**: Otherwise, it retries up to `retries` times, waiting `retry-delay` between each attempt.
5.  **Retry Exhaustion**: After the final retry fails, the message is sent to the DLQ.
