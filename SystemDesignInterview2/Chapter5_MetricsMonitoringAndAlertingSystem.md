# Chapter 5. Metrics Monotoring and Alerting System

<br/>

# Step 1. Understand The Problem and Establish Design Scope

## High-Level requirements and assumption

- 100 million daily active users
- 1000 server pools, 100 machines per pool, 100 metrics per machine => ~10 million metric
- 1 year data retention
- Data rention policy : raw form for 7 days, 10minute resolution for 30 days, 1-hour resolutionn for 1 year
- Metrics can be monitored including Low-level (ex) EPU usuage etc), High-level (ex) Request count)

## Non-functional requirements

- Scalability, Low latency, Reliability, Flexibility

<br/>

# Step 2. Propose High-Level Design and Get Buy-In

## Fundamentals

![KakaoTalk_Photo_2024-04-01-09-44-30 001](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/5befa411-8d47-489d-96bc-519a474fef46)

## Data model
- It is recorded as a time series that contains a set of values with their associated timestamps.

example) CPU load

![KakaoTalk_Photo_2024-04-01-09-44-30 002](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/d5c972c9-5f0c-4bc6-af75-92ee3f05b85a)

## Data access pattern

![KakaoTalk_Photo_2024-04-01-09-44-30 003](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/70c19de4-ebb1-4c07-86bf-2f333a2ad9f7)

- y-axis : a time series (uniquely identified by the names and labels)
- x-axis : time

<br/>

- `write load` : heavy

+) Many time-series data points written at any moment

- `read load` : spiky

+) Visualization and alert services send queries to the DB and depending on the access patterns of the graphs and alerts

<br/>

## Data Storage System

### Relational DataBase
- General Purpose RDB does not perform well under constant heavy write load
- It would require expert-level tuning to make it work at scale

ex)
- Time-Series Data : computing the moving average in a rolling time window requires complicated SQL that is difficult to read
- Tagging/Labeling data : Need to add inde

<br/>

### +) PostgreSQL write logic

**[MVCC (Multi Version Concurrency Contrl)]**

- All row in PostgreSQL are intact
- Index points the heap page which store the present data

**[Write Amplification]**

- If data is updated and new page is created, that row renews all the indexs which it have

**[Fill Factor]**

- How much data might be filled by a page

If 100%)
1. Create new page
2. Create new version row in new page
3. Change all the index's pointers to new page heap

If 80%)
1. New version row creates in the same page

= Hot Update

**Advantages**

1. Vacumm less data
2. Reduce the WAL file size
   
<br/>

### NoSQL

- Cassandra, Bigtable can both be use for time series
- However, it would require deep knowledge of the internal workings of each NoSQL

+) NoSQL does not use WAL file like RDS
- except Cassandra, MongoDB, Redis, DynamoDB 

### Time Series Database

#### OpenTSDB
- Distributed time-series database
- It is based on Hadoop and Hbase, running a Hadoop/HBase cluster adds complexity

#### MetricsDB
- Twitter uses and Amazon offers Timestream as a time-serie sdatabase

#### InfluxDB, Prometheus
- They are designed to store large volumes of time-series data
- Quickly perform real-time analysis on that data
- Rely on an in-memory cache and on-disk storage
- It is efficient aggregation and analysis of a large amount of time-series data bby labels
- Handle durability and performance quite well
ex) 8 core, 32GB RAM > 250,000 writes per second

<br/>

### +) Carnidality

- In the context of databases, refers to the uniqueness of data values contained in a particular column of a database table
- **Low Cardinality** : It contains a large number of "repeats" of a small set of unique values
- **High Cardinality** : Most of all of its values are unique

<br/>

## High-level design

![KakaoTalk_Photo_2024-04-01-09-44-30 004](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/c3a39d0f-1747-4289-a6ea-6de304fe505c)

<br/>

# Step 3. Design Deep Dive

## Metrics collection

![KakaoTalk_Photo_2024-04-01-09-44-30 005](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/819ba835-85d2-49db-aeb4-740e14ef367a)

- It's acceptable for occasional data loss

<br/>

## Pull Model

- Over HTTP
- The metrics collector needs to know the complete list of service endpoints to pull data from

ex) Use a file to hold DNS/IP information for ever service endpoint on the "metric collector" servers

- etcd, ZooKeeper etc can be notified by the Service Discovery component whenever the list of service endpoints changes

![KakaoTalk_Photo_2024-04-01-09-44-30 006](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/adddacb8-c886-4def-bedb-080f8c8dc9dc)

### +) etcd
- A distributed key-value store
- store and manage all its cluster data, including configuration data, state data, and the metdata that represents the state of the cluster
- TTLs for Keys
- Implementing robust backup and recoverty procedures for etcd data to protect against data loss or corruption

### Pull Model in detail

![KakaoTalk_Photo_2024-04-01-09-44-30 007](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/4784da57-8d5e-42a3-b702-22cdfe72b525)

1. The metric collector fetches configuration metadata of service endpoint from Service Discovery
2. The metric collector pulls metrics data via a pre-defined HTTP endpoint
3. Optionally, the metrics collector registers a change event notification with Service Discovery to receive an update whenever the service endpoints change

### Scailiability
- A single metrics collector will not be able to handle thousands of servers
- Designate each collector to a range in a consistent hash ring, map every single server being monitored by its unique name in the hash ring

![KakaoTalk_Photo_2024-04-01-09-44-30 008](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/f2106fdd-23b4-4ecb-ab81-f263bc2ad0be)

<br/>

## Push Model

![KakaoTalk_Photo_2024-04-01-09-44-30 009](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/2fd14355-adfd-411b-a7e4-16d28cd72f74)

- A collection agent is commonly installed on every server being monitored
- The collection agent may also aggregate metrics logically before sending them to metric collectors

### Aggregation
- An effective way to reduce the volume of data
- Problem : If the servers are in an auto-scaling group, then holding data locally might result in data loss
- Solution : The Metric collector should be in an auto-scling cluster with a load balancer in front of it

![KakaoTalk_Photo_2024-04-01-09-44-30 010](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/62dfde4f-24ad-44d5-b365-00168003ea7d)

<br/>

## Push or Push?

### Easy Debugging
- `Pull` : One endpoint

### Health check
- `Pull` : If an application server doesn't respond to the pull, you can quickly figure out if an application server is down
- Push : The problem might be cause by network issues

### Short-lived jobs 
- `Push` : Some of the batch jobs might be short-lived and don't last long enough to be pulled

### Firewall or complicated network setups
- Pull : It is potentially problematic in multiple data center setups
- `Push` : If the metrics collector is set up with a LB and Autoscaling group, it is possible to receive data from anywhere

### Performence
- Pull : Use TCP
- `Push` : Use UDP

<br/>

## +) UDP (User Datagram Protocol)

![image](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/ca25e9d9-0663-44d0-8f38-f0b6e635cc8a)

**Datagram Structure** : It is a self-contained, independent packet of data that includes a header and payload

**+) Packet** : It is more generic and is used to describe any enncapsulated data unit that is transmitted across a network

### Memory Use

![Untitled](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/a0bce4ac-e1cf-46f8-8aef-d7f9d375ebe7)

**+) NIC (Network Interface Card)**

![image](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/1c5911a7-e4a8-46c9-a71b-ba12114a89e7)

**+) DMA (Direct Memory Access)**

![Untitled](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/b5644f1c-a287-42ec-97f9-3f5952370e23)

**Socket Buffers**
- UDP is typically copied into a socket buffer in the kernel space. (TCP as well)
- The size of these buffers is much smaller compared to TCP, as there's no need to store data for potential retransmission

**Receive Buffer**
- If this buffer becomes full, incoming datagrams can be discarded

### Data Handling

**No Connection State**
- It does not maintain a conenction state between endpoints
- No need for storing information about the connection

**Direct to Application**
- There's no need for the data to be acknowledged, sequenced or otherwise processed to fit into a stream

**Handling in the Network Stack**
- UDP packets are processed with minimal interventions
- After basic checks, the payload is passed directly to the application layer

### Connection Management

**TCP**
- OS maintains state information about each connection, including sequence numbers for data packets, acknowledgment numbers, window size for flow control and more

**UDP**
- Each UDP datagram is independent of the others, with minimal header information
- Th IS does not keep taack of UDP communications the way it does with TCP sessions

<br/>

### Data authenticity
- Pull : Application servers to collect metrics from are defined in config files in advance
- Push : Any kind of client cna push metrics to the metric collecters. Require whitelisting servers, or authentication

<br/>

## Scale the metrics transmission pipeline

![KakaoTalk_Photo_2024-04-01-09-44-30 011](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/d5cb8cb9-060d-4380-93be-2437df0444c6)

- The metrics collector is a cluster of servers
- The cluster receives enormous amounts of data
- It is set up for autoscaling, to ensure that there are an adequate number of collector instances to handle the demand

### Queueing components
- To mitigate data loss

![KakaoTalk_Photo_2024-04-01-09-44-31 012](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/58c43e32-8a20-4500-929d-36047186afb7)

1. The metric collector sends metrics data to queueing systems like Kafka
2. The consumers or streaming processing services push data to the time-series database

**[Advantages]**
- Kafka : a highly reliable and scalable distributed messaging platform
- It decouples the data collection and data processing service
- Easily prevent data loss when the database is unavailable

### Scale through Kafka

![KakaoTalk_Photo_2024-04-01-09-44-31 013](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/390579f4-5225-4d1a-aecd-e72f8ce29177)

- Configure the number of partitions based on throughput requirements
- Partition metrics data by metric names, so consumers can aggregate data by metric names

<br/>

## Where aggregation can happen

### Collection agent (Client Side)
- simple aggregation logic

### Ingestion pipeline (Before writing to the storeage)
- We need stream processing engines such as Flink
- Advantages : The write volume will be significantly reduced since only the calculated result is written to the DB
- Disadvantages : Handling lat-arriving events could be a challenge an another downside is existed

### Query Side
- Faw data can be aggregate over a given time period at query time
- Advantages : Not data loss with the approach
- Disadvantages : The query speed might be slower

<br/>

## Cache Layer

![KakaoTalk_Photo_2024-04-01-09-44-31 014](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/52939f0a-bd7e-4920-9d99-6273418be4e4)

<br/>

## Storage layer 

### Data Encoding and Compression

![KakaoTalk_Photo_2024-04-01-09-44-31 015](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/c8a6afb3-726b-4154-b331-24a34f1a527c)

- Just store the difference between the time, instead of the full timestamp of 32 bits

### +) 32 bits = Unix Timestamp
- counts the number of seconds that have elapsed sinze the "`Unix epoch`" (00:00:00 UTC on 1 January 1970)
- 32-bit Signed Integer :  -2,147,483,648 ~ 2,147,483,647 (−2^31 ~ 2^31-1)

### Downsampling

- The process of converting high-resolution data to low-resolution to reduce overall disk usage

![KakaoTalk_Photo_2024-04-01-09-44-31 016](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/86f8ec9a-fdc2-4922-a05f-3b1aaa8f013f)

![KakaoTalk_Photo_2024-04-01-09-44-31 017](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/99d23b63-fd7c-475d-ac9a-b56c5683d435)

### Cold storage
- The storage of inactive data that is rarely used
- The financial cost for cold sotrage is much lower

ex) AWS Glacier

<br/>

## Alert System

![KakaoTalk_Photo_2024-04-01-09-44-31 018](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/ea9017f6-b430-4604-a711-30f367a9a2de)

1. Load config files to cache servers. Rules are defined as configure files on the disk. (ex) YAML format)
2. The alert manager fetches alery configs from the cache
3. Based on config rules, the alert manager calls the query service at a predifined interval

![KakaoTalk_Photo_2024-04-01-09-44-31 019](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/5c9512c9-da90-4370-9514-280ca879c7ab)

- Filter, merge and dedupe alerts

4. The alert store is a key-value database that keeps the state of all alerts. It ensures a notification is sent at least once.
5. Eligible alerts are inserted into Kafka
6. Alert consumers pull alert events from Kafka
7. Alert consumers process alert events from Kafka and send notifications over to different channels such as email, text message, PagerDuty or HTTP endpoints

### +) YAML format
```
person:
  name: John Doe
  age: 30
  married: true
  children:
    - name: Jane Doe
      age: 10
    - name: Joe Doe
      age: 12
  address:
    street: "1234 Elm Street"
    city: "Anytown"
    state: "Anystate"
    zipcode: "123456"
```
