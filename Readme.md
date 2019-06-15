# AWS Big Data Specialty Cert Study

***Disclaimer***: *this guide is the tail end of my studying so is not complete, mostly just focusing on trivia/gotchas.*

Anti-patterns taken from [AWS Big Data Whitepaper](https://d1.awsstatic.com/whitepapers/Big_Data_Analytics_Options_on_AWS.pdf).

## Data Collection

### [Kinesis Data Streams](https://docs.aws.amazon.com/streams/latest/dev/introduction.html)
- **Service notes**
  - 200 ms latency (1 standard consumer), 70 ms with enhanced fan out HTTP2 push - both considered real time
  - [Server side encryption](https://docs.aws.amazon.com/streams/latest/dev/what-is-sse.html) supported (default CMK, user managed CMK, or KMS imported key material)
- **Limits**
  - retention period default 24 hours, max 7 days
  - shard can ingest 1 MiB/s or 1000 messages/sec, max message size 1 MiB
  - consumer throughput is 2 MiB/s with max of a single API call 10,000 records or 10 MiB
- **Anti-patterns**
  - small scale consistent throughput (less than 200 KB/sec)
  - long term storage/analytics

### [Kinesis Firehose](https://docs.aws.amazon.com/firehose/latest/dev/what-is-this-service.html)
- **Service notes**
  - "near real time"
  - destinations: S3, Redshift, Elasticsearch, Splunk
  - stores records up to 24 hours in case downstream system is unavailable
  - can store pre-transformation records, log transform and delivery errors
  - a Kinesis Data Stream can feed a Firehose stream
- **Limits**
  - max record size 1,000 KiB, throughput just soft limits
  - buffer hints:
    - S3: 1-128 MiB
    - ElasticSearch service: 1-100 MiB
    - Lambda processor: 1-3 MiB

### [Kinesis Analytics](https://docs.aws.amazon.com/kinesisanalytics/latest/dev/what-is.html)
- **Service notes**
  - Input streams: Kinesis data streams *or Firehose*, static reference tables from S3 for joins
  - `pump`s select from one stream and insert into the next stream
  - > You can have multiple writers insert into an in-application stream, and there can be multiple readers selected from the stream. Think of an in-application stream as implementing a publish/subscribe messaging paradigm.
  - Window types:
    - [Stagger Windows](https://docs.aws.amazon.com/kinesisanalytics/latest/dev/stagger-window-concepts.html): A query that aggregates data using keyed time-based windows that open as data arrives. The keys allow for multiple overlapping windows.  Using stagger windows is a windowing method that is suited for analyzing groups of data that arrive at inconsistent times.
    - [Tumbling Windows](https://docs.aws.amazon.com/kinesisanalytics/latest/dev/tumbling-window-concepts.html): A query that aggregates data using distinct time-based windows that open and close at regular intervals.
    - [Sliding Windows](https://docs.aws.amazon.com/kinesisanalytics/latest/dev/sliding-window-concepts.html): A query that aggregates data continuously, using a fixed time or rowcount interval.
- **Limits**
  - 1 input stream, 1 reference source, 3 output streams

### [Kinesis Video Streams](https://docs.aws.amazon.com/kinesisvideostreams/latest/dg/what-is-kinesis-video.html)
- **Service notes**
  - > You can also send non-video time-serialized data such as audio data, thermal imagery, depth data, RADAR data, and more
  - > Metadata can either be transient, such as to mark an event within the stream, or persistent, such as to identify fragments where a given event is taking place. A persistent metadata item remains, and is applied to each consecutive fragment, until it is canceled.

#### [Kinesis Agent](https://docs.aws.amazon.com/streams/latest/dev/writing-with-agents.html)
- stand-alone Java program, reads from config and processes files
- can do some transformations like multiline, CSV to JSON, or Apache log to JSON

### Kinesis Libraries/Helpers
#### [KPL](https://docs.aws.amazon.com/streams/latest/dev/developing-producers-with-kpl.html)
- [Key concepts overview](https://docs.aws.amazon.com/streams/latest/dev/kinesis-kpl-concepts.html)
- manages retries and batching
- types of batching:
  - [Aggregation](https://docs.aws.amazon.com/streams/latest/dev/kinesis-kpl-concepts.html#kinesis-kpl-concepts-aggretation):
    - many small **user** records into less **data** records
    - decreases cost since less shards needed, also increases client PUT performance
    - aggregated user records go to the same shard since a single data record
    - firehose supports KPL deaggregation when fed from a kinesis data stream [link](https://docs.aws.amazon.com/streams/latest/dev/kpl-with-firehose.html)
  - [Collection](https://docs.aws.amazon.com/streams/latest/dev/kinesis-kpl-concepts.html#kinesis-kpl-concepts-collection):
    - wait and send multiple **data** records at the same time using `PutRecords` API rather than sending each in its own HTTP transaction
    - increases client PUT performance since smaller number of requests
    - records in `PutRecords` API call don't necessarily have to go to the same shard
- [Monitoring](https://docs.aws.amazon.com/streams/latest/dev/monitoring-with-kpl.html) has metric level (`NONE`, `SUMMARY`, or `DETAILED`) and granularity level (`GLOBAL`, `STREAM`, or `SHARD`)
- integrates with KCL to deaggregate records

#### [KCL](https://docs.aws.amazon.com/streams/latest/dev/developing-consumers-with-kcl.html)
- > The KCL takes care of many of the complex tasks associated with distributed computing, such as load balancing across multiple instances, responding to instance failures, checkpointing processed records, and reacting to resharding.
- uses DynamoDB to coordinate between multiple workers for leases and checkpoints
- can deaggregate KPL data records into user records
