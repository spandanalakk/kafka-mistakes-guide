# 📝 Mistake #5 — Ignoring Retention Settings (and Losing Data Unexpectedly)

![Kafka Retention Diagram](../images/kafka-retention-settings.png)

Kafka doesn’t store data forever. If your team doesn’t explicitly configure **retention policies**, you may end up **losing critical messages** — silently.

---

## 🔴 The Problem

By default, Kafka topics retain data for **7 days** (`retention.ms = 604800000`). If your consumers are down, slow, or start late — they may **miss data entirely**.

Worse: if you're relying on Kafka as a backup or source of truth, this could mean:

- Lost customer events  
- Missed analytics  
- Broken pipelines  
- Incomplete data in downstream systems

> ⚠️ Kafka is a buffer, not a warehouse. Retention must be **explicitly** defined.

---

## 🧯 Why It Happens

Kafka has **multiple retention configurations** that vary per topic:

- `retention.ms`: How long to keep data  
- `retention.bytes`: Max size before Kafka deletes old segments  
- `segment.ms`: Frequency of log segment rolling  
- `delete.retention.ms`: Grace period before deletion (for compaction)

If your retention is misconfigured, you risk:

- Data deleted too early  
- Storage exhausted (if retention too long)  
- Consumers falling behind permanently  
- Surprise behavior in compacted topics

---

## ✅ How to Avoid / Fix It

---

### 1. **Explicitly Set Retention Per Topic**

Don’t rely on defaults. Set retention based on business needs.

```bash
# Set time-based retention: 30 days
bin/kafka-configs.sh --bootstrap-server <BROKER> \
  --entity-type topics \
  --entity-name user-events \
  --alter \
  --add-config retention.ms=2592000000
```
Use `kafka-topics.sh --describe` to verify settings.


### 2. Use Compaction for State Topics

If you only care about latest value per key, enable log compaction.

```bash
# Enable compaction
cleanup.policy=compact
retention.ms=-1  # Infinite retention (only latest value kept per key)
```
This is great for:

- Inventory state
- User profile updates
- IoT sensor readings

Compaction avoids bloat and keeps data fresh.

3. Monitor Topic Retention Usage

Use Prometheus or other monitoring tools to track:

- Retention window remaining
- Disk usage per broker
- Partitions approaching size limit
- Segment file counts

Sample Prometheus metrics:
```arduino
kafka_log_log_size{topic="orders",partition="2"} 83493312
kafka_topic_partition_current_offset{...}
```

4. Document Retention SLAs for Stakeholders

Make it clear to downstream users how long data will be available.

Sample SLA:

```
Topic: order-events  
Retention: 14 days  
Backup: Redshift ETL (daily)  
Consumers must process within 3 days or risk loss
```

This builds trust and avoids surprises.

### 🧰 Checklist for Teams

- Set retention.ms explicitly for all critical topics
- Use compaction where appropriate
- Monitor disk and segment usage
- Communicate retention SLAs with stakeholders
- Don’t treat Kafka as a data warehouse

## 📚 References

- [Kafka Topic-Level Configs](https://kafka.apache.org/documentation/#topicconfigs)  
- [Confluent Docs: Log Retention](https://docs.confluent.io/platform/current/kafka/log-retention.html)  
- [Retention vs Compaction](https://www.confluent.io/blog/kafka-fastest-messaging-system/)  
- [Monitoring with Prometheus](https://github.com/danielqsj/kafka_exporter)


## 💡 TL;DR
Kafka deletes data — by design.
Configure retention like your data depends on it.





