---
title: "🚀 Optimizing PySpark Shuffle Partitions on AWS Glue: A 32% Speed Boost Without Extra Workers"
date: 2025-05-01
---

At my team, we rely on a modern data stack built on AWS: Step Functions for orchestration, Athena for ad-hoc queries, S3 for storage, Lambda for lightweight processing, and PySpark jobs running on AWS Glue for heavier data transformations.

Recently, we encountered performance issues with one of our production Glue jobs and managed to cut execution time by 32%—from 40 to 25 minutes—without increasing worker count. Instead, we optimized the number of shuffle partitions. Here’s how we approached it.


## 🧩 The Problem

We have a PySpark job that:

- Performs an **inner join** between two tables:
  - `smaller_table`: **8.2 GB**
  - `bigger_table`: **827 GB**
- Selects a few columns
- Writes the result to **S3 in Parquet format**, partitioned by `source_type`

The total data volume is about **835 GB**, which is too large for AWS Lambda or Athena INSERTs to handle efficiently.

This job runs on AWS Glue with:

- **26 G2X workers**, each with **8 vCPUs and 32 GB RAM**
- Total resources: **208 vCPUs and 832 GB RAM**





## 🔧 Original Configuration: 500 Shuffle Partitions

We initially had this Spark configuration:

```bash
--conf spark.sql.shuffle.partitions=500
```

This means Spark divides the shuffled data into 500 partitions:

- ~**1.67 GB per partition** (835 GB / 500)
- Each executor runs 8 tasks concurrently
- So each executor needs ~**13.36 GB RAM** for shuffle data

Since only ~30% of executor memory is usable for execution (per [Apache Spark memory docs](https://medium.com/analytics-vidhya/apache-spark-memory-management-49682ded3d42)), each executor has ~12.8 GB execution memory — **not enough**, causing **disk spills**.

**Execution time: 37 minutes**

![image](https://github.com/user-attachments/assets/a903f45a-0080-4d1c-ac06-122517e14d9d)

## ⚙️ Improved Configuration: 1000 Shuffle Partitions

We updated the configuration:

```bash
--conf spark.sql.shuffle.partitions=1000
```

Now:

- Each partition is ~**855 MB**
- 8 tasks per executor → ~**6.7 GB RAM** required
- Fits comfortably in memory, avoiding spills

**Execution time: 25 minutes**  
✅ Same number of workers  
✅ No disk spills  
✅ Better data distribution  
✅ Same CPU/memory usage, just more efficient execution

![image](https://github.com/user-attachments/assets/8be4aac0-c85d-4ba4-93ce-ed415e120e75)

We still noticed CPU and data movement during the write to S3, but overall, shuffle performance improved significantly.


## 💡 Key Takeaways

- Tuning `spark.sql.shuffle.partitions` is a **simple and effective optimization** for large datasets.
- The default value (200) is often too low.
- Consider partition size per executor and available memory to **avoid shuffle spills**.


## 🧠 Conclusion

By increasing the shuffle partitions from 500 to 1000, we reduced job run time by **32%** and avoided adding more infrastructure.

If you're working with Spark on large datasets—especially in AWS Glue—**tune your shuffle settings** before scaling up your cluster.  
Sometimes, smarter partitioning is more effective than more hardware.

A future improvement that would be easy to implement would be partitioning the source tables to reduce shuffle data.

---

*Thanks for reading! Feel free to reach out if you're tackling similar challenges.*
