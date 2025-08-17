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

-   **`acks=all` (Full Durability)**
    -   **What it is:** Waits for the leader and all in-sync replicas (ISR) to confirm.
    -   **Use Case:** Critical data that cannot be lost (e.g., financial transactions, orders). Safest but slowest.

> ⚠️ **Note:**
> The only valid values for `acks` are `0`, `1`, and `all` (or `-1`). To enforce *how many* replicas must ack, use `min.insync.replicas` on the broker side.

---

### 2. ✅ Broker-Side Durability Controls (ISR & Min-Sync)

-   **In-Sync Replicas (ISR):** The set of replicas fully caught up with the leader. Only ISR members can be elected as the new leader, preventing data loss during failover.
-   **`min.insync.replicas`:** A **broker setting** that enforces the minimum number of ISRs required for a write to succeed when a producer uses `acks=all`. This prevents writing to a single broker that might fail, which would lead to data loss.

---

### 3. Fault Tolerance & Timeouts

These settings define how the producer behaves when it encounters transient errors, like network glitches or temporary broker unavailability.

-   **`retries`**
    -   **What it is:** The number of times the producer will resend a message upon encountering a *retryable* error (e.g., `NetworkException`, `LeaderNotAvailableException`).
    -   **Default:** `Integer.MAX_VALUE`. This means it retries effectively forever, until `delivery.timeout.ms` is hit.
    -   **Note:** It will **not** retry non-retryable errors like `SerializationException` (bad message format).

-   **`retry.backoff.ms`**
    -   **What it is:** The amount of time to wait before attempting a retry.
    -   **Default:** `100ms`.
    -   **Why it's important:** This prevents the producer from overwhelming a temporarily struggling broker with a rapid-fire "retry storm."

-   **`request.timeout.ms`**
    -   **What it is:** The maximum time the producer will wait for a response to a **single request**.
    -   **Default:** `30000ms` (30 seconds).
    -   **How it works:** If no response is received within this window, the producer considers the request failed and will initiate a retry (if `retries` > 0).

-   **`delivery.timeout.ms`**
    -   **What it is:** An **overall time limit** for a message's entire delivery process. The timer starts when `producer.send()` is called.
    -   **Default:** `120000ms` (2 minutes).
    -   **What it includes:** This timeout covers the time spent in the producer's local buffer (`linger.ms`), all network round-trips for the initial attempt and all subsequent retries, and all backoff periods. If this total time is exceeded, the delivery fails permanently.
 
-   **`linger.ms`**
    -   **What it is:** This setting controls how long the producer waits to collect more messages into a single batch before sending.
    -   **Default:** `0ms` (send immediately, no batching delay).
    -   How it works:
          - If `linger.ms=0` → each record is sent right away.
          - If `linger.ms>0` → the producer waits up to this delay to collect more records into a single batch, improving throughput at the cost of latency.
    -   **Example:**  With `linger.ms=10`, the producer waits up to 10ms for more messages before sending. If the batch fills earlier (due to batch.size), it is sent immediately without waiting the full 10ms.
#### The Golden Rule of Timeouts
For your producer to function correctly, the following must be true:

```
delivery.timeout.ms >= linger.ms + request.timeout.ms
```

-   **Why?** A message must be given at least *one full chance* to be delivered. The minimum time for one attempt includes:
    1.  The maximum time it could wait in a batch (`linger.ms`).
    2.  The maximum time for that one network request to complete (`request.timeout.ms`).

    If `delivery.timeout.ms` is less than this sum, a message could time out and fail before it even has a single complete opportunity to be sent and acknowledged by the broker.

---

### 4. `enable.idempotence=true` – The Modern Standard

This single setting provides **exactly-once, in-order delivery guarantees** per partition by preventing duplicates from retries.

-   **What it does:** Automatically configures the safest settings for you:
    -   `acks=all`
    -   `retries=Integer.MAX_VALUE`
    -   It also enforces in-order delivery.
-   **Recommendation:** **Always enable this for reliable systems.** It's the simplest way to get a robust producer that handles retries safely.

---

### 5. Performance Tuning: Throughput, Latency, and Compression

#### Batching (`linger.ms`)
-   **For High Throughput:** `linger.ms=10` (or higher).
-   **For Low Latency:** `linger.ms=0` (sends immediately).

#### Compression (`compression.type`)
Compression encodes message batches into a smaller size, reducing network bandwidth and broker disk usage at the cost of some CPU. The consumer receives the compressed data and decompresses it.

-   **`none` (default):** No compression. Lowest CPU, highest network/disk usage.
-   **`gzip`:** Strongest compression, but highest CPU overhead. Best for archival or when bandwidth is extremely limited.
-   **`snappy`:** Fast, lightweight, with a good compression ratio. **The most commonly used** for a great balance.
-   **`lz4`:** Very fast, low-latency compression. Excellent for real-time pipelines where speed is critical.
-   **`zstd` (Kafka 2.1+):** Modern algorithm with an excellent compression ratio and high speed. A top-tier choice for large-scale analytics.

#### Best Practices for Performance
-   **General Purpose (Most Workloads):** `snappy`
-   **Low Latency / Real-Time:** `lz4`
-   **Max Compression (Storage/Bandwidth is expensive):** `zstd`


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
        # --- Reliability ---
        enable.idempotence: true   # exactly-once guarantees per partition
        acks: all                  # wait for leader + ISR confirmations
        max.in.flight.requests.per.connection: 5  # safe with idempotence

        # --- Fault tolerance ---
        retries: 2147483647        # effectively "infinite"
        retry.backoff.ms: 100
        request.timeout.ms: 30000
        delivery.timeout.ms: 120000

        # --- Performance tuning ---
        linger.ms: 10              # small batching window (10ms)
        batch.size: 32768          # 32 KB (default 16 KB, can increase)
        compression.type: snappy   # good balance: speed + compression
```

#### Quarkus Producer (`application.properties`)
```properties
kafka.bootstrap.servers=localhost:9092

mp.messaging.outgoing.payments-out.connector=smallrye-kafka
mp.messaging.outgoing.payments-out.topic=payments
mp.messaging.outgoing.payments-out.key.serializer=org.apache.kafka.common.serialization.StringSerializer
mp.messaging.outgoing.payments-out.value.serializer=org.apache.kafka.common.serialization.StringSerializer

# --- Reliability ---
mp.messaging.outgoing.payments-out.enable.idempotence=true
mp.messaging.outgoing.payments-out.acks=all
mp.messaging.outgoing.payments-out.max.in.flight.requests.per.connection=5

# --- Fault tolerance ---
mp.messaging.outgoing.payments-out.retries=2147483647
mp.messaging.outgoing.payments-out.retry.backoff.ms=100
mp.messaging.outgoing.payments-out.request.timeout.ms=30000
mp.messaging.outgoing.payments-out.delivery.timeout.ms=120000

# --- Performance tuning ---
mp.messaging.outgoing.payments-out.linger.ms=10
mp.messaging.outgoing.payments-out.batch.size=32768
mp.messaging.outgoing.payments-out.compression.type=snappy
```

#### Broker (`server.properties` or Topic-Level)
```properties
# Reliability: require acknowledgments from at least 2 brokers in ISR
min.insync.replicas=2

# Replica management: kick out lagging followers after 10s
replica.lag.time.max.ms=10000

# Replication factor recommendation (set per topic)
# For critical data: RF=3 (leader + 2 followers)
# Example:
# bin/kafka-topics.sh --create --topic payments --partitions 6 \
#   --replication-factor 3 --config min.insync.replicas=2
