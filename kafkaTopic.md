# Apache Kafka Basics

## Create a Topic

``` bash
kafka-topics.sh --bootstrap-server localhost:9092   --create --topic product-events   --partitions 3 --replication-factor 2
```

## Update a Topic

-   **Increase partitions**:

``` bash
kafka-topics.sh --bootstrap-server localhost:9092   --alter --topic product-events --partitions 6
```

-   **Change retention** (by time or size):

``` bash
kafka-topics.sh --bootstrap-server localhost:9092   --alter --topic product-events --config retention.ms=43200000
kafka-topics.sh --bootstrap-server localhost:9092   --alter --topic product-events --config retention.bytes=1073741824
```

## Delete a Topic

``` bash
kafka-topics.sh --bootstrap-server localhost:9092   --delete --topic product-events
```

## Read Messages

-   From beginning, print key-value:

``` bash
kafka-console-consumer.sh --bootstrap-server localhost:9092   --topic product-events --from-beginning   --property print.key=true,key.separator=-
```

# Kafka Basics – Topics, Partitions, Brokers, Consumers & Consumer Groups

## 1. Kafka Cluster
- A Kafka **cluster** is made up of **brokers**.
- Each **broker** is a Kafka server (typically runs on its own machine/container).
- A broker stores **partitions** of topics.
- Each broker has an ID (e.g., `broker-1`, `broker-2`).

**Example:**
- 3 brokers in a cluster: `broker-1`, `broker-2`, `broker-3`.

---

## 2. Topic
- Producers **write messages** to a topic.
- Consumers **read messages** from a topic.
- Topics are **logically divided** into **partitions**.

**Example:**
```text
Topic: payments
Partitions: 2 → P0, P1
```

---

## 3. Partition
A **partition** is the smallest unit of parallelism in Kafka, more partions , more throughput ( read messages per sec) 

Each partition is an ordered, immutable sequence of messages.

Each message in a partition has an **offset** (a unique ID).

**Rules:**
- One partition → one consumer (per consumer group).
- If a consumer group has more consumers than partitions, the extras sit idle.
- If partitions < consumers → some consumers idle.
- If partitions > consumers → some consumers handle multiple partitions.

---

## 4. Replication
Partitions are replicated across brokers for fault tolerance.

One broker hosts the **leader** partition; others host **replicas**.

Producers/consumers only interact with the leader.

**Example:**
- **Topic:** `payments`
- **Partitions:** 2
- **Replication Factor:** 3
- **P0 leader** → broker-1; replicas on broker-2, broker-3.
- **P1 leader** → broker-2; replicas on broker-1, broker-3.

---

## 5. Consumer
- A **consumer** is an application instance that reads messages from Kafka.
- Each consumer is part of a **consumer group**.
- Within a group → partitions are divided among consumers.

---

## 6. Consumer Group
- A **consumer group** is a set of consumers sharing the same `group.id`.
- Kafka guarantees that each partition is consumed by exactly one consumer in the group.
- Different groups can consume the same topic independently.

**Example:**
- **Topic:** `payments` (2 partitions: P0, P1)

- **Group:** `fraud-detector`
  - `C1` → `P0`
  - `C2` → `P1`

- **Group:** `analytics-service`
  - `C3` → `P0`
  - `C4` → `P1`

Here:
- **Fraud detector** has 2 consumers → partitions split (C1 reads P0, C2 reads P1).
- **Analytics service** has its own 2 consumers → partitions split again (C3 reads P0, C4 reads P1).

---

## 7. Throughput & Scaling
Kafka scales horizontally by increasing partitions.

More partitions → more consumers can read in parallel.

**Throughput rule:** max consumers in a group = number of partitions.

Extra consumers sit idle.

**Example:**
- **Topic:** `payments`
- **Partitions:** 2
- **Group:** `fraud-detector`
- **Consumers:** 3

**Result:**
- Only 2 consumers actively read.
- 1 consumer sits idle.

---

**Explanation:**
- Both services listen to the same topic `payments`.
- **Fraud detector group** → splits partitions among its instances.
- **Analytics service group** → splits partitions among its own instances.
- Kafka ensures parallel but independent consumption.

---

# Kafka Configuration: Spring Boot vs Quarkus


## Spring Boot Configuration (application.yml)

### Example: `application.yml`

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      auto-offset-reset: earliest
```

### Explanation of Key Properties

*   `spring.kafka.bootstrap-servers`: Kafka broker(s) address.
*   `spring.kafka.consumer.key-deserializer` / `value-deserializer`: Deserialize Kafka byte data.
*   `spring.kafka.consumer.auto-offset-reset`: What to do when no offset is found.
    *   `earliest`: start from beginning.
    *   `latest`: read only new messages.
*   `group-id`: Defined **in code** with `@KafkaListener`.

### Consumer Example

```java
@Service
public class FraudDetectorConsumer {

    @KafkaListener(
        topics = "payments",
        groupId = "fraud-detector"   // group-id here
    )
    public void consume(String message) {
        System.out.println("Fraud Detector received: " + message);
    }
}
```

*   Here, multiple **instances** of `FraudDetectorConsumer` (C1, C2) can be deployed.
*   If topic `payments` has 2 partitions → C1 reads P0, C2 reads P1.
*   Another consumer group (e.g., `analytics-service`) can **independently** consume all partitions.

---

## Quarkus Configuration (application.properties)

### Example: `application.properties`

```properties
# Kafka Cluster Address
kafka.bootstrap.servers=localhost:9092

# Fraud Detector Consumer Configuration
mp.messaging.incoming.fraud-detector-in.connector=smallrye-kafka
mp.messaging.incoming.fraud-detector-in.topic=payments
mp.messaging.incoming.fraud-detector-in.group.id=fraud-detector
mp.messaging.incoming.fraud-detector-in.value.deserializer=org.apache.kafka.common.serialization.StringDeserializer
mp.messaging.incoming.fraud-detector-in.auto.offset.reset=earliest

```

### Explanation

*   `mp.messaging.incoming.<channel>.topic`: Binds a channel to a Kafka topic.
*   `mp.messaging.incoming.<channel>.group.id`: Defines the consumer group.
*   `fraud-detector-in`: The **channel name** (acts like a logical binding between code & topic).

### Consumer Example

```java
import org.eclipse.microprofile.reactive.messaging.Incoming;
import jakarta.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class FraudDetectorConsumer {

    @Incoming("fraud-detector-in")   // channel mapped in application.properties
    public void consume(String message) {
        System.out.println("Fraud Detector received: " + message);
    }
}
```

*   Here, `fraud-detector-in` is the **channel**.
*   Channels abstract Kafka details → you can switch topics/brokers without touching code, only config.

---

## 4. Comparison: Spring Boot vs Quarkus

| Feature           | Spring Boot                                | Quarkus (SmallRye)                         |
| ----------------- | ------------------------------------------ | ------------------------------------------ |
| Broker Config     | `spring.kafka.bootstrap-servers`           | `kafka.bootstrap.servers`                  |
| Consumer Group    | `@KafkaListener(groupId="...")`            | `mp.messaging.incoming.<channel>.group.id` |
| Topic Binding     | Direct in `@KafkaListener` annotation      | Done via channel in config                 |
| Code Annotation   | `@KafkaListener(topics="...")`             | `@Incoming("channel-name")`                |
| Channel Concept   | Not present                                | Present (abstraction over topics)          |




