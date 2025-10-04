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
```
Use cases:

- Inventory state
- Last known sensor reading
- User preferences

This reduces storage and ensures consistent state lookup across consumers.

---

### 3. **Materialize Views from Streams**

Instead of querying Kafka directly to answer business questions, create **materialized views** by processing the stream and writing the output to a database or state store.

You can use:

- ✅ **Kafka Streams**
- ✅ **ksqlDB**
- ✅ Sink connectors (JDBC, Elasticsearch, ClickHouse, etc.)

These tools allow you to build **stateful aggregations**, **joins**, and **transformations** in real time — without querying Kafka directly.

#### 🧪 Example with Kafka Streams (Java):

```java
KStream<String, Order> orders = builder.stream("orders");

KTable<String, Long> orderCounts = orders
    .groupByKey()
    .count(Materialized.as("order-counts-store"));

orderCounts.toStream().to("order-counts-output");
```
This creates a continuously updated table of order counts, where:

- `KStream` is your event stream (e.g., `orders`)
- `KTable` is your stateful view (e.g., `orderCounts`)
- You can **query** the state store or sink the results to a database

---

### 4. **Set Retention Strategically**

Kafka is not meant to store data forever unless there's a real use case (e.g., event replay, audit trail).

Keeping too much data in Kafka:

- Increases storage costs
- Slows down broker performance
- Causes consumer lag if new consumers need to replay months of data

Instead, tune your topic-level retention settings to match your business needs.

```properties
# Example retention settings
retention.ms=604800000   # 7 days
segment.bytes=1073741824 # 1 GB
```
> 📌 **Best Practice**: Move historical data to long-term storage like:
> 
> - Amazon S3 or Google Cloud Storage  
> - Snowflake, Redshift, or BigQuery  
> - Apache Iceberg / Hudi / Delta Lake  
> 
> Also consider setting up **tiered storage** if using managed Kafka (e.g., Confluent Cloud).


---

### ✅ Bonus Step 5 (Optional): Checklist for Teams

```markdown

## 🧰 Checklist for Teams

- [ ] Do not query Kafka directly for random access
- [ ] Use compacted topics only for latest-state use cases
- [ ] Materialize downstream views using Kafka Streams or sink connectors
- [ ] Offload raw data to cold storage (e.g., S3) if needed for historical analysis
- [ ] Tune topic retention settings for space efficiency
