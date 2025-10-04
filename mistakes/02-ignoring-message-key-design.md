# 📝 Mistake #2 — Ignoring Message Key Design

In Kafka, every message can have an optional **key**. And that key **matters a lot**.

Ignoring or poorly designing your message key is one of the most common and subtle mistakes that teams make when using Kafka — leading to **partition imbalance**, **ordering issues**, and **unexpected consumer behavior**.

This section explains why key design is critical, and how to choose effective keys based on real-world scenarios.

---

## 🔴 The Problem

Many developers:

- Send Kafka messages **without a key**
- Use meaningless/random UUIDs as keys
- Hardcode a single key for all messages (e.g., `"constant"`)

### This leads to:
- All messages going to a single partition (hot spot)  
- No ordering guarantees across related messages  
- Poor parallelism across consumers  
- Uneven broker workload  

> 💥 Even in large clusters, improper keying leads to 80% of messages landing on 1 or 2 partitions. That kills throughput.

---

## 🧯 Why It’s a Problem

Kafka’s partitioning model relies on **keys** to determine which partition a message goes to.

If the key is:
- ❌ Missing → Round-robin behavior (no control over where it lands)
- ❌ Constant → All messages go to one partition (bottleneck)
- ❌ Random → Poor ordering + low cache/locality hit rate

This leads to:
- Backlogged partitions  
- Broker storage imbalance  
- Performance degradation under load  
- Inability to guarantee ordering for critical flows (e.g., user updates)

---

## ✅ How to Avoid / Fix It

The key takeaway: **design your Kafka keys purposefully**, based on the **domain entity** you're tracking.

Here are the strategies.

---

### 1. Key by Natural Domain ID

If you’re tracking a user, order, product, or session — use its ID as the Kafka key.

```python
# Good
producer.send("user-activity", key="user_123", value="clicked button")

# Bad
producer.send("user-activity", value="clicked button")  # no key
