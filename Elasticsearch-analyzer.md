Build: Elasticsearch Cluster Workload, Shard Allocation & Ingest Analyzer

Objective

Build a production-quality diagnostic and optimization tool for Elasticsearch 8.14.x.

The tool connects to an Elasticsearch cluster using a user-provided URL and credentials, collects comprehensive cluster telemetry using Elasticsearch REST APIs, analyzes indexing workload, shard allocation, ingest pipelines, rollover/ILM behavior, disk distribution, thread pools, circuit breakers, JVM/CPU pressure, recoveries and other relevant metrics, and produces an actionable diagnosis.

The primary use case is a large Elasticsearch cluster with:

- 90 total nodes
- 3 dedicated master nodes
- 3 coordinating-only nodes
- 48 hot data nodes
- 36 warm data nodes
- Hot nodes have 16 CPU cores, 64 GB RAM, ~31 GB Elasticsearch heap and 5 TB disks
- Warm nodes have 16 CPU cores, 64 GB RAM, ~31 GB Elasticsearch heap and 13 TB disks
- Hot nodes also have the ingest role
- 1 replica
- ~25,000 indices
- ~21,890 shards currently
- ~520 current write aliases / write indices
- Rollover-based indices
- Typical rollover configuration around 50 GB maximum primary shard size, 5 GB minimum shard size and 1 day
- Data remains hot for approximately 1 day, then warm, then deleted after approximately 7 days
- Approximately 400,000 primary documents/sec and ~800,000 total primary+replica indexing operations/sec
- Average indexed storage per document around 1.3 KB, but document sizes vary widely
- Approximately 100 ingest pipelines
- Many pipelines contain grok, script and nested pipeline processors
- Clients include Beats, Jaeger and custom applications
- Only the 3 coordinating nodes are exposed to clients; hot/warm nodes are not directly exposed for ingestion

The main problems to diagnose are:

1. One or more hot nodes develop large write queues.
2. Current write primary shards may be unevenly distributed.
3. Some hot nodes appear significantly more utilized than others.
4. Rollover may be delayed.
5. Hot-to-warm movement may be delayed.
6. Circuit breakers may trigger.
7. Data may be dropped or rejected.
8. Disk usage is uneven across both hot and warm tiers.
9. Some nodes may be around 70–75% disk while others are around 25–30%.
10. There may be excessive shard movement/recovery.
11. Ingest pipelines may contribute substantial CPU pressure.
12. The cluster may have an imbalance between indexing workload and shard-count-based allocation.

The analyzer must determine whether the primary bottleneck is:

- shard allocation imbalance
- active primary write-load imbalance
- ingest pipeline CPU
- bulk/request pressure
- coordinating-node pressure
- CPU saturation
- JVM/heap pressure
- merge pressure
- disk I/O
- recovery pressure
- circuit breaker pressure
- rollover/ILM behavior
- oversharding
- undersharding
- insufficient hot-tier capacity
- allocation configuration
- or a combination of these.

---

1. Safety Requirements

The application MUST operate in read-only diagnostic mode.

Do NOT execute:

- PUT
- POST mutations
- DELETE
- cluster settings changes
- index settings changes
- shard rerouting
- allocation changes
- rollover
- ILM actions
- pipeline modifications

The only POST requests allowed should be read-only diagnostic APIs such as:

- "_cluster/allocation/explain"
- "_nodes/hot_threads"
- ingest pipeline "_simulate", only if explicitly enabled by the user and using user-provided sample documents

Default mode must be completely read-only.

Credentials must never be written to reports, logs, browser local storage, generated Excel files or HTML reports.

Support:

- Basic authentication
- API key authentication
- HTTPS
- optional CA certificate
- optional TLS verification toggle
- connection timeout
- request timeout
- configurable sampling interval

Prefer using a read-only Elasticsearch role.

---

2. Application Architecture

Build the project as a proper repository.

Recommended stack:

Backend:

- Go preferred
- Elasticsearch REST client or clean HTTP client
- concurrent API collection
- structured internal data model
- analysis engine
- report engine

Frontend:

- React/TypeScript preferred
- charts and tables
- sortable/filterable node and shard views

Reports:

- HTML
- XLSX
- optional JSON export

The tool should also work in CLI-only mode so it can be run against production without a browser.

Recommended commands:

es-analyzer connect
es-analyzer collect
es-analyzer analyze
es-analyzer report
es-analyzer serve

Or provide one command:

es-analyzer analyze

which performs:

connect
→ collect
→ sample
→ analyze
→ generate reports
→ start/display UI

---

3. Connection Screen

Create a connection screen with:

- Elasticsearch URL
- Authentication type
  - Basic Auth
  - API Key
- username
- password
- API key
- TLS verification
- CA certificate
- request timeout
- sampling duration
- sampling interval
- "Run Analysis" button

First perform:

GET /

Use this to determine:

- Elasticsearch version
- cluster name
- node name
- build version
- compatibility

The analyzer is primarily designed for Elasticsearch 8.14.x.

If the cluster version differs, display:

Detected Elasticsearch version: X

This analyzer was designed for Elasticsearch 8.14.x.
Some metrics/APIs/settings may differ.

Do not silently assume version compatibility.

---

4. Cluster Overview APIs

Collect:

GET /
GET /_cluster/health
GET /_cluster/health?level=indices
GET /_cluster/state/cluster_manager_node,nodes
GET /_cluster/pending_tasks
GET /_cluster/settings?include_defaults=true&flat_settings=true
GET /_cluster/settings?include_defaults=true&flat_settings=false

Collect and display:

- cluster name
- version
- cluster UUID
- cluster health
- number of nodes
- number of data nodes
- active shards
- relocating shards
- initializing shards
- unassigned shards
- delayed unassigned shards
- active primary shards
- pending cluster tasks
- cluster-manager/master node
- allocation settings
- discovery settings relevant to the analysis
- shard allocation settings
- rebalance settings
- disk watermark settings
- recovery settings
- awareness settings
- forced awareness
- allocation filtering
- tier preferences
- balancing weights
- write-load balancing settings
- total shards per node
- concurrent recovery settings
- cluster routing settings

The analyzer must distinguish between:

- explicitly configured settings
- default settings
- persistent settings
- transient settings

Highlight dangerous or unusual overrides.

---

5. Node Inventory

Collect:

GET /_cat/nodes?v&h=name,node.role,master,ip,heap.percent,heap.max,ram.percent,ram.max,cpu,load_1m,disk.used_percent,disk.avail,node.data,node.ingest

Also:

GET /_nodes

And:

GET /_nodes/stats

Prefer targeted node stats rather than repeatedly downloading unnecessary metrics.

Determine node roles accurately.

Classify nodes into:

- master
- coordinating-only
- hot
- warm
- ingest
- other

Do not infer roles only from node names.

Build a node inventory table:

Node
IP
Roles
CPU cores
Heap max
Heap used
RAM
Disk
Hot/warm/master/coordinating
JVM version
Elasticsearch version

Detect nodes with inconsistent:

- roles
- heap
- CPU
- disk capacity
- Elasticsearch version
- JVM version

---

6. Node Resource Analysis

Collect per-node:

GET /_nodes/stats/os,process,jvm,fs,indices,thread_pool,breakers,ingest,transport,http

Analyze:

CPU

- CPU percent
- load average
- process CPU
- JVM CPU
- CPU normalized by core count
- sustained high CPU
- CPU imbalance across nodes

Flag:

- «70% sustained»
- «80% concerning»
- «90% critical»

These are configurable heuristics, not absolute Elasticsearch limits.

Memory

Collect:

- JVM heap used
- heap max
- heap used percent
- GC
- old generation
- young generation
- GC collection count
- GC collection time
- circuit breaker usage
- OS memory
- process memory

Detect:

- sustained high heap
- GC pressure
- heap imbalance
- nodes with significantly higher heap than peers

Do NOT simply say "heap >75% means bad". Consider sustained behavior and GC.

---

7. Disk Analysis

Collect:

GET /_cat/allocation?v
GET /_nodes/stats/fs

Analyze:

- disk total
- disk available
- disk used
- disk used %
- primary size
- replica size
- shard count
- shards per node
- hot vs warm utilization

Calculate:

cluster average disk %
hot-tier average disk %
warm-tier average disk %
node deviation from tier average

Identify:

- nodes above cluster/tier average
- nodes below average
- nodes near low/high/flood-stage watermarks
- large disk imbalance
- shard-size concentration

Generate a disk distribution chart.

Calculate:

max disk usage - min disk usage
standard deviation
p50
p75
p90
p95

Do not treat the configured 70% watermark as a target utilization. Explain the difference between watermark behavior and desired steady-state balance.

---

8. Shard Allocation Analysis

Collect:

GET /_cat/shards?h=index,shard,prirep,state,node,store,docs,docs.deleted
GET /_cat/allocation?v

For large clusters, stream/process the response rather than loading unnecessary structures into memory.

Calculate per node:

- total shards
- primary shards
- replica shards
- total shard store size
- primary store size
- replica store size
- active write primary shards
- active write replica shards
- historical hot shards
- warm shards
- unassigned shards
- relocating shards

Generate:

shards/node
primary shards/node
disk/node
active write primaries/node
active write storage/node

---

9. Identify Current Write Indices

This is one of the most important analyses.

Collect:

GET /_alias

or equivalent alias metadata.

Identify aliases with:

is_write_index = true

Build:

alias
write index
number of primary shards
number of replicas
docs
store size
creation date
age
ILM policy
index lifecycle phase
rollover configuration

Determine:

- number of active write aliases
- number of current write indices
- primary shard count distribution
- average
- median
- p90
- p95
- max
- total active write primaries

Generate a histogram of primary shard counts per current write index.

---

10. Active Write Primary Distribution

This is a CORE feature.

Map every current write primary shard to its physical hot node.

Calculate:

active write primaries per node

Then:

cluster average
median
p50
p75
p90
p95
max
min
standard deviation
coefficient of variation

Also calculate:

node_write_primary_ratio =
node active write primaries / cluster average

Flag:

- «1.5x average»
- «2x average»
- «3x average»

But do not declare a node overloaded based on shard count alone.

---

11. Active Write LOAD Distribution

This is more important than shard count.

Use node/indexing statistics and shard statistics to estimate:

- indexing docs/sec per node
- indexing bytes/sec per node
- primary indexing rate per node
- replica indexing rate per node
- indexing time
- indexing current
- indexing totals

Calculate active-write load per hot node.

If direct per-shard rate cannot be retrieved from the available APIs, clearly state that limitation.

Use a sampling interval:

default = 60 seconds

Take two snapshots:

T0
T1

Calculate deltas:

docs/sec = (counter_T1 - counter_T0) / interval

Do not calculate rates from a single cumulative counter.

Generate:

Node
Active write primaries
Primary docs/sec
Total docs/sec
Primary bytes/sec estimate
Indexing time/sec
Write queue
CPU
Heap
Disk

Create a "Write Hotspot Score".

Example weighted score:

write_load_score =
  normalized_primary_docs_rate
+ normalized_primary_bytes_rate
+ normalized_write_queue
+ normalized_CPU
+ normalized_indexing_time

Make weights configurable.

Do not claim this score is an Elasticsearch-native metric.

---

12. Primary vs Replica Analysis

Because the cluster has one replica, analyze:

primary indexing
replica indexing

Estimate whether replica indexing is contributing to:

- CPU
- disk
- write queues
- merge activity

Compare:

primary docs/sec
replica docs/sec
total docs/sec

Flag unexpected differences.

---

13. Indexing Statistics

Collect:

GET /_nodes/stats/indices/indexing,merge,refresh,flush,translog,segments,store

Analyze per node:

Indexing

- index_total
- index_time_in_millis
- index_current
- throttle_time_in_millis

Merge

- current
- current_docs
- current_size
- total
- total_time_in_millis
- total_docs
- total_size
- total_stopped_time_in_millis
- total_throttled_time_in_millis

Refresh

- total
- total_time
- listeners

Flush

- total
- total_time

Translog

- operations
- size
- uncommitted operations
- uncommitted size

Segments

- count
- memory
- terms memory
- stored fields memory
- doc values memory
- points memory
- norms memory
- index writer memory
- version map memory

Identify nodes with abnormal merge/indexing pressure.

---

14. Thread Pool Analysis

Collect:

GET /_cat/thread_pool?v

and:

GET /_nodes/stats/thread_pool

Analyze at minimum:

- write
- search
- search_coordination
- management
- refresh
- flush
- merge
- warmer
- snapshot
- system_read
- system_write
- fetch_shard_store

For write:

- active
- queue
- completed
- rejected

Calculate rejection rate over the sampling period.

Do not just show current queue size.

Calculate:

queue_delta
rejected_delta
completed_delta

Flag:

- growing queue
- sustained queue
- increasing rejection
- queue oscillation
- queue only during recoveries
- queue correlated with CPU

---

15. Determine Whether Write Queue Is Ingest or Indexing Pressure

This is a critical analysis.

Collect:

GET /_nodes/stats/ingest

Analyze pipeline and processor statistics.

For each pipeline and processor collect, where available:

- execution count
- failures
- execution time
- processor type

Calculate:

average processor time =
total processor time / execution count

Identify expensive processors.

Rank:

pipeline by total processing time
pipeline by average processing time
processor by total processing time
processor by average processing time

Pay special attention to:

- grok
- script
- pipeline
- nested pipeline
- foreach
- json
- dissect
- date
- convert

For each pipeline estimate:

pipeline executions/sec
pipeline processing ms/doc
estimated CPU demand

Do not claim that processor time maps perfectly to CPU time; clearly label it as an approximation.

Compare ingest pressure with:

- CPU
- write queue
- primary indexing rate
- node role
- active write primaries

Decision logic:

Likely ingest bottleneck

If:

- ingest processing time is high
- expensive grok/script processors dominate
- CPU is high
- write queue is high
- active primary write load is not disproportionately high

Likely shard/write hotspot

If:

- write queue is high
- active primary write load is disproportionately high
- ingest processing is not unusually high
- node CPU/indexing is high

Combined

If both are high.

Do NOT make binary claims if the data is inconclusive.

Return:

Diagnosis:
HIGH CONFIDENCE
MEDIUM CONFIDENCE
LOW CONFIDENCE
INSUFFICIENT DATA

---

16. Ingest Pipeline Definition Analysis

Collect:

GET /_ingest/pipeline

Analyze pipeline definitions.

Build a pipeline dependency graph.

For example:

pipeline-A
 ├── grok
 ├── script
 └── pipeline-B
       ├── grok
       └── script

Detect:

- nested pipeline depth
- pipelines called by many other pipelines
- pipelines containing many expensive processors
- repeated grok
- repeated script
- potential redundant processing
- deeply nested pipelines
- cycles if detectable

Rank pipelines by:

complexity score
execution rate
estimated processing cost

Do not recommend replacing grok blindly.

Recommend "dissect" only when the pattern appears delimiter-structured and deterministic.

---

17. Bulk Request Analysis

Clearly distinguish what Elasticsearch can and cannot determine.

Elasticsearch node stats do not necessarily provide a direct authoritative:

bulk_requests/sec

metric per client/application.

The tool should not fabricate one.

Use available node statistics to estimate:

- indexing operations/sec
- HTTP requests/sec
- transport activity
- thread-pool activity

Collect:

GET /_nodes/stats/http
GET /_nodes/stats/transport

If possible, calculate HTTP request rate using sampling.

Explain that this is:

HTTP request rate

not:

Bulk request rate

Provide a recommendation:

«To obtain exact bulk requests/sec, client-side metrics, load-balancer metrics, APM, application metrics or request logs are required.»

If client metadata is available through external integrations, support optional import.

Also calculate approximate:

docs per HTTP request

only if the request type can be reliably identified.

Do not assume every HTTP request is a bulk request.

---

18. Coordinating Node Analysis

Since clients only connect to the 3 coordinating nodes, analyze them separately.

Collect:

GET /_nodes/stats/http,transport,thread_pool,os,process,jvm,indices

Determine:

- HTTP requests/sec
- HTTP open connections
- CPU
- heap
- GC
- write queue
- search queue
- indexing-related work
- transport activity

Calculate traffic distribution:

coord-01 %
coord-02 %
coord-03 %

Detect:

- uneven load balancing
- one coordinating node receiving significantly more traffic
- insufficient coordinating-node capacity
- high coordinating CPU
- high coordinating heap

---

19. Allocation Settings Analysis

Parse:

GET /_cluster/settings?include_defaults=true&flat_settings=true

Specifically analyze:

- cluster.routing.allocation.enable
- cluster.routing.rebalance.enable
- cluster.routing.allocation.allow_rebalance
- cluster.routing.allocation.balance.shard
- cluster.routing.allocation.balance.index
- cluster.routing.allocation.balance.threshold
- cluster.routing.allocation.balance.write_load
- cluster.routing.allocation.total_shards_per_node
- cluster.routing.allocation.node_concurrent_incoming_recoveries
- cluster.routing.allocation.node_concurrent_outgoing_recoveries
- cluster.routing.allocation.cluster_concurrent_rebalance
- cluster.routing.allocation.node_initial_primaries_recoveries
- cluster.routing.allocation.same_shard.host
- awareness settings
- forced awareness
- allocation filters
- disk threshold settings
- data tier settings

Compare configured settings against defaults.

Flag potentially restrictive settings.

Do not automatically recommend changing a setting merely because it differs from default.

Explain why the setting matters.

---

20. Allocation Explain

Do not call allocation explain for all ~20k shards.

Instead automatically identify representative problematic shards:

1. Primary write shard on hottest node
2. Large shard on highest disk node
3. Shard on lowest disk node
4. Relocating shard
5. Unassigned shard
6. Shard that appears unable to rebalance

Run:

POST /_cluster/allocation/explain

with appropriate shard/index information.

Capture:

- current node
- target allocation decisions
- YES
- NO
- THROTTLE
- deciders
- reasons

Explain which allocation decider is preventing movement.

---

21. Relocation / Recovery Analysis

Collect:

GET /_cat/recovery?v
GET /_cat/shards?v
GET /_cluster/pending_tasks

Analyze:

- number of relocating shards
- relocation source/target
- shard size
- recovery duration
- recovery stage
- bytes recovered
- files recovered
- translog recovery
- recovery rate

Determine whether recoveries correlate with:

- CPU
- disk
- write queue
- cluster instability

Flag excessive simultaneous recovery.

Do not blindly recommend increasing recovery concurrency.

---

22. Hot Threads

When a hotspot exists, optionally run:

GET /_nodes/<node-id>/hot_threads?threads=20&ignore_idle_threads=true&type=cpu

Also optionally:

GET /_nodes/<node-id>/hot_threads?threads=20&ignore_idle_threads=true&type=wait

Only run on:

- top 3 CPU nodes
- top 3 write queue nodes
- top ingest-processing nodes

Do not run against every node continuously.

Analyze hot-thread output for:

- grok
- Painless/script
- indexing
- Lucene merge
- refresh
- transport
- GC
- filesystem
- recovery

Store raw output separately.

---

23. Circuit Breaker Analysis

Collect:

GET /_nodes/stats/breaker

Analyze:

- request breake
