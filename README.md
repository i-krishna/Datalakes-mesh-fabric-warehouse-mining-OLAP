# Lakehouse architecture 
<img width="562" height="450" alt="image" src="https://github.com/user-attachments/assets/b8c9586d-9f29-4e30-842a-94fad0748ad7" />

<img width="405" height="448" alt="image" src="https://github.com/user-attachments/assets/46d37cf0-d547-4004-9916-979509bb82d0" />


# Data compute

For data compute (Spark), the bottleneck isn't RAM — it's the network and disk:

Spark bottleneck chain:
S3 (object storage)
    ↓  ~10 GB/s (network)        ← usually the real bottleneck
Node RAM (DDR5)
    ↓  ~300 GB/s
CPU L3 cache
    ↓  ~1,000 GB/s
CPU cores

Spark jobs spend most of their time waiting for data to arrive from S3 over the network. The CPU cores are fast enough — they're just starved of data. This is why Spark parallelism (adding more machines) helps so much — you're adding more network connections to S3, not faster CPUs.

## CPU tenant isolation 
```
┌─────────────────────────────────────────────────┐
│ LEVEL 4: Kubernetes (orchestration)             │
│ ResourceQuota, LimitRange, Namespaces           │
│ "Customer 1 gets max 16 cores cluster-wide"     │
├─────────────────────────────────────────────────┤
│ LEVEL 3: cgroups v2 (Linux kernel)              │
│ Enforces CPU throttle + memory hard limits      │
│ "This container cannot use > 4 cores right now" │
├─────────────────────────────────────────────────┤
│ LEVEL 2: Linux namespaces (kernel)              │
│ Process, network, filesystem visibility         │
│ "This container cannot SEE other containers"    │
├─────────────────────────────────────────────────┤
│ LEVEL 1: CFS scheduler (Linux kernel)           │
│ Decides which process runs on which core        │
│ "Fair share of CPU time across all processes"   │
├─────────────────────────────────────────────────┤
│ LEVEL 0: Physical hardware (shared)             │
│ Cores, RAM, L3 cache — NO hardware partitioning │
│ "Everything shares the same silicon"            │
└─────────────────────────────────────────────────┘
```
The critical insight: every layer above Level 0 is software. The hardware itself is fully shared. This is the fundamental difference from GPU MIG.

```
GPU MIG              CPU (cgroups + K8s)
─────────────────────────────────────────────────────────
Partition unit    Hardware slice        Software container
Memory isolation  Physical (MMU)        Logical (cgroup limit)
CPU isolation     Dedicated SMs         Time-shared (throttle)
Cache isolation   L2 partitioned        L3 fully shared ← weak
Network isolation Dedicated NVLink      Shared NIC (QoS optional)
Noisy neighbour   Impossible            Possible (cache, I/O)
Strongest form    7x MIG on one A100    Dedicated node per tenant
Cost efficiency   7 tenants, 1 GPU      Many tenants, shared nodes
Use case          Inference serving     ETL, batch, data pipelines
```
