# 📝 Mistake #3 — Too Many Kafka Partitions Doesn’t Equal More Performance

![Too Many Partitions](../images/kafka-too-many-partitions.png)

Kafka allows you to split topics into partitions, enabling parallelism and scalability.  
But many teams mistakenly assume:  
> "More partitions = faster throughput"

This **isn’t always true** — and can actually hurt performance, reliability, and cluster stability.

Let’s explore why.

---

## 🔴 The Problem

When designing topics, developers often:

- Default to **100+ partitions** for "future scalability"
- Increase partitions thinking it will boost performance
- Use **one partition per key/entity**
- Assume partitions are lightweight

### In reality:
- Too many partitions = more overhead  
- Kafka brokers and consumers **slow down under high partition count**
- Partition expansion **doesn't rebalance data retroactively**

---

## 🧯 Why It’s a Problem

Kafka treats each partition as a separate log, thread, and file on disk.

Having too many:

- Wastes CPU and memory on brokers  
- Increases controller load  
- Slows down topic creation, leader elections, reassignments  
- Leads to **longer recovery times** during restarts  
- Makes consumer groups slower to rebalance

> ⚠️ Even with no data, a topic with 1000 partitions costs system resources.

---

## ✅ How to Avoid / Fix It

Partition count should be a **carefully calculated decision**, not a guess.

---

### 1. **Estimate Partitions Based on Throughput + Consumers**

Start with:
```bash
# Formula:
partitions = target_throughput / max_per_partition_throughput
```
Or, if you want parallelism:

```bash
# Example:
partitions = number_of_consumers * expected_parallelism
```
Common benchmarks:
- 5–10 MB/sec per partition
- 1,000–10,000 msgs/sec per partition
- Ideal: Keep total partitions per broker under 4,000 (includes all topics)

### 2. Avoid “One Partition per Key” Design

Don’t create a partition for every customer, tenant, or product.
Better: Use keys to group data and let Kafka hash them to a small, fixed number of partitions.
```python
# GOOD: Use key for routing
producer.send("orders", key="user_42", value="order123")

# BAD: One topic or partition per user
topic = f"orders-user-{user_id}"
```

### 3. Monitor Partition Explosion

Set up alerts for:
- Topics with unusually high partition counts
- Total partitions per broker
- Controller CPU / ZooKeeper pressure
- Slow rebalance or ISR shrinkage

Use tools like:
- `kafka-topics.sh --describe`
- Cruise Control
- Confluent Control Center
- Grafana dashboards

### 4. Split Topics, Not Just Partitions

Sometimes, splitting topics makes more sense than exploding partitions.
Example:
```text
orders-us-east     → 24 partitions  
orders-eu-west     → 24 partitions  
orders-apac        → 24 partitions
```
This limits per-topic overhead and gives better region-based control.


### 5. Know That Adding Partitions Later ≠ Rebalancing

Kafka does not redistribute old data when partitions are added.

> If you start with 4 partitions and later expand to 20,
> the original 4 will still have most of the old data + load.

So, plan ahead — or manually reprocess/migrate if needed.

## 🧰 Checklist for Teams

- [ ] Estimate partitions using throughput or consumer count  
- [ ] Keep partitions per broker within reasonable limits  
- [ ] Avoid “one partition per key” anti-pattern  
- [ ] Monitor partition count growth  
- [ ] Understand that adding partitions later won’t rebalance existing data  

---

## 📚 References & Further Reading

- [Kafka Docs: Partitions](https://kafka.apache.org/documentation/#intro_topics)  
- [Confluent: Partitioning Best Practices](https://www.confluent.io/blog/how-to-choose-the-number-of-topic-partitions-in-a-kafka-cluster/)  
- [KIP-336: Partition Reassignment Tooling](https://cwiki.apache.org/confluence/display/KAFKA/KIP-336%3A+Improve+ReassignPartitionsTool)










