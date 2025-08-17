# Kafka Send: Sync vs Async

## Sync vs Async Send

-   **Async** → `send()` returns a `Future` immediately; the producer doesn’t block.
-   **Sync** → `future.get()` or `.join()` waits for the Kafka ack (itself dependent on the `acks` setting).

🔑 **Sync/Async = producer-side waiting strategy.**

### Error Handling

-   **Async** → Handled in a callback (`whenComplete`, `thenAccept`).
-   **Sync** → An exception is thrown on `.get()`.

---

## Spring Boot Example

```java
@Service
public class PaymentProducer {

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    public void sendAsync(String msg) {
        // "payments" is the topic name
        kafkaTemplate.send("payments", msg) // returns CompletableFuture
            .whenComplete((result, ex) -> {
                if (ex != null) {
                    System.err.println("Failed: " + ex.getMessage());
                } else {
                    System.out.println("Message written successfully: "
                        + result.getRecordMetadata());
                }
            });
    }

    public void sendSync(String msg) {
        try {
            // This is a blocking call that waits for the broker acknowledgment
            kafkaTemplate.send("payments", msg).get();
            System.out.println("Sync sent: " + msg);
        } catch (Exception e) {
            System.err.println("Sync error: " + e.getMessage());
        }
    }

     public void sendSyncWithJoin(String msg) {
            // .join() is also blocking but throws an unchecked CompletionException
            // This avoids the need for a try-catch block if you don't want to handle it here
            kafkaTemplate.send("payments", msg).join();
            System.out.println("Sync (with join) sent: " + msg);
    }
}
```

---

## Quarkus Example

```java
import jakarta.enterprise.context.ApplicationScoped;
import org.eclipse.microprofile.reactive.messaging.Channel;
import org.eclipse.microprofile.reactive.messaging.Emitter;
import java.util.concurrent.CompletionStage;

@ApplicationScoped
public class PaymentProducer {

    @Channel("payments-out") // This channel is mapped to a Kafka topic in application.properties
    Emitter<String> emitter;

    // Async send
    public void sendAsync(String msg) {
        emitter.send(msg)
            .whenComplete((unused, ex) -> {
                if (ex != null) {
                    System.err.println("Failed: " + ex.getMessage());
                } else {
                    System.out.println("Message written successfully: " + msg);
                }
            });
    }

    // Sync send
    public void sendSync(String msg) {
        try {
            // send() returns a CompletionStage, which we block on to achieve sync behavior
            emitter.send(msg).toCompletableFuture().get();
            System.out.println("Sync sent: " + msg);
        } catch (Exception e) {
            System.err.println("Sync error: " + e.getMessage());
        }
    }

    public void sendSyncWithJoin(String msg) {
            // .join() is also blocking but throws an unchecked CompletionException
            emitter.send(msg).toCompletableFuture().join();
            System.out.println("Sync (with join) sent: " + msg);
    }
}
```

---



# Kafka Producer Acks 

### 1. `acks` – Durability & Data Safety

`acks` controls how many broker replicas must confirm a write before it's considered successful.

-   **`acks=0` (Fire & Forget)**
    -   **What it is:** Producer sends and doesn't wait for a response.
    -   **Use Case:** Non-critical data where loss is acceptable (e.g., metrics, logging). Fastest but unsafe.

-   **`acks=1` (Leader Ack - Default)**
    -   **What it is:** Waits for the leader broker's confirmation only.
    -   **Use Case:** General messaging. A good balance of performance and durability, but with a small risk of data loss if the leader fails before replication.

-   **`acks=all` (Full Durability)**
    -   **What it is:** Waits for the leader and all in-sync replicas to confirm.
    -   **Use Case:** Critical data that cannot be lost (e.g., financial transactions, orders). Safest but slowest.

---

### 2. `enable.idempotence=true` – The Modern Standard

This single setting provides **exactly-once, in-order delivery guarantees** per partition, preventing duplicates from retries.

-   **What it does:** Automatically sets `acks=all` and enables infinite retries (within `delivery.timeout.ms`).
    - override acks=1 and retries  # or a different value based on your use case if you are not good to go with `enable.idempotence=true` default configs 
-   **Recommendation:** **Always enable this for reliable systems.** It's the simplest way to build a robust producer.

---

### 3. Performance Tuning: Throughput vs. Latency

-   **For High Throughput (Batching):**
    -   **Settings:** `linger.ms=10` + `compression.type=snappy`
    -   **What it does:** Waits briefly to batch messages and compresses them, sending more data in fewer requests.
    -   **Use Case:** Analytics pipelines, bulk data ingestion, or any high-volume system.

-   **For Low Latency (Real-time):**
    -   **Settings:** `linger.ms=0`
    -   **What it does:** Sends messages immediately without any delay.
    -   **Use Case:** Interactive applications or request/response style messaging where immediate feedback is critical.

---

### ⚡ Recommended Configurations (Spring & Quarkus)

**For Maximum Reliability (Most Common):**
```yaml
# Spring Boot (application.yml)
spring:
  kafka:
    producer:
      bootstrap-servers: localhost:9092
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.apache.kafka.common.serialization.StringSerializer
      properties:
        enable.idempotence: true
        linger.ms: 10
        compression.type: snappy
```
```properties
# Quarkus (application.properties)
kafka.bootstrap.servers=localhost:9092
mp.messaging.outgoing.payments-out.connector=smallrye-kafka
mp.messaging.outgoing.payments-out.topic=payments
mp.messaging.outgoing.payments-out.key.serializer=org.apache.kafka.common.serialization.StringSerializer
mp.messaging.outgoing.payments-out.value.serializer=org.apache.kafka.common.serialization.StringSerializer
mp.messaging.outgoing.payments-out.enable.idempotence=true
mp.messaging.outgoing.payments-out.linger.ms=10
mp.messaging.outgoing.payments-out.compression.type=snappy
```
}
```
