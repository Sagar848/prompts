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

- request breaker
- fielddata breaker
- in-flight requests
- accounting
- parent
- model inference if applicable
- any available breakers in the detected version

For each:

- limit
- estimated
- tripped
- overhead

Use sampling where counters exist.

Detect:

trips increasing over time

Do not recommend simply increasing breaker limits.

Explain that circuit breaker trips are symptoms of memory pressure and the underlying workload must be investigated.

---

24. JVM / GC Analysis

Collect:

GET /_nodes/stats/jvm

Analyze:

- heap used
- heap max
- heap percent
- GC collectors
- collection count
- collection time
- pool usage
- old generation
- young generation

Calculate deltas over sampling interval.

Identify:

- GC-heavy nodes
- heap pressure
- nodes whose heap behavior differs substantially from peers

---

25. Search Pressure

Even though the current issue is indexing, search workload can steal resources.

Collect:

GET /_nodes/stats/indices/search

Analyze:

- query_total
- query_time_in_millis
- query_current
- fetch_total
- fetch_time_in_millis
- fetch_current
- scroll_total
- scroll_time
- open scroll contexts

Calculate query/sec and average query latency.

Determine whether search workload overlaps with indexing hotspots.

---

26. Refresh / Merge / Segment Pressure

Analyze:

- refresh rate
- refresh time
- refresh listeners
- merge count
- merge time
- merge throttle
- segment count
- index writer memory
- version map memory

Identify nodes where indexing is healthy but merge pressure is causing CPU/disk saturation.

---

27. Translog Analysis

Analyze:

- translog size
- uncommitted operations
- uncommitted size
- flush frequency

Flag abnormal translog growth.

---

28. Index-Level Analysis

Collect index-level stats for all indices where practical:

GET /_stats/docs,store,indexing,merge,refresh,translog,segments

For large clusters, make collection configurable and avoid excessive repeated downloads.

Calculate per index:

- docs
- deleted docs
- store size
- primary store size
- indexing rate
- indexing time
- merge time
- refresh time
- primary shard count
- replica count

Rank indices by:

- indexing rate
- storage size
- indexing time
- merge pressure
- active write status

---

29. Shard Size Analysis

For primary shards calculate distribution:

min
p25
median
p75
p90
p95
p99
max

Do separately for:

- hot current write shards
- hot historical shards
- warm shards
- primary shards
- replicas

Identify:

- tiny shards
- normal shards
- large shards
- oversized shards

Compare actual shard size against the configured 50 GB target.

---

30. Rollover Analysis

Retrieve:

GET /_ilm/policy

and relevant index settings:

GET /<relevant-index-pattern>/_settings?flat_settings=true

Use:

GET /<index>/_ilm/explain

for representative current and recently rolled indices.

Analyze:

- rollover condition
- max_primary_shard_size
- max_age
- max_docs if configured
- min conditions
- actual rollover reason
- time since index creation
- current shard size
- current indexing rate
- current ILM phase
- phase execution time
- indices stuck in hot
- indices delayed in warm
- indices waiting for allocation

Detect:

rollover age exceeded
but rollover condition not satisfied

and distinguish whether this is caused by:

- size
- age
- docs
- ILM execution
- cluster health
- write pressure

---

31. Hot-to-Warm Analysis

Determine:

- number of indices expected to be warm
- number actually warm
- indices stuck in hot
- age of oldest hot index
- age distribution
- warm-node disk distribution
- allocation delays

Compare:

expected lifecycle state
vs
actual lifecycle state

Highlight ILM backlog.

---

32. Index Template Analysis

Collect:

GET /_index_template
GET /_template

where applicable.

Analyze:

- shard count
- replica count
- refresh interval
- codec
- routing settings
- index sorting
- default pipeline
- final pipeline
- ILM policy
- rollover configuration
- tier preference

Detect templates creating unnecessary shard counts.

---

33. Data Stream Analysis

Also collect:

GET /_data_stream

Determine whether any workloads are using data streams rather than aliases.

Analyze:

- backing indices
- generation
- templates
- lifecycle policy
- rollover configuration

---

34. Mapping Complexity

Analyze representative/high-volume index mappings:

GET /<index>/_mapping

Do not download all mappings if enormous.

Identify:

- extremely high field counts
- dynamic mapping explosion
- many text fields
- many keyword fields
- excessive nested/object fields
- multi-fields
- runtime fields

Flag potential mapping complexity.

Do not claim mapping complexity is causing the current problem without correlated evidence.

---

35. Fielddata / Doc Values

Use node/index stats to identify:

- fielddata memory
- doc values memory
- global ordinals
- segment memory

This is particularly important for heap pressure.

---

36. Snapshot Analysis

If snapshots are configured, optionally inspect:

GET /_snapshot
GET /_snapshot/<repository>/_status

Determine whether snapshot/repository activity may compete for resources.

Do not perform snapshot operations.

---

37. Tasks API

Collect:

GET /_tasks?detailed=true&actions=indices:*

and inspect running tasks.

Look for:

- reindex
- update_by_query
- delete_by_query
- recovery-related work
- force merge
- snapshot
- heavy searches
- cluster state operations

Highlight unexpected background workloads.

---

38. Pending Cluster Tasks

Analyze:

GET /_cluster/pending_tasks

If tasks remain pending or queue time is high, flag:

cluster state / cluster-manager pressure

---

39. Cluster State Size

Where practical, inspect cluster state metadata size.

Identify whether:

- 25k indices
- 21k shards
- templates
- mappings
- aliases

are generating unusually large cluster state.

Do not claim cluster state is a problem solely from index count.

Correlate with:

- cluster-manager CPU
- cluster-state update latency
- pending tasks

---

40. Shard Count Health

Calculate:

total shards / node
shards / GB heap
shards / CPU core

But explicitly state these are diagnostic ratios, not hard Elasticsearch limits.

Analyze:

- total shards
- primaries
- replicas
- active shards
- shards per hot node
- shards per warm node
- shards per master node

---

41. Shard Allocation Fairness

Calculate fairness separately for:

Total shards

Primary shards

Active write primaries

Disk usage

Primary disk usage

Estimated indexing load

Use statistical measures:

mean
median
standard deviation
coefficient of variation
Gini coefficient
max/min ratio

Generate charts.

This is essential.

A node having 70 active write primaries is NOT automatically a problem.

The analyzer must compare it with:

- cluster average
- node indexing rate
- bytes/sec
- CPU
- write queue

---

42. Write Hotspot Score

Create an explainable score.

Example:

Write Hotspot Score =
  30% normalized primary indexing rate
  20% normalized indexing bytes/sec
  20% write queue
  15% CPU
  10% ingest processing cost
  5% merge pressure

Make the weights configurable.

Display each component so users understand why a node was classified as a hotspot.

Never present this as an Elasticsearch-native score.

---

43. Disk Hotspot Score

Create:

Disk imbalance score

based on:

- disk used %
- shard store size
- primary store size
- deviation from tier average

Again, explain the score.

---

44. Correlation Engine

This is one of the most important parts.

Correlate:

write queue
CPU
heap
GC
ingest processing
primary indexing
replica indexing
merge activity
disk
recovery
rollover
ILM

over the sampling window.

For example:

CPU ↑
   +
ingest time ↑
   +
write queue ↑
   +
primary indexing normal

→ likely ingest bottleneck.

Or:

CPU ↑
   +
primary indexing MB/sec ↑
   +
active write primaries ↑
   +
ingest time normal

→ likely shard/write hotspot.

Or:

disk IO ↑
   +
merge time ↑
   +
write queue ↑

→ likely merge/storage pressure.

---

45. Sampling

Do not rely solely on instantaneous values.

Default sampling:

interval = 30 seconds
duration = 2 minutes

Allow configuration:

15 sec
30 sec
60 sec
5 min

For production clusters, provide:

Quick Analysis
2 samples
30 seconds

Standard Analysis
5 samples
30 seconds

Deep Analysis
10 samples
30 seconds

Use counter deltas for rates.

---

46. Query Efficiency

The cluster is large.

Do NOT execute expensive APIs repeatedly.

Use:

- concurrent collection
- caching
- sampling
- targeted APIs
- pagination where available
- one-time collection of static metadata
- repeated collection only for dynamic counters

Separate:

Static data

from:

Dynamic data

Static:

- node roles
- templates
- pipelines
- ILM policies
- cluster settings
- aliases

Dynamic:

- CPU
- indexing
- thread pools
- ingest
- JVM
- disk
- recovery

---

47. Raw Data Storage

Save every API response in a local analysis bundle:

analysis/
  metadata/
  cluster/
  nodes/
  shards/
  indices/
  pipelines/
  ilm/
  templates/
  threadpools/
  recoveries/
  allocation/
  raw/

Sensitive fields must be redacted.

Never store:

- passwords
- API keys
- Authorization headers

Allow users to export this bundle as ZIP.

---

48. Executive Report

The first report page should answer:

Is the cluster healthy?

HEALTHY
WARNING
CRITICAL

Then:

Top 5 Problems

Example:

1. Severe active-write imbalance
2. High ingest pipeline CPU cost
3. Disk imbalance in warm tier
4. 17 nodes experiencing write queue pressure
5. ILM rollover backlog

For each:

Severity
Evidence
Affected nodes
Likely cause
Confidence
Recommended action
Expected impact
Risk

---

49. Recommendation Engine

Recommendations must be evidence-based.

Examples:

If active write primary imbalance is high

Recommend investigating:

- allocation settings
- write-load balancing
- shard allocation decisions
- index routing
- shard distribution

Do NOT immediately recommend rerouting thousands of shards.

If ingest pipeline CPU is high

Recommend:

- identify expensive grok
- identify expensive scripts
- inspect nested pipelines
- optimize patterns
- replace deterministic grok with dissect where appropriate
- consider dedicated ingest tier if justified

If disk imbalance is high

Recommend:

- investigate allocation deciders
- disk watermarks
- awareness
- allocation filters
- shard-size concentration
- tier configuration

If rollover creates tiny shards

Recommend:

- revisit age rollover
- size-based rollover
- target shard size
- shard count per index

If recovery pressure is high

Recommend:

- investigate why recoveries occur
- avoid blindly increasing recovery concurrency
- assess disk/network/CPU capacity

If circuit breakers are tripping

Recommend finding the memory consumer rather than simply increasing limits.

---

50. Shard Sizing Calculator

Build an explicit shard-sizing tool.

Inputs:

- expected daily volume
- average indexed bytes/doc
- peak docs/sec
- target primary shard size
- validated docs/sec/shard
- validated MB/sec/shard
- replicas

Calculate:

shards_by_size
shards_by_docs_rate
shards_by_bytes_rate
recommended_primary_shards

Use:

recommended =
max(
  shards_by_size,
  shards_by_validated_docs_rate,
  shards_by_validated_bytes_rate
)

Do NOT use a hardcoded 800 docs/sec rule.

Allow users to enter a benchmark-derived capacity.

Clearly explain:

«Elasticsearch does not provide a universal documents/sec/shard limit.»

---

51. Bulk Request Analysis

Create a section called:

Bulk Request Visibility

Clearly explain:

«Elasticsearch does not necessarily expose an authoritative bulk-request/sec metric through the standard node statistics APIs.»

Show what can be measured:

- HTTP request rate
- indexing operation rate
- docs/sec
- thread-pool completion/rejection
- transport activity

Then show what requires external telemetry:

- exact bulk request/sec
- documents/bulk
- average bulk payload size
- client-specific bulk rate

Recommend collecting these from:

- Beats metrics
- application metrics
- Logstash metrics
- load balancer
- APM
- client instrumentation

Do not fabricate bulk rate.

---

52. UI Dashboard

Build dashboard sections:

Overview

Cards:

- cluster health
- nodes
- shards
- active write primaries
- indexing docs/sec
- primary docs/sec
- write queue
- rejected writes
- max disk
- max heap
- max CPU

Node Heatmap

Rows:

node

Columns:

CPU
Heap
Disk
Total shards
Primary shards
Write primaries
Primary docs/sec
Write queue
Ingest processing
Merge

Use conditional formatting.

Hot Node Analysis

Top hotspots.

Shard Allocation

Interactive node map.

Active Write Shards

Show:

index
alias
shard
node
primary/replica
docs
store
estimated docs/sec

Ingest Pipeline Analysis

Show:

pipeline
executions/sec
avg processor time
total time
grok
script
nested pipeline
failures

Disk Distribution

Charts:

- hot tier disk
- warm tier disk
- shard size distribution

ILM/Rollover

Show:

- current write indices
- age
- size
- rollover condition
- current phase
- stuck indices

Thread Pools

Show:

- active
- queue
- completed
- rejected
- rate of change

Circuit Breakers

Show:

- estimated
- limit
- trips

Recoveries

Show:

- relocating shards
- recovery bytes
- recovery duration

Recommendations

Sort by:

Critical
High
Medium
Low

---

53. Excel Report

Generate XLSX with separate worksheets:

Executive Summary
Cluster Health
Nodes
Hot Nodes
Warm Nodes
Master Nodes
Coordinating Nodes
Shard Allocation
Active Write Shards
Write Load
Indexing
Thread Pools
Ingest Pipelines
Ingest Processors
Disk Analysis
JVM
GC
Circuit Breakers
Merges
Refresh
Translog
Segments
Recovery
ILM
Rollover
Aliases
Templates
Mappings
Allocation Settings
Pending Tasks
Tasks
Recommendations
Raw API Index

Use filters, frozen headers, conditional formatting and charts.

---

54. HTML Report

Generate a standalone HTML report that can be opened without the application.

Include:

- executive summary
- severity badges
- charts
- node heatmaps
- sortable tables
- detailed evidence
- recommendations
- methodology
- limitations
- timestamp
- Elasticsearch version

Allow expanding technical details.

---

55. Report Methodology

Every recommendation must show:

Observation
Evidence
Analysis
Conclusion
Confidence
Recommended action
Risk

Example:

Observation:
hot-17 has 3.2x the average active primary write load.

Evidence:
Primary docs/sec = 31,000
Cluster hot-node average = 9,700
Write queue = 184
CPU = 91%

Conclusion:
Likely write-load hotspot.

Confidence:
HIGH

Recommended action:
Investigate shard allocation decisions and write-load balancing before manually rerouting shards.

---

56. Important Safety Around Recommendations

The analyzer must NOT automatically recommend:

- increasing heap
- increasing circuit breakers
- disabling allocation
- disabling replicas
- increasing recovery concurrency
- changing shard count
- increasing refresh interval
- changing rollover
- rerouting shards

unless the evidence supports it.

Every recommendation must include:

Why
Expected benefit
Potential downside
Risk
What to monitor after change

---

57. Important Analytical Rules

The analyzer must follow these rules:

Rule 1

Do not equate shard count with workload.

Rule 2

Do not equate documents/sec with shard capacity.

Rule 3

Use bytes/sec as well as docs/sec.

Rule 4

Separate primary indexing from replica work.

Rule 5

Separate ingest processing from indexing.

Rule 6

Separate disk balance from write-load balance.

Rule 7

Do not assume a node with 70 write primaries is overloaded.

Rule 8

Compare against cluster/tier averages and actual workload.

Rule 9

Use time-series deltas, not only instantaneous counters.

Rule 10

Never claim exact bulk request rate unless the data supports it.

Rule 11

Do not blindly modify allocation settings.

Rule 12

Do not treat Elasticsearch defaults as universal best practices.

---

58. Final Diagnosis Engine

At the end, produce a diagnosis such as:

PRIMARY ROOT CAUSE
------------------
Active write-load imbalance

CONFIDENCE
----------
High

EVIDENCE
--------
Hot-17:
Primary docs/sec: 31,200
Hot-tier average: 9,400
Write queue: 184
CPU: 92%
Active write primaries: 68
Average: 22

Ingest:
Processor time: normal

Disk:
Hot-17: 54%
Hot-tier average: 51%

CONCLUSION
----------
The node is receiving disproportionately high active primary
indexing workload rather than simply having more total shards.

RECOMMENDED FIRST ACTION
------------------------
Investigate shard allocation/write-load balancing.

DO NOT FIRST
------------
Increase shard count.
Add nodes.
Increase circuit breakers.
Increase write queue.

The diagnosis engine must be capable of producing a different conclusion if the evidence points to ingest pipeline CPU instead.

---

59. Confidence Scoring

Every major diagnosis must have:

HIGH
MEDIUM
LOW
INSUFFICIENT DATA

Explain the evidence supporting the confidence.

---

60. Before/After Comparison

Allow two analysis bundles to be loaded:

Before
After

Compare:

- indexing rate
- write queue
- rejected writes
- CPU
- heap
- ingest processing
- shard distribution
- disk distribution
- rollover backlog
- recovery rate

Show whether a remediation actually improved the cluster.

---

61. CLI JSON Output

Provide:

es-analyzer analyze --output report.json

The JSON must contain:

{
  "cluster": {},
  "health": {},
  "nodes": {},
  "shards": {},
  "write_load": {},
  "ingest": {},
  "disk": {},
  "thread_pools": {},
  "recovery": {},
  "ilm": {},
  "recommendations": []
}

Make the JSON stable and documented so it can later be consumed by other AI tools.

---

62. AI-Friendly Summary

Generate a concise Markdown file:

AI_SUMMARY.md

It must contain:

1. Cluster architecture
2. Cluster health
3. Node distribution
4. Active write indices
5. Active write primary distribution
6. Primary indexing rate
7. Replica indexing rate
8. Ingest pipeline pressure
9. CPU pressure
10. Heap pressure
11. Disk imbalance
12. Thread-pool pressure
13. Circuit breakers
14. Recoveries
15. ILM/rollover
16. Root-cause candidates
17. Evidence
18. Recommended actions
19. Confidence
20. Raw API references

This file should be optimized so another AI assistant can read it and diagnose the cluster.

---

63. Recommended Project Structure

Use a clean architecture similar to:

es-cluster-analyzer/
│
├── cmd/
│   └── analyzer/
│
├── internal/
│   ├── elasticsearch/
│   │   ├── client.go
│   │   ├── cluster.go
│   │   ├── nodes.go
│   │   ├── shards.go
│   │   ├── indices.go
│   │   ├── ingest.go
│   │   ├── ilm.go
│   │   ├── templates.go
│   │   ├── allocation.go
│   │   ├── recovery.go
│   │   └── tasks.go
│   │
│   ├── collector/
│   ├── sampler/
│   ├── analyzer/
│   │   ├── cluster.go
│   │   ├── write_load.go
│   │   ├── shard_balance.go
│   │   ├── disk.go
│   │   ├── ingest.go
│   │   ├── threadpool.go
│   │   ├── jvm.go
│   │   ├── recovery.go
│   │   ├── ilm.go
│   │   └── recommendations.go
│   │
│   ├── model/
│   ├── report/
│   └── security/
│
├── web/
│   ├── frontend/
│   └── assets/
│
├── reports/
├── tests/
├── examples/
├── docs/
│
├── Dockerfile
├── docker-compose.yml
├── go.mod
└── README.md

---

64. Testing Requirements

Create unit tests for:

- shard distribution calculations
- percentile calculations
- write-load scoring
- disk imbalance
- rollover analysis
- pipeline dependency graph
- pipeline complexity
- rate calculations
- counter deltas
- hotspot detection
- recommendation engine

Create integration tests using a mocked Elasticsearch API.

Create sample datasets representing:

1. Healthy cluster
2. Write hotspot
3. Ingest CPU hotspot
4. Disk imbalance
5. Recovery pressure
6. Circuit breaker pressure
7. Rollover backlog
8. Combined failure

---

65. Performance Requirements

The analyzer itself must be efficient enough for clusters containing:

- 100 nodes
- 50,000 indices
- 100,000 shards
- 500 ingest pipelines

Do not use O(nodes × shards × indices) algorithms unnecessarily.

Use maps/indexes for joins.

For example:

shard → node
index → alias
index → ILM policy
node → stats

Avoid repeated API calls for the same object.

---

66. UI UX

The UI should immediately answer:

«"What is wrong with my cluster?"»

The first screen should show:

Cluster Health

Overall:
WARNING

Primary suspected issue:
WRITE LOAD IMBALANCE

Confidence:
HIGH

Affected nodes:
7

Secondary issue:
INGEST PIPELINE CPU

Confidence:
MEDIUM

Third issue:
DISK IMBALANCE

Confidence:
MEDIUM

Then allow drilling down.

---

67. Do Not Hide Uncertainty

If the available Elasticsearch APIs cannot prove something, say:

Unable to determine directly from Elasticsearch APIs.
Additional telemetry required:
- client bulk metrics
- load balancer metrics
- application metrics

Never invent a value.

---

68. The Most Important Output For This Specific Cluster

The analyzer MUST generate this table:

Node| Active Write Primaries| Primary Docs/sec| Primary MB/sec| Total Docs/sec| Write Queue| Write Rejected| CPU| Heap| Disk| Ingest Time| Merge Time| Relocating

Sort it by:

1. Write hotspot score
2. Primary docs/sec
3. Write queue

This table should immediately reveal whether the problem is actually shard distribution.

---

69. Second Most Important Table

Generate:

Index| Alias| Primary Shards| Current Write| Docs/sec| MB/sec| Avg Shard Size| Age| ILM Phase| Pipeline| Node Distribution

This lets us identify whether a particular high-volume index is causing a hotspot.

---

70. Third Most Important Table

Generate:

Pipeline| Executions/sec| Avg ms/doc| Total processing time| Grok| Script| Nested Pipeline| Failures| Complexity

This should reveal whether ingest processing is contributing to the write queue.

---

71. Fourth Most Important Table

Generate:

Node| Total Shards| Primary| Replica| Write Primary| Total Store| Primary Store| Disk %| Tier

This separates shard-count imbalance from storage imbalance.

---

72. Final Deliverables

The generated project must produce:

cluster-analysis.html
cluster-analysis.xlsx
cluster-analysis.json
AI_SUMMARY.md
raw-data.zip

Optionally:

cluster-analysis.pdf

The UI should also allow downloading all reports.

---

73. Final README

The README must explain:

- architecture
- installation
- authentication
- required Elasticsearch permissions
- APIs used
- data collected
- analysis methodology
- limitations
- security
- how to run
- how to interpret results
- sample screenshots
- sample reports

Provide an example read-only Elasticsearch role sufficient for the analyzer.

Use least privilege.

Do not request "superuser".

---

74. Important Elasticsearch 8.14 Compatibility Requirement

The implementation must be tested against Elasticsearch 8.14.x.

Do not blindly copy APIs/settings from Elasticsearch 7.x or 9.x.

Where an API or metric differs by version:

- detect Elasticsearch version
- use the appropriate API
- mark unsupported metrics
- continue the rest of the analysis

The tool must degrade gracefully.

---

75. Development Strategy

Build the project in phases:

Phase 1:
Connection + cluster discovery

Phase 2:
Node/shard/index collection

Phase 3:
Sampling engine

Phase 4:
Write-load analysis

Phase 5:
Ingest pipeline analysis

Phase 6:
Allocation/disk analysis

Phase 7:
ILM/rollover analysis

Phase 8:
Recommendation engine

Phase 9:
HTML report

Phase 10:
Excel report

Phase 11:
React UI

Phase 12:
Testing + Docker

Do not create a superficial mockup.

Implement the real Elasticsearch API integration and real calculations.

---

Most Important Goal

The tool must answer this question with evidence:

«"Why is this Elasticsearch cluster developing write queues and node hotspots, and what should we change first?"»

It must distinguish among:

A. Too many/few primary shards
B. Uneven active write-primary allocation
C. Uneven actual indexing workload
D. Ingest pipeline CPU pressure
E. Coordinating-node bottleneck
F. Bulk/request pressure
G. Merge pressure
H. Disk I/O pressure
I. JVM/GC pressure
J. Circuit breaker pressure
K. Recovery/relocation pressure
L. ILM/rollover backlog
M. Disk allocation imbalance
N. Allocation configuration
O. Combination of multiple factors

The final output should never simply say:

«"Add more shards."»

or:

«"Add more nodes."»

Instead it must provide:

Problem
→ Evidence
→ Root cause
→ Confidence
→ Recommended change
→ Risk
→ Expected effect
→ What to monitor after the change

The analyzer should be useful both to a human Elasticsearch engineer and to an AI assistant that receives the generated "AI_SUMMARY.md" and JSON report for further analysis.
