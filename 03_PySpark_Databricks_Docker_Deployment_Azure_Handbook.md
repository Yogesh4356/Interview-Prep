# Deloitte Interview Handbook 3 --- PySpark + Databricks + Docker + Deployment + Azure

## AI / ML Engineer II \| Cloud + Production Engineering

> **Goal:** Understand distributed data processing, Databricks,
> containerization, deployment and Azure architecture well enough to
> discuss a production ML system confidently.

------------------------------------------------------------------------

# 1. Production ML picture

``` text
Data Sources → Data Lake/Warehouse → Databricks/Spark → Features
→ Training → MLflow/Registry → Dockerized service or batch job
→ Azure compute → API Gateway/LB → Monitoring
```

Focus on responsibilities of layers, not memorizing every cloud service.

# 2. Spark architecture

Spark distributes processing across a driver and executors.

**Driver:** application logic, planning and coordination.

**Executors:** execute tasks and can cache data.

Avoid collecting huge datasets to the driver:

``` python
df.collect()  # dangerous for very large datasets
```

# 3. DataFrames

``` python
from pyspark.sql import functions as F

df = spark.read.parquet("data/")
df.select("customer_id", "amount")
df.filter(F.col("amount") > 1000)
```

Know select, filter, withColumn, groupBy, agg, join, orderBy, drop and
distinct.

# 4. Transformations vs actions

Transformations build a plan (`select`, `filter`, `groupBy`). Actions
trigger execution (`count`, `collect`, `show`, writes).

This lazy model enables Spark to optimize the execution plan.

# 5. Catalyst and explain plans

Conceptually:

``` text
Logical plan → Optimization → Physical plan → Tasks
```

Use `df.explain()` when diagnosing execution.

# 6. Partitions

Spark divides data into partitions; tasks process partitions.

Too few → underutilization. Too many → scheduling overhead/small-file
problems depending on workload.

# 7. Shuffle

Shuffle redistributes data between partitions and can involve
network/disk I/O.

Common causes: groupBy, joins, distinct and orderBy.

**Interview line:**

> "When optimizing Spark, I inspect shuffles, partitioning, join
> strategy, skew and the execution plan rather than changing settings
> blindly."

# 8. Repartition vs coalesce

`repartition(n)` generally reshuffles and redistributes data.

`coalesce(n)` is commonly used to reduce partitions with less movement.

Choose based on workload and output requirements.

# 9. Broadcast joins

If one side is sufficiently small:

``` python
from pyspark.sql.functions import broadcast
result = large_df.join(broadcast(small_df), "customer_id")
```

This can avoid a large shuffle. Do not broadcast data that is too large
for executor memory.

# 10. Window functions

``` python
from pyspark.sql.window import Window
window = Window.partitionBy("customer_id").orderBy(F.col("timestamp").desc())
df = df.withColumn("rn", F.row_number().over(window))
```

Useful for latest records, rankings and rolling features.

# 11. Spark performance checklist

1.  Inspect execution plan.
2.  Check partitions.
3.  Identify shuffles.
4.  Optimize joins.
5.  Filter/select early.
6.  Avoid unnecessary Python UDFs.
7.  Check data skew.
8.  Check small files.
9.  Cache only reused data.

# 12. Why Python UDFs can be slower

Python UDFs may introduce serialization and process-boundary overhead.
Prefer built-in Spark functions when possible.

# 13. Data skew

If one key has vastly more records than others, one partition can become
a straggler.

Possible approaches: salting, better partitioning, broadcast joins when
suitable and skew-aware strategies.

# 14. Delta Lake

Delta Lake provides a table/storage layer around data files with
features such as ACID transactions, schema enforcement/evolution, time
travel and reliable updates/deletes.

Conceptually:

``` text
Object storage + transaction log + data files
```

# 15. Parquet vs Delta

Parquet is primarily a columnar file format.

Delta is a table/storage layer that adds transaction and
table-management capabilities around data files.

**Interview answer:**

> "Parquet is a file format; Delta Lake adds transactional and
> table-management semantics on top of data files."

# 16. Databricks

Think of Databricks as a platform combining Spark, lakehouse/data
engineering, ML, workflows and governance.

Know:

-   notebooks
-   compute
-   jobs/workflows
-   Spark
-   Delta
-   Unity Catalog
-   MLflow

# 17. Unity Catalog

Centralized governance for data and AI assets: permissions, catalogs,
schemas, tables, models and lineage.

Production value: access control, auditing, discoverability and
governance.

# 18. MLflow

Track experiments, parameters, metrics, artifacts and models.

``` text
Experiment
 ├── Run A → params + metrics + model
 └── Run B → params + metrics + model
```

This makes model comparison and reproducibility easier.

# 19. Docker

Docker packages an application and its dependencies into an image.

``` text
Dockerfile → docker build → Image → docker run → Container
```

**Image:** packaged artifact/template.

**Container:** running instance of an image.

# 20. FastAPI Docker example

``` dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app/ ./app
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Production considerations: pin dependencies, small images, vulnerability
scanning, non-root execution where appropriate, externalized
configuration/secrets and health endpoints.

# 21. FastAPI deployment

``` text
Client → API Gateway/LB → FastAPI container → preprocessing → model → response
```

With replicas:

``` text
LB → Container 1
   → Container 2
   → Container 3
```

# 22. Health vs readiness

**Health/liveness:** is the process alive?

**Readiness:** can it accept traffic? For an ML service this may include
model-loaded state and required dependencies.

# 23. Batch vs online vs async

**Batch:** periodic large-scale scoring.

**Online:** immediate request/response.

**Async:** request → queue → worker → result.

Long OCR/document workflows often benefit from async processing.

# 24. CI/CD

``` text
Developer → Git branch → PR → CI
                          ├── tests
                          ├── lint/security
                          └── build
                               ↓
                         artifact/image
                               ↓
                              CD
                               ↓
                       Dev/QA/UAT/Prod
```

CI validates/builds; CD deploys the validated artifact.

# 25. Azure fundamentals

Think in layers:

-   **Compute:** VMs, containers, managed application compute
-   **Storage:** Blob/Data Lake
-   **Networking:** VNet, subnet, NSG, load balancing/gateway
-   **Identity:** Microsoft Entra ID, managed identity, RBAC
-   **Monitoring:** Azure Monitor, Application Insights and logs/metrics

# 26. Azure VM

A VM is a virtual server. Your Apache hands-on maps to:

``` text
Internet → Public IP → NSG → VM → Apache → index.html
```

This teaches compute, ports, inbound access and server processes.

# 27. NSG

Network Security Groups control network traffic using rules.

Production principle: expose only required ports and restrict
administrative access appropriately.

# 28. VNet and subnet

VNet is the private network boundary; a subnet segments it.

``` text
VNet
 ├── public-facing subnet
 └── private application subnet
```

Internal ML services can remain private while only a controlled endpoint
is exposed.

# 29. Authentication and authorization

``` text
Authentication = who are you?
Authorization = what are you allowed to access?
```

Azure RBAC governs permissions. Managed identities can avoid embedding
credentials in applications.

# 30. Secrets

Never hard-code API keys or passwords. Use secret management,
environment/configuration and managed identity where supported.

``` text
Code ≠ secrets
```

# 31. Azure ML-style architecture

``` text
Azure Storage/Data Lake
        ↓
    Databricks
        ↓
Feature Engineering
        ↓
     Training
        ↓
      MLflow
        ↓
  Model Registry
        ↓
 Deployment Endpoint
        ↓
    Monitoring
```

Exact services can vary by organization.

# 32. API Gateway

Acts as a controlled front door for APIs. Typical responsibilities:
routing, authentication, rate limiting, TLS termination, policies and
versioning.

# 33. Load balancing

``` text
Client → Load Balancer → multiple healthy service replicas
```

Useful for horizontal scaling and availability.

# 34. Autoscaling

Scale replicas based on signals such as CPU, memory, request rate, queue
depth or latency. ML workloads may also need GPU-related signals.

# 35. Monitoring

Three broad categories:

-   **Metrics:** latency, errors, CPU, memory, request rate
-   **Logs:** detailed events
-   **Traces:** request path across services

# 36. p50/p95/p99

If p50=100 ms, p95=300 ms and p99=700 ms:

-   50% of requests are ≤100 ms
-   95% are ≤300 ms
-   99% are ≤700 ms

Tail latency is often more important than averages for SLAs/SLOs.

# 37. Production ML API safeguards

Use:

-   request validation
-   authentication/authorization
-   timeouts
-   bounded retries
-   rate limiting
-   payload limits
-   structured logs/correlation IDs
-   health checks
-   monitoring
-   graceful errors

# 38. RAG/LLM deployment connection

``` text
Client → Gateway → Agent Service
                 ├── Retriever
                 ├── Vector DB
                 ├── Reranker
                 └── LLM
                      ↓
                   Response
```

Monitor retrieval latency, reranking latency, LLM latency, token/cost
usage, failures and quality metrics.

# 39. Docker + Azure production example

``` text
Developer → Git → CI → Docker image → Container Registry
→ Azure compute → Gateway/LB → FastAPI replicas → Model
```

Be able to draw and explain this without memorizing a specific Azure
product for every box.

# 40. Interview questions

### Why Docker?

Consistent packaging of application and dependencies.

### Image vs container?

Image is the packaged artifact; container is a running instance.

### What is a shuffle?

Redistribution of data between partitions, often with network/disk I/O.

### How optimize Spark?

Inspect plans, shuffles, partitioning, joins, skew, data volume and UDF
use.

### Delta vs Parquet?

Parquet is a columnar file format; Delta adds transactional/table
semantics.

### Why NSG?

To control network traffic to/from resources.

### Why API Gateway?

Controlled front door for routing, auth, rate limiting and policies.

### Why p99?

Tail latency affects worst-case user experience and SLA behavior.

# 41. Priority for this profile

### Must know

-   Linux/server basics
-   Docker
-   REST/FastAPI
-   CI/CD
-   VM
-   networking basics
-   VNet/subnet
-   NSG
-   authentication/authorization
-   storage
-   monitoring
-   load balancing
-   basic scaling

### Good to know

-   Databricks
-   Delta
-   Unity Catalog
-   MLflow
-   managed identities
-   API gateway
-   queues

### Lower priority

-   deep Kubernetes administration
-   advanced networking
-   Terraform internals
-   obscure Azure services
-   deep Spark internals

Focus on architecture and practical reasoning.
