
### End-to-End Kafka Consumer Setup: Spring Boot and Quarkus

#### Spring Boot: Declarative and Factory-Driven

In Spring Boot, the setup is split between "what" to do and "how" to do it.

*   **`@KafkaListener` is the "What" and the "Where":**
    *   It explicitly tells the Spring container *what* topic(s) to listen to and *which* `groupId` to use.

*   **`ConcurrentKafkaListenerContainerFactory` is the "How":**
    *   This is a factory bean responsible for creating and configuring the underlying machinery (the "worker" or message listener container) that actually connects to Kafka, fetches records, and passes them to your `@KafkaListener` method.
    *   It defines *how* the consumer should behave: connection details (bootstrap servers), serialization/deserialization logic, error handling strategies (like using a `DefaultErrorHandler`), threading models (concurrency), and more.

#### Quarkus: Channel-Based and Configuration-Driven

Quarkus, using the SmallRye Reactive Messaging framework, abstracts away the listener and factory concepts into a more streamlined, configuration-centric model based on "channels".

*   **`@Incoming` is the "What" and the "Where":**
    *   This annotation links a method to a named "channel" (e.g., `@Incoming("payments-in")`). It doesn't know or care about Kafka topics or group IDs directly.
    *   It's a declarative statement: "This method consumes messages from a channel named 'payments-in'."

*   **`application.properties` is the "How":**
    *   This configuration file is where all the Kafka-specific details are defined. It acts as the "factory" and configuration layer combined.
    *   You create the link between the abstract channel and the physical Kafka topic here (`mp.messaging.incoming.payments-in.topic=payments`).
    *   It also defines *how* the consumer for that channel should behave: connection details (`kafka.bootstrap.servers`), the consumer group (`group.id`), deserialization, and crucial error handling logic like the `failure-strategy`.

---

### Configurations

### 1. Spring Boot: End-to-End Setup

#### `application.yml`

```yaml
spring:
  kafka:
    consumer:
      bootstrap-servers: localhost:9092
      group-id: payments-consumer-group
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer

      # ❌ Naive approach (commented out):
      # Using JsonDeserializer directly can cause poison pill issues —
      # a single malformed message will block the consumer forever.
      # value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer

      # ✅ Correct approach:
      # Use ErrorHandlingDeserializer to wrap the actual deserializer.
      # This ensures deserialization errors (like "poison pill" messages)
      # won’t break the consumer; instead, they get routed to a Dead Letter Topic (DLT).
      value-deserializer: org.springframework.kafka.support.serializer.ErrorHandlingDeserializer

      properties:
        # ✅ Correct approach:
        # Tell ErrorHandlingDeserializer which actual deserializer to delegate to.
        spring.deserializer.value.delegate.class: org.springframework.kafka.support.serializer.JsonDeserializer
        # Still allow all packages for JSON deserialization.
        spring.json.trusted.packages: "*"
    
    producer:
      # This producer is required by the error handler to send messages to the DLT.
      bootstrap-servers: localhost:9092
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
```

#### Java Configuration (`@Configuration`)

```java
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
        config.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);

        // ❌ Naive approach (commented out):
        // Directly using JsonDeserializer can cause poison pill issues —
        // a malformed message will block the consumer forever.
        // config.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JsonDeserializer.class);

        // ✅ Correct approach:
        // Use ErrorHandlingDeserializer to wrap the actual JsonDeserializer.
        // This ensures poison pill messages go to Dead Letter Topic (DLT),
        // and the consumer continues processing other records.
        config.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, ErrorHandlingDeserializer.class);
        config.put(ErrorHandlingDeserializer.VALUE_DESERIALIZER_CLASS, JsonDeserializer.class);

        // ❌ Old property (commented out):
        // Trust all packages for deserialization (required by JsonDeserializer).
        // config.put(JsonDeserializer.TRUSTED_PACKAGES, "*");

        // ✅ Correct approach:
        // Still set trusted packages, but pulled from application properties.
        config.put(JsonDeserializer.TRUSTED_PACKAGES, environment.getProperty("spring.kafka.consumer.properties.spring.json.trusted.packages"));

        config.put(ConsumerConfig.GROUP_ID_CONFIG, environment.getProperty("spring.kafka.consumer.group-id"));
        return new DefaultKafkaConsumerFactory<>(config);
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Object> kafkaListenerContainerFactory(
            ConsumerFactory<String, Object> consumerFactory,
            KafkaTemplate<String, Object> kafkaTemplate) {

        // Configure DeadLetterPublishingRecoverer so poison pills go to a DLT.
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
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, environment.getProperty("spring.kafka.producer.bootstrap-servers"));

        // ❌ Naive approach (commented out):
        // A generic serializer could be used, but JsonSerializer is needed
        // for structured message formats.
        // config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);

        // ✅ Correct approach:
        // Use JsonSerializer for values and StringSerializer for keys.
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);

        return new DefaultKafkaProducerFactory<>(config);
    }
}
```

#### Consumer Code

```java
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Component;

@Component
public class PaymentConsumer {

    // This listener will be built by the custom "kafkaListenerContainerFactory" bean.
    @KafkaListener(topics = "payments", groupId = "payments-consumer-group")
    public void consume(PaymentEvent event) {
        System.out.println("Consumed: " + event);

        // If this method throws an exception, the DefaultErrorHandler will catch it
        // and send the original message to a topic named "payments.DLT".
        if (event.getAmount() < 0) {
            throw new IllegalArgumentException("Payment amount cannot be negative.");
        }
    }
}
```

---

### 2. Quarkus: Setup

#### `application.properties`

```properties
# ===================================================================
# Global Kafka broker configuration
# ===================================================================
kafka.bootstrap.servers=localhost:9092


# ===================================================================
# Consumer configuration for 'payments-in' channel
# ===================================================================

# ❌ Naive approach (commented out):
# This setup uses the default failure strategy 'fail'. 
# That means if a single message causes an exception, 
# the whole consumer/app will stop — not safe for production!
# mp.messaging.incoming.payments-in.connector=smallrye-kafka
# mp.messaging.incoming.payments-in.topic=payments
# mp.messaging.incoming.payments-in.group.id=payments-consumer-group
# mp.messaging.incoming.payments-in.value.deserializer=io.quarkus.kafka.client.serialization.JsonbDeserializer
# mp.messaging.incoming.payments-in.value.jsonb.type=com.example.PaymentEvent
# # failure-strategy defaults to 'fail' if not specified!


# ✅ Correct approach:
# Use Dead Letter Queue (DLQ) failure strategy to handle poison pills
# and other message processing errors gracefully.

# 1. Connector and topic configuration
mp.messaging.incoming.payments-in.connector=smallrye-kafka
mp.messaging.incoming.payments-in.topic=payments
mp.messaging.incoming.payments-in.group.id=payments-consumer-group

# 2. Deserializers
mp.messaging.incoming.payments-in.key.deserializer=org.apache.kafka.common.serialization.StringDeserializer
mp.messaging.incoming.payments-in.value.deserializer=io.quarkus.kafka.client.serialization.JsonbDeserializer
# Specify the target class for JsonbDeserializer
mp.messaging.incoming.payments-in.value.jsonb.type=com.example.PaymentEvent

# 3. Failure strategy
# If consumer logic throws an exception, message goes to DLQ instead of stopping the app.
mp.messaging.incoming.payments-in.failure-strategy=dead-letter-queue

# (Optional) Custom DLQ topic name.
# Default: dead-letter-topic-payments-in
# mp.messaging.incoming.payments-in.dead-letter-queue.topic=custom-payments-dlt
```

#### Consumer Code

```java
import jakarta.enterprise.context.ApplicationScoped;
import org.eclipse.microprofile.reactive.messaging.Incoming;

@ApplicationScoped
public class PaymentConsumer {

    // Links this method to the 'payments-in' channel configured in application.properties.
    @Incoming("payments-in")
    public void consume(PaymentEvent event) {
        System.out.println("Consumed: " + event);

        // If this method throws an exception, the 'dead-letter-queue' failure strategy
        // will automatically catch it and send the message to the DLQ.
        if (event.getAmount() < 0) {
            throw new IllegalArgumentException("Payment amount cannot be negative.");
        }
    }
}
```

#### The `PaymentEvent` Class (for both examples)

```java
// A simple POJO for deserialization.
// Ensure it has a no-argument constructor for JSON deserializers.
public class PaymentEvent {
    private String transactionId;
    private double amount;

    // Getters, setters, and toString...
    public String getTransactionId() { return transactionId; }
    public void setTransactionId(String transactionId) { this.transactionId = transactionId; }
    public double getAmount() { return amount; }
    public void setAmount(double amount) { this.amount = amount; }
    
    @Override
    public String toString() {
        return "PaymentEvent{" +
                "transactionId='" + transactionId + '\'' +
                ", amount=" + amount +
                '}';
    }
}
```
