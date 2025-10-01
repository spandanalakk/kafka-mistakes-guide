# 📝 Mistake #1 — Overusing Kafka as a Database

![Kafka Streams](../images/kafka-streams-header.png)

Apache Kafka is an **event streaming platform**, not a transactional database.  
Yet one of the most common pitfalls engineers run into is treating Kafka topics as if they were tables you can query at will.

This section explains **why this is a mistake**, how it manifests in production, and how to avoid it with concrete patterns.

---

## 🔴 The Problem

Teams start writing producers that dump every possible event into Kafka and then try to:

- Query historical data directly from topics  
- Treat a topic as the “single source of truth”  
- Use it for random access reads instead of sequential streaming  

This approach quickly leads to:

- **Huge retention costs** (storing months/years of raw data)
- **Operational complexity** (topics balloon in partitions)
- **Slow consumers** and increased lag

> ⚠️ Kafka is **append‑only**, designed for high‑throughput, sequential reads.  
> It’s not meant for ad‑hoc queries like SQL databases.

---

## 🧯 Why It’s a Problem

- Kafka **does not index messages** for arbitrary lookups.
- High retention on large topics increases disk I/O and broker strain.
- “Replaying” topics for queries causes lag and timeouts.
- No built‑in transactional constraints like foreign keys or ACID.

In production, you’ll see symptoms like:

- Consumer lag spikes when a new query job starts  
- Brokers running out of storage  
- Retention misconfigurations causing accidental data loss  

---

## ✅ How to Avoid / Fix It

### 1. **Separate Streaming from Storage**
Use Kafka for **transporting events**. Persist the “source of truth” in a database or data lake.

```bash
# good pattern:
Producer --> Kafka Topic --> Stream Processor --> Database (OLAP / OLTP)

# avoid:
Producer --> Kafka Topic --> Query Topic like a DB
```

### 2. **Use Compact Topics for State**

If you need the **latest value per key** (e.g., user profile updates, product stock levels), use a **compacted topic** rather than full retention.  
Kafka will retain only the latest message per key, significantly reducing disk usage.

```properties
# Topic configuration
cleanup.policy=compact
retention.ms=-1  # Retain forever with compaction
