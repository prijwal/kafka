# Start Multiple Kafka Servers (KRaft Mode)

When running Kafka in **KRaft mode** (without ZooKeeper), each Kafka broker also participates in the metadata quorum. To run multiple brokers, you need to adjust each `server.properties` file.

---

## Steps

### 1. Copy the Config File
Go to your Kafka installation:
```bash
cd $KAFKA_HOME/config/kraft
cp server.properties server-1.properties
cp server.properties server-2.properties
cp server.properties server-3.properties
```

---

### 2. Update `server.properties`

#### (a) **Node ID**
Each Kafka broker must have a unique ID:
```properties
node.id=1   # Broker 1
node.id=2   # Broker 2
node.id=3   # Broker 3
```

---

#### (b) **Listeners**
```properties
listeners=PLAINTEXT://:9092,CONTROLLER://:9093
```

- `PLAINTEXT://:9092` → The **broker port** used by producers/consumers/other brokers.  
  Change per broker: `9092`, `9094`, `9096` …  
- `CONTROLLER://:9093` → The **controller port** used for internal quorum communication.  
  Change per broker: `9093`, `9095`, `9097` …

✅ Example for Broker 1:
```properties
listeners=PLAINTEXT://:9092,CONTROLLER://:9093
```

In distributed systems, quorum means “minimum number of nodes that must agree before making a decision.”

- If you have 3 brokers, a quorum is 2.  
- If you have 5 brokers, a quorum is 3.  

This ensures that decisions are not made by a single broker, and the cluster can still work if some brokers fail.  

👉 So quorum = **majority vote system**.

---

2. **What is Internal Quorum Communication?**  
Brokers talk to each other over the **`CONTROLLER://` ports** to:

- Elect a **controller leader** (who manages metadata like topics, partitions, configs).  
- Replicate **metadata logs** (like Raft log).  
- Keep cluster state in sync (which broker has which partitions, who is alive, etc.).  

This is **control-plane traffic**, not producer/consumer data.


Broker 2:
```properties
listeners=PLAINTEXT://:9094,CONTROLLER://:9095
```

---

#### (c) **Controller Quorum Voters**
```properties
controller.quorum.voters=1@localhost:9093,2@localhost:9095,3@localhost:9097
```
- Format: `<node.id>@<host>:<controller-port>`
- This defines which nodes participate in the metadata quorum (like a Raft cluster).  
- Example above:  
  - Node 1 uses port `9093`  
  - Node 2 uses port `9095`  
  - Node 3 uses port `9097`

---

#### (d) **Advertised Listeners**
```properties
advertised.listeners=PLAINTEXT://localhost:9092
```
- What clients (producers/consumers) use to connect.  
- Must match the **actual hostname/IP and port** accessible by clients.  
- Change per broker: `9092`, `9094`, `9096` …

---

#### (e) **Log Directories**
Each broker must have its own log directory:
```properties
log.dirs=/tmp/server-1/kraft-combined-logs
log.dirs=/tmp/server-2/kraft-combined-logs
log.dirs=/tmp/server-3/kraft-combined-logs
```

---

## 3. Start Brokers

Start each broker with its config file:

```bash
./bin/kafka-server-start.sh config/kraft/server-1.properties
./bin/kafka-server-start.sh config/kraft/server-2.properties
./bin/kafka-server-start.sh config/kraft/server-3.properties
```

---

## Summary

- **`PLAINTEXT`** → Client/broker communication.  
- **`CONTROLLER`** → Raft quorum internal communication.  
- **`controller.quorum.voters`** → Defines Raft cluster members.  
- **`advertised.listeners`** → Address/port that clients use to connect.  
- **`node.id` + `log.dirs`** → Must be unique per broker.  

Now you can run a **multi-broker Kafka cluster in KRaft mode** 🎉
