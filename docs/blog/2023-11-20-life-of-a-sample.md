---
title: Life of a sample in Thanos, and how to configure it
date: "2023-11-20"
author: Thibault Mangé (https://github.com/thibaultmg)
---

## Life of a sample in thanos, and how to configure it

### Introduction

Thanos is a sophisticated distributed system with a broad range of capabilities, and with that comes a certain level of configuration intricacies. In this article, we'll take a deep dive into the lifecycle of a sample within Thanos, tracking its journey from initial ingestion to final retrieval. Our focus will be to explain Thanos' critical internal mechanisms and pinpoint the essential configurations for each component, guiding you to achieve the operational results you're aiming for. We'll be covering the following thanos components:

* **Receive**: Ingests samples from Prometheus servers and prepares them for object storage.
* **Compactor**: Merges and deduplicates blocks in object storage.
* **Store**: Exposes blocks in object storage for querying.
* **Query**: Retrieves data from stores and processes queries.
* **Query Frontend**: Distributes queries to query instances.

The objective of this article is to make Thanos more accessible to new users, helping to alleviate any initial apprehensions. We'll also assume that the working environment is Kubernetes. Given the extensive ground to cover, we'll aim to maintain conciseness throughout our exploration.

Before diving deeper, please check annexes to clarify some essential terminology. If you're already familiar with these concepts, feel free to skip ahead.

### The sample origin: do you have close integration capabilities?

The sample usually originates from a Prometheus instance that is scraping targets in a cluster. There are two possible scenarios:

* The **Prometheus instances are under your control and you can access it from your Thanos deployment**. In this case you can use the Thanos sidecar that you will attach the the pod running the prometheus server. The Thanos sidecar will read directly the samples from the Prometheus server using the read API. Then the sidecar will behave similarly to the other scenario without the routing and ingestion parts. Thus we will not go further into this use case.
* The **Prometheus servers are running in clusters that you do not control**. You'll often hear air gapped clusters. In this case, you cannot attach a sidecar to the Prometheus server. The samples will trvel to your Thanos system using the remote write protocol. This is the scenario we will focus on.

The following schema illustrates the two scenarios:

<img src="img/life-of-a-sample/close-integration.png" alt="Close integration vs external client" width="400"/>

Comparing the two deployment modes, the Sidecar Mode is generally preferable due to its simpler configuration and fewer moving parts. However, if this isn't possible, opt for the **Receive Mode**. Bear in mind, this mode requires careful configuration to ensure high availability, scalability, and durability. It adds another layer of indirection and comes with the overhead of operating the additional component.

<!-- Add schema of both topologies, with side car in grey -->

### Sending samples to Thanos

#### The remote write protocol

Say hi to our first Thanos component, the **Receive** or **Receiver**, the entry point to the system. It was introduced with this [proposal](https://thanos.io/tip/proposals-done/201812-thanos-remote-receive.md/). This component facilitates the ingestion of metrics from multiple clients, eliminating the need for close integration with the clients' Prometheus deployments.

Thanos Receive is a server that exposes a remote-write endpoint (see [Prometheus remote-write](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#remote_write)) that Prometheus servers can use to transmit metrics. The only prerequisite on the client side is to configure the remote write endpoint on each Prometheus server, a feature natively supported by Prometheus. 

On the Receive, the remote write endpoint is configured with the `--remote-write.address` flag. You can also configure TLS options using other `--remote-write.*` flags. 

The remote-write protocol is based on HTTP POST requests. The payload consists of a protobuf message containing a list of time-series samples and labels. Generally, a payload contains at most one sample per time series in a given payload and spans numerous time series. Metrics are typically scraped every 15 seconds, with a maximum remote write delay of 5 seconds to minimize latency, from scraping to query availability on the receiver. 

#### Tuning the remote write protocol

The Prometheus remote-write configuration offers various parameters to tailor connection specifications, parallelism, and payload properties (compression, batch size, etc.). While these may seem like implementation details for Prometheus, understanding them is essential for optimizing ingestion, as they form a sensitive part of the system.

Implementation-wise for Prometheus, the key idea is to read directly from the TSDB WAL (Write Ahead Log), a simple mechanism commonly used by databases to ensure data durability. If you wish to delve deeper into the TSDB WAL, check out this [great article](https://ganeshvernekar.com/blog/prometheus-tsdb-wal-and-checkpoint). Once samples are extracted from the WAL, they are aggregated into parallel queues (shards) as remote-write payloads. When a queue reaches its limit or a maximum timeout is attained, the remote-write client stops reading the WAL and dispatches the data. The cycle continues. The parallelism is defined by the number of shards, their number is dynamically optimized. More insights on Prometheus' remote write tuning can be found [here](https://prometheus.io/docs/practices/remote_write/).

<img src="img/life-of-a-sample/remote-write.png" alt="Remote write" width="700"/>

Key Points to Consider:

* **Send deadline setting**: `batch_send_deadline` should be set to arround 5s to minimize latency. This timeframe strikes a balance between minimizing latency and avoiding excessive request frequency that could burden the receiver. While a 5-second delay might seem substantial in critical alert scenarios, it's generally acceptable considering the typical resolution time for most issues.
* **Backoff settings**: The `min_backoff` should ideally be no less than 250 milliseconds, and the `max_backoff` should be at least 10 seconds. These settings help prevent receiver overload, particularly in situations like system restarts, by controlling the rate of data sending.

#### Protecting the receiver from overuse

In scenarios where you have limited control over client configurations, it becomes essential to shield the Receive component from potential misuse or overload. The Receive component includes several configuration options designed for this purpose, comprehensively detailed in the [official documentation](https://thanos.io/tip/components/receive.md/#limits--gates-experimental). Below is a diagram illustrating the impact of these configuration settings:

<img src="img/life-of-a-sample/receive-limits.png" alt="Receive limits" width="900"/>

When implementing a topology with separate router and ingestor roles, these limits should be enforced at the router level.

Key points to consider:

* **Series and samples limits**: Typically, with a standard target scrape interval of 15 seconds and a maximum remote write delay of 5 seconds, the `series_limit` and `samples_limit` tend to be functionally equivalent. However, in scenarios where the remote writer is recovering from downtime, the `samples_limit` may become more restrictive, as the payload might include multiple samples for the same series.
* **Handling request rimits**: If a request exceeds these limits, the system responds with a 413 (Entity Too Large) HTTP error. Presently, Prometheus does not support splitting requests in response to this error, which leads to data loss.
* **Active series limiting**: The limitation on active series persists as long as the count remains above the set threshold in the receivers' TSDB. The number of active series decreases when data is deleted in accordance with the `--tsdb.retention` setting. Consequently, longer retention periods can prolong the duration of incoming data rejection, potentially causing data loss if it exceeds the client’s WAL retention time. Requests reching this limit are rejected with 429 (Too Many Requests) HTTP code, triggering retries.

Considering these aspects, it is importamt to carefully monitor and adjust these limits. While they serve as necessary safeguards, overly restrictive settings can inadvertently lead to data loss. 

### Receiving samples with High Availability and Durability 

#### The need for multiple Receive instances

Relying on a single instance of Thanos Receive is not sufficient for two main reasons:

* Scalability: As your metrics grow, so does the need for scaling your infrastructure.
* Reliability: If a single Receive instance fails, it disrupts metric collection, affecting rule evaluation and alerting. Furthermore, during downtime, Prometheus servers will buffer data in their Write-Ahead Log (WAL). If the outage exceeds the WAL's retention duration (default 2 hours), this can lead to data loss.

#### The Hashring Mechanism

To achieve high availability, it is necessary to deploy multiple receive replicas. However, it's not just about having more instances; it's crucial to maintain consistency in sample ingestion. In other words, samples from a given time series must always be ingested be the same Receive instance. This consistency is key for constructing chunks and blocks, as we'll discuss later. 

To that effect, you guessed it, the receive uses a hashring! With the hashring, every Receive participant knows and agrees on who must ingest what sample. When clients send data, they connect to any Receive instance, which then routes the data to the correct instances based on the hashring. This is why the Receive component is also known as the **IngestorRouter**.

<img src="img/life-of-a-sample/ingestor-router.png" alt="IngestorRouter" width="350"/>

Receive instances use a gossip protocol to maintain a consistent view of the hashring, requiring inter-instance communication via a configured HTTP server (`--http-address` flag).

There are two possible hashrings:

* **hashmod**: This algorithm distributes time series by hashing labels. It's effective in evenly distributing load but requires data flushing to object storage during scaling operations, potentially causing downtime. This is especially critical if you are running big receive nodes. The most data they have, the longer it will take to flush it to the object store. 
* **ketama** A more recent addition, employing consistent hashing. It means that during scaling operations, most of the time series will remain attached to the same nodes. It allows for less data movement during scaling, reducing the impact on nodes. However, it's less efficient in load distribution compared to hashmod.

We recommend using the Ketama hashring (--receive.hashrings-algorithm flag) for its operational benefits. This is configured with the `--receive.hashrings-algorithm` flag.

When the hashring configuration changes, Receive instances need to flush data to object storage EVEN WITH KETAMA??. During that time, the Receive replies with 503 to the clients which is interpreted as a temporary failure and remote-writes are retried. At that moment, your receive will have to catch up and receive a lot of data. This is why we recommend using the `--receive.limits-config` flag to limit the amount of data that can be received. This will prevent the receive from being overwhelmed by the catch up. 

EXPLAIN HOW TO SETUP NODES LIST, TALK ABOUT DNS DISCOVERY

#### Ensuring Samples Durability

For clients requiring high data durability, the `--receive.replication-factor` flag ensures data duplication across multiple receivers. When set to n, it will only reply a succesful processing to the client once it has duplicated the data to `n-1` other receivers. Additionally, an external replicas label can be added to each Receive instance (`--label` flag) to mark replicated data. This setup increases data resilience but also expands the data footprint. 
HOW DOES IT PLAYS WITH `--receive.replica-header`?

#### Improving Scalability and Reliability

To address the challenges in scalability and reliability, a new deployment topology was [proposed](https://thanos.io/tip/proposals-accepted/202012-receive-split.md/), separating the **router** and **ingestor** roles. Routers now manage the hashring, directing data to appropriate ingestors. This approach simplifies scaling and reduces downtime. DEVELOP.

<img src="img/life-of-a-sample/router-and-ingestor.png" alt="IngestorRouter" width="350"/>

### Preparing samples for object storage: building chunks and blocks

#### Using object storage

A key feature of Thanos is its ability to leverage economical object storage solutions like AWS S3 for long-term data retention. This contrasts with Prometheus's typical approach of storing data locally for shorter periods.

The Receive component is responsible for preparing data for object storage. Thanos adopts the TSDB (Time Series Database) data model, with some modifications, for its object storage. This involves aggregating samples over time to construct TSDB Blocks.

These blocks are built by aggregating data over two-hour periods. Once a block is ready, it's sent to the object storage, which is configured using the `--objstore.config` flag. This configuration is uniform across all components requiring object storage access.

On restarts, the Receive component ensures data preservation by immediately flushing existing data to object storage, even if it doesn't constitute a full two-hour block. These partial blocks are less efficient but are later optimized by the compactor.

#### Exposing local data for queries

During the block-building phase, the data isn't accessible to the store gateway. Hence, the Receive component also serves as a data store, making the local data available to queriers through the store API. This is a common gRPC API used across all Thanos components for time series data access, set with the `--grpc-address` flag. The receive will serve all data is has. The more data it serves, the more resources it will use for this duty in addition of ingesting client's data. WHY WOULD WE SERVE MORE THAN 2H OF DATA? The amount of data the Receive component serves can be managed through two parameters:

* `--tsdb.retention`: Sets the local storage retention duration. The minimum is 2 hours, aligning with block construction periods.
* `--store.limits.request-samples` and `--store.limits.request-series`: These limit the volume of data that can be queried. Exceeding these limits will result in the query being TRUNCATED OR DENIED??, ensuring system stability. WHAT HAPPENS TO thE ORIGINAL QUERY?

#### Handling Out-of-Order Timestamps

Issues can arise when clients send data with out-of-order timestamps, often due to unsynchronized clocks or more complex client setups with several prometheusis handling the same target. Explain `--receive.max-block-duration`, WHAT TRADEOFFS?. SCHEMA 


### Maintaining data: compaction, downsampling, and retention

#### The need for compaction

HOW DOES THE COMPACTOR DETECTS NEW BLOCKS? HOW LONG BEFORE A NEW BLOCK IS PROCESSED?

The Receive component implements many strategies to ingest samples reliably. However this can result in unoptimized data in object storage. This is due to:

* Inefficient partial blocks sent to object storage on shutdowns
* Duplicated data when replication is set.  Several Receive instances will send the same data to object storage.
* Incompete blocks (invalid blocks) sent to object storage when the receive fails in the middle of an upload.

The following schema illustrates the impact on data expansion (6 times) in object storage when samples from a given target are ingested from a high availabity prometheus setup (with 2 instances) and repliacation (factor 3) is set on the receive:

<img src="img/life-of-a-sample/data-expansion.png" alt="Data expansion" width="400"/>

The Compactor component is responsible for maintaining and optimizing data in object storage. It's a long-running process that can be configured to wait for new blocks with the `--wait` flag. It also needs access to the object storage with the `--objstore.config` flag.

#### Compaction modes

Compaction consist in merging blocks that have overlapping or adjacent time ranges. This is called **horizontal compaction** Using the Metadata file that contains the minimum and maximum timestamp of samples in the block, the compactor can determine if two blocks overlap. If they do, they are merged into a new block. This new block will have its compaction level index increased by one. So from two adjacent blocks of 2 hours, we will get a new block of 4 hours. 

During this compaction, the compactor will also deduplicate samples. This is called **vertical compaction**. The compactor provides two deduplication modes:

* `one-to-one`: This is the default mode. It will deduplicate samples that have the same timestamp and the same value. But different replica label values. The replica label is configured by the `--deduplication.replica-label` flag REALLY?. Usually set to `replica`, make sure it is set up as external label on the receivers. The benefit of this mode is that it is straightforward and will remove replicated data from the receive. However, it is not able to remove data replicated by high availability prometheus setups. Because, these samples will rarely be scraped at exactly the same timestamps.
* `penalty`: This a more complex deduplication algorithm that is able to deduplicate data coming from high availability prometheus setups. It can be set with the `--deduplication.func` flag and requires also setting the `--deduplication.replica-label` flag that identifies the label that contains the replica label. Usually `prometheus_replica`. Here is a schema illustrating how Prometheus replicas generate samples with different timestamps that cannot be deduplicated with the `one-to-one` mode:

<img src="img/life-of-a-sample/ha-prometheus-duplicates.png" alt="High availability prometheus duplication" width="500"/>

Getting back to our example illustrating the data duplication happening in the object storage, here is how each compaction process will impact the data:

<img src="img/life-of-a-sample/compactor-compaction.png" alt="Compactor compaction" width="700"/>


You want to deduplicate data as much as possible because it will lower your object storage cost and improve query performance. But using the penalty presents some limitations. Have a look at (https://thanos.io/tip/components/compact.md/#vertical-compaction-risks)

#### Downsampling and retention

The compactor is also optimizing data read for long range queries. If you are querying data for several months, you don't need the typical 15s raw resolution. Processing such a query will be very inefficient as it will retrieve a lot of unnecessary data. To enable performant long range queries, the compactor can downsample data. It supports two downsampling levels: 5m and 1h. These are the resolution of the downsamples series. They will tipically come on top of the raw data, so that you can have both raw and downsampled data. We'll see later how to configure the query to use the downsampled data.

Finally you can configure how long you want to retain your data on object storage. One flag `--retention.resolution-*` for each resolution is available. We recommand having the same value for each. 

#### The compactor UI and the block streams

All blocks covering the same time range are not compacted together. Instead, the Compactor organizes them into distinct [compaction groups or block streams](https://thanos.io/tip/components/compact.md/#compaction-groups--block-streams). The key here is to leverage external labels to group data originating from the same source. This strategic grouping is particularly effective for compacting indexes, as blocks from the same source tend to have nearly identical labels.

The Compactor's functionality and the progress of its operations can be monitored through the Block Viewer UI. This web-based interface is accessible if the Compactor is configured with the `--http-address` flag. Additional UI settings are controlled via `--web.*` and `--block-viewer.*` flags. The Compactor UI provides a visual representation of the compaction process, showing how blocks are grouped and compacted over time. Here’s a glimpse of what the UI looks like:

<img src="img/life-of-a-sample/compactor-ui.png" alt="Receive and Store data overlap" width="800"/>

Occasionally, some blocks may display an artificially low level in the UI, appearing lower in the stream compared to adjacent blocks. This scenario often occurs in situations like rolling receiver upgrades, where receivers restart sequentially, leading to the creation and upload of partial blocks to the object store. The Compactor then vertically compacts these blocks as they arrive, resulting in a temporary increase in compaction levels. When these blocks are horizontally compected with adjactent blocks, they will also be displayed lower down the stream.

By default, the Compactor’s strategy involves compacting 2-hour blocks into 8-hour blocks once they are available, then progressing to 2-day blocks, and so on, following a structured compaction timeline.

### Exposing buckets data for queries: the store gateway and the store API

#### Exposing data for queries

HOW DOES THE STORE KNOWS NEW BLOCKS ARE AVAILABLE? WHAT DELAY? SYNC MECHANISMS WITH THE RECEIVE? CAN THE CACHE BECOME INVALID?

The Store Gateway is acting as a facade for the object storage, making bucket data accessible via the Thanos Store API, a feature first introduced with the Receive component. The store exposes the store API with the `--grpc-address` flag.

The Store Gateway requires access to the object storage bucket to retrieve data, set with the `--objstore.config` flag. Additionally, it utilizes caches to efficiently store indexes and chunks of data.

#### Retrieving samples from the object storage

Consider the simple following query done on the Querier:

```promql
# Between now and 2 days ago, compute the rate of http requests per second, filtered by method and status
rate(http_requests_total{method="GET", status="200"}[5m])
```

This PromQL query will be parsed by the Querier that will emit a Thanos [Store API](https://github.com/thanos-io/thanos/blob/main/pkg/store/storepb/rpc.proto) request to the Store Gateway with the following parameters:

```proto
SeriesRequest request = {
  min_time: [Timestamp 2 days ago],
  max_time: [Current Timestamp],
  max_resolution_window: 1h,  // the minimum time range between two samples, relates to the downsampling levels
  matchers: [
    { name: "__name__", value: "http_requests_total", type: EQUAL },
    { name: "method", value: "GET", type: EQUAL },
    { name: "status", value: "200", type: EQUAL }
  ]
}
```

The Store Gateway processes this request in several steps:

* **Metadata processing**: The Store Gateway first examines the block [metadata](https://thanos.io/tip/thanos/storage.md/#metadata-file-metajson) to determine the relevance of each block to the query. It evaluates the time range (`minTime` and `maxTime`) and external labels (`thanos.labels`). Blocks are deemed relevant if their timestamps overlap with the query's time range and if their resolution (`thanos.downsample.resolution`) matches the query's maximum allowed resolution.
* **Index processing**: Next, the Gateway retrieves the [indexes](https://thanos.io/tip/thanos/storage.md/#index-format-index) of candidate blocks. This involves:
  * Fetching postings lists for each label specified in the query. These are inverted indexes where each label and value has an associated sorted list of all the corresponding time series. Example:
    * "__name__=http_requests_total": [1, 2, 3],
    * "method=GET": [1, 2, 6],
    * "status=200": [1, 2, 5]
  * Intersecting these postings lists to pinpoint series that match all query labels. In our example these are series 1 and 2.
  * Retrieving the Series section from the index for these series, which includes chunk information and their time ranges. Example:
    * Series 1: [Chunk 1: mint=t0, maxt=t1, fileRef=0001, offset=0], ...
  * Determining the relevant chunks based on their time range intersection with the query.
* **Chunks retrieval**: The Gateway then fetches the appropriate chunks, either from the object storage directly or from a chunk cache. When retrieving from the object store, the Gateway leverages its API to read only the needed bytes (i.e. using S3 range requests), bypassing the need to download entire chunk files. Then, the gateway streams selected chunks to the Querier.

Understanding the retrieval algorithm highlights the critical role of an external [index cache](https://thanos.io/tip/components/store.md/#index-cache) in the Store Gateway's operation. This is configured using the `--index-cache.config` flag. Indexes contain all labels and values of the block which can result in big sizes. When the cache is full, Last-In-First-Out (LIFO) eviction is applied. In scenarios where no external cache is configured, a portion of the memory will be utilized as a cache, managed via the `--index-cache.size` flag.

Moreover, the direct retrieval of chunks from object storage can be suboptimal, and result in excessive costs, especially with a high volume of queries. To mitigate this, employing a [caching bucket](https://thanos.io/tip/components/store.md/#caching-bucket) can significantly reduce the number of queries to the object storage. This chunk caching strategy is particularly effective when data access patterns are predominantly focused on recent data. By caching these frequently accessed chunks, query performance is enhanced, and the load on object storage is reduced.

#### Enhancing Store Performance through Sharding

The performance of Thanos Store components can be notably improved by managing concurrency and implementing sharding strategies.

Adjusting the level of concurrency can have a significant impact on performance. This is managed through the `--store.grpc.series-max-concurrency` flag for the allowed concurrent series requests on the store API. Other lower level concurrency settings are also available.

After having optimized the store processing, you can distribute the queries load using sharding strategies enabled by the relabel configuration. This approach involves processing different blocks on different Store instances. Here’s an example of how to set up sharding using the `--selector.relabel-config` flag:

```yaml
- action: hashmod
  source_labels:
    - __block_id
  target_label: shard
  modulus: 2 # number of store replicas
- action: keep
  source_labels:
    - shard
  regex: 0 # shard number
```

In this configuration, the `hashmod` action is used to distribute blocks across multiple Store instances (replicas) based on the `__block_id` label. The `modulus` should match the number of Store replicas you have. Each replica will then keep only the blocks that match its shard number, as defined by the `regex` in the `keep` action. This setup allows for a more balanced distribution of query processing, enhancing overall system performance. 

However, this sharding approach isn't a universal solution. One potential issue comes from the fact that series are grouped within a block based on their external labels, typically originating from the same data source. In such cases, if the load is predominantly from one source, sharding may be less effective. This is especially true for blocks that have undergone horizontal compaction and cover extensive time ranges, potentially resulting in an uneven query load on a single Store instance. OTHER SOLUTION? ADD RANDOM SET OF EXTERNAL LABELS AT THE SERIES LEVEL TO INCREASE THE NUMBER OF STREAMS?

### Querying data from the Thanos stores: the querier

#### Configuring Samples Access in the Querier

The Querier component in Thanos is entry point for samples retrieval. It is responsible for processing PromQL queries. This process involves:

* Parsing the query.
* Fetching data from Thanos stores.
* Processing and returning the query result.

Thanos stores, comprising the Receive component for recent data and the Store Gateway for older records, are configured using  the `--endpoint*` flags. As the Querier is unaware of which store owns what data, it distributes requests across all configured stores. 

<!-- schema -->

To prevent data overlap between the Receive component (handling data up to its TSDB retention period) and the Store Gateway (handling blocks exported to the object store), the `--max-time` flag is used in the Store Gateway. This flag ensures the Store Gateway does not serve data too recent and already covered by the Receive. However, a slight overlap is necessary to prevent data gaps:

<img src="img/life-of-a-sample/data-overlap.png" alt="Receive and Store data overlap" width="400"/>

#### Deduplication and Partial Response Handling

The Querier receives data that may be duplicated. This can be the case when:

* The Receive component is configured with replication and recent data is requested. The query will receive the same data from multiple Receive instances as it fans out its requests to all Thanos stores.
* Some data is served by both the receive and the store gateway when the query time range overlaps with common time ranges served by both components.
* Some data come from high availability prometheus setups.

As a result, the querier may receive duplicated data from the receivers in case of duplication or high availability setups upstream. This is why the querier will deduplicate data before processing the query. Deduplicating data from high availability prometheusis requires setting the flag `--query.replica-label` to the label identifying replicas (commonly prometheus_replica). It uses the same algorithm as the compactor.

In a distributed environment, not all stores may successfully retrieve necessary samples. If partial responses occur, the behavior is controlled by `--query.partial-response`. By default, queries are aborted on partial responses, a sensible default for rule evaluations. However, for ad-hoc queries where availability is prioritized over consistency, setting --query.partial-response=warn allows the query to proceed with warnings visible in the Thanos Query UI, exposed by the Querier. It is the same as the Prometheus UI, adapted to Thanos.

WHY ARE THERE STORE API SETTINGS?

#### Downsampling and Query Efficiency

The Querier also dictates the level of downsampling permitted to the Store Gateway. Downsampling is disabled by default but can be enabled with `--query.auto-downsampling` on the Querier. The UI or tools like Grafana automatically calculate an appropriate 'step' for queries, influencing the `max_resolution_window` parameter in store API requests. For long-range queries, this feature enhances efficiency by reducing the volume of data processed, facilitating pattern detection over extended periods. Once a pattern is identified, zooming in provides more detailed data. Therefore, it is recommended to maintain equal retention for both raw and downsampled data for a comprehensive analysis range.

Let's see two examples of how the Querier handles downsampling:

* **A one hour range query**: Commonly, the UI requires 250 points to draw a graph. The UI computes a 14s step (3600s/250). From there, the Querier computes the `max_resolution_window` parameter to be sent to stores as the requested step divided by 5 as stated in the [documentation](https://thanos.io/tip/components/query.md/#auto-downsampling). In this case it will be 2s. Which is much lower than the first 5m level downsamplig. No dowsampled data will be fetched.
* **A one month range query**: The UI computes a 10368s (30d*24h*3600s/250) step. The querier sets the value of the `max_resolution_window` as 2074s (10368/5). This resolution is compatible with the 5m downsampled step (300s) but too low for the 1h downsampling (3600s). In this case, the store will return the 5m downsampled data.

Considering that the raw data has a 15s samplig period, using the 5m downsampled data will reduce the number of samples by a factor of 20 (300s/15s) and the 1h downsampled data will reduce the number of samples by a factor of 240 (3600s/15s). This drastically improves the query performance of long range queries.

#### Protecting from big queries

The querier is not able to plan 

Setting limits on stores, splitting ruler. WHAT IS THE QUERY LIMITS FOR?

#### Speeding up queries evaluations 

After having set caches for the store, you can also optimize the query component for faster query evaluations.



<!-- Schema comparing both techniques -->

### Wrapping Up: Navigating Thanos Configuration

Through this exploration, I hope you've gained a clearer understanding of how to configure various Thanos components and the implications of certain options. While the Ruler component wasn't discussed to maintain the article's conciseness, it's worth noting that the Ruler also functions as a store for evaluated rules and contributes to pushing resulting blocks to object storage.

There are numerous other significant aspects, such as multi-tenancy configuration and cache management, that we haven't delved into here. However, this guide should provide a solid foundation for getting started with Thanos, helping you comprehend its critical components and their interactions.

### Annexes

#### Metrics terminology: Samples, Labels and Series

* **Sample**: A sample in Prometheus represents a single data point, capturing a measurement of a specific system aspect or property at a given moment. It's the fundamental unit of data in Prometheus, reflecting real-time system states.
* **Labels** very sample in Prometheus is tagged with labels, which are key-value pairs that add context and metadata. These labels typically include:

  * The nature of the metric being measured.
  * The source or origin of the metric.
  * Other relevant contextual details.

* **External labels**: External labels are appended by the scraping or receiving component (like a Prometheus server or Thanos Receive). They enable:

  * Sharding: Included in the `meta.json` file of the block created by Thanos, these label are used by the compactor and the store to shard blocks processing effectively.
  * Deduplication: In high-availability setups where Prometheus servers scrape the same targets, external labels help identify and deduplicate similar samples.
  * Tenancy isolation: In multi-tenant systems, external labels are used to segregate data per tenant, ensuring logical data isolation.

* **Series** or **Time Series**: In the context of monitoring, a Series, which is a more generic term is necessarily a time series. A series is defined by a unique set of label-value combinations. For instance:

```
http_requests_total{method="GET", handler="/users", status="200"}
^                                        ^
Series name (label `__name__`)           Labels (key=value format)                     
```

In this example, http_requests_total is a specific label (`__name__`). The unique combination of labels creates a distinct series. Prometheus scrapes these series, attaching timestamps to each sample, thereby forming a dynamic time series.

For our discussion, samples can be of various types, but we'll treat them as simple integers for simplicity.

The following schema illustrates the relationship between samples, labels and series:

<img src="img/life-of-a-sample/series-terminology.png" alt="Series terminology" width="500"/>

#### TSDB terminology: Chunks, Chunk Files and Blocks

Thanos adopts its [storage architecture](https://thanos.io/tip/thanos/storage.md/#data-in-object-storage) from [Prometheus](https://prometheus.io/docs/prometheus/latest/storage/), utilizing the TSDB (Time Series Database) [file format](https://github.com/prometheus/prometheus/blob/release-2.48/tsdb/docs/format/README.md) as its foundation. Let's review some key terminology that is needed to understand some of the configuration options.

**Samples** from a given time series are first aggregated into small **chunks**. The storage format of a chunk is highly compressed ([see documentation](https://github.com/prometheus/prometheus/blob/release-2.48/tsdb/docs/format/chunks.md#xor-chunk-data)). Accessing a given sample of the chunk requires decoding all preceding values stored in this chunk. This is why chunks hold up to 120 samples, a number chosen to strike a balance between compression benefits and the performance of reading data.

Chunks are created over time for each time series. As time progresses, these chunks are assembled into **chunk files**. Each chunk file, encapsulating chunks from various time series, is limited to 512MiB to manage memory usage effectively during read operations. Initially, these files cover a span of two hours and are subsequently organized into a larger entity known as a **block**. 

A **block** is a directory containing the chunk files in a specific time range, an index and some metadata. The two-hour duration for initial blocks is chosen for optimizing factors like storage efficiency and read performance. Over time, these two-hour blocks undergo horizontal compaction by the compactor, merging them into larger blocks. This process is designed to optimize long-term storage by extending the time period each block covers.

The following schema illustrates the relationship between chunks, chunk files and blocks:

<img src="img/life-of-a-sample/storage-terminology.png" alt="TSDB terminology" width="900"/>