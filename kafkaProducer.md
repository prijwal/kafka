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

# Kafka Producer Acks & Durability

### 1. `acks` – Durability & Data Safety

`acks` controls how many broker replicas must confirm a write before it's considered successful.

-   **`acks=0` (Fire & Forget)**
    -   **What it is:** Producer sends and doesn't wait for a response.
    -   **Use Case:** Non-critical data where loss is acceptable (e.g., metrics, logging). Fastest but unsafe.

-   **`acks=1` (Leader Ack - Default)**
    -   **What it is:** Waits for the leader broker's confirmation only.
    -   **Use Case:** General messaging. A good balance of performance and durability, but with a small risk of data loss if the leader fails before replication.

-   **`acks=all` | `-1` (Full Durability)**
    -   **What it is:** Waits for the leader and all in-sync replicas (ISR) to confirm.
    -   **Use Case:** Critical data that cannot be lost (e.g., financial transactions, orders). Safest but slowest.

> ⚠️ **Note:**
> The only valid values for `acks` are `0`, `1`, and `all` (or `-1`).
> You cannot set `acks=2`, `acks=3`, etc. — the producer will fail with a `ConfigException`. To enforce how many replicas must ack, use `min.insync.replicas` on the broker side.

---

### 2. ✅ In-Sync Replicas (ISR)

-   **What they are:** The set of replicas (leader + followers) that are fully caught up with the leader’s log.
-   **Why it's important:** Only ISR members can become the new leader if the current leader fails.
-   **Controlled by:** Broker property `replica.lag.time.max.ms` (default: 10000ms = 10s). If a follower does not fetch data from the leader within this time, it is removed from the ISR.

---

### 3. `min.insync.replicas` – Stronger Safety Net

-   **What it is:** A broker-side setting (topic-level or cluster-wide).
-   **Purpose:** Defines the *minimum* number of ISR replicas that must acknowledge a write when a producer uses `acks=all`.
-   If the number of available ISRs drops below this value, producer requests with `acks=all` will fail with a `NotEnoughReplicasException`.

🔎 **Example:**
-   Replication factor = `3`
-   `min.insync.replicas=2`
-   Producer uses `acks=all`

✔ **Write succeeds** if at least 2 brokers (leader + 1 follower) confirm.
❌ If only the leader is alive, **writes are rejected**, protecting against silent data loss where data is written to a single replica that might fail.

---

### 4. `enable.idempotence=true` – The Modern Standard

This single setting provides **exactly-once, in-order delivery guarantees** per partition, preventing duplicates from retries.

-   **What it does:** Automatically sets `acks=all` and enables infinite retries (within `delivery.timeout.ms`).
-   **Recommendation:** **Always enable this for reliable systems.** It's the simplest way to build a robust producer. (If you manually override `acks` or `retries`, ensure they remain compatible with idempotence.)

---

### 5. Performance Tuning: Throughput vs. Latency

-   **For High Throughput (Batching):**
    -   **Settings:** `linger.ms=10` + `compression.type=snappy`
    -   **What it does:** Waits briefly to batch messages and compresses them, sending more data in fewer requests.
    -   **Use Case:** Analytics pipelines, bulk data ingestion, or any high-volume system.

-   **For Low Latency (Real-time):**
    -   **Settings:** `linger.ms=0`
    -   **What it does:** Sends messages immediately without any delay.
    -   **Use Case:** Interactive applications or request/response style messaging where immediate feedback is critical.

---

### ⚡ Recommended Configurations (Producer + Broker)

#### Spring Boot Producer (`application.yml`)

```yaml
spring:
  kafka:
    producer:
      bootstrap-servers: localhost:9092
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.apache.kafka.common.serialization.StringSerializer
      properties:
        enable.idempotence: true
        # acks: all is set automatically by idempotence
        linger.ms: 10
        compression.type: snappy
```

#### Quarkus Producer (`application.properties`)
```properties
# Kafka Broker Address
kafka.bootstrap.servers=localhost:9092

# Producer Channel Configuration
mp.messaging.outgoing.payments-out.connector=smallrye-kafka
mp.messaging.outgoing.payments-out.topic=payments
mp.messaging.outgoing.payments-out.key.serializer=org.apache.kafka.common.serialization.StringSerializer
mp.messaging.outgoing.payments-out.value.serializer=org.apache.kafka.common.serialization.StringSerializer

# Reliability Settings
mp.messaging.outgoing.payments-out.enable.idempotence=true
# mp.messaging.outgoing.payments-out.acks=all is set automatically
mp.messaging.outgoing.payments-out.linger.ms=10
mp.messaging.outgoing.payments-out.compression.type=snappy
```

#### Broker (`server.properties` or Topic-Level)
```properties
# Require at least 2 ISR acknowledgments for acks=all requests
min.insync.replicas=2

# Remove lagging followers from ISR after 10s of no fetching
replica.lag.time.max.ms=10000
```
