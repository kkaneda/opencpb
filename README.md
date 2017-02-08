# OpenCPB: Open Computing Platform for Bioinformatics

## Objective

The goal of the project is to build an open computing platform for
Bioinformatics. Our initial focus is to build a storage engine
specialized for processing large-scale genome data efficiently.

The storage engine (let’s call it *GDB*) provides the following functionality:

- Upload and store genome data associated with metadata
- Read genome data (with some in-memory caching)
- Update and index metadata
- Run coprocessors (e.g., motif search, genome sequencing, machine learning)

The design principle of GDB is *"simple”*, and it is highly
inspired by [Cockroach DB](http://cockroachlabs.com).

## A Naive Design and Implementation

Our first is to design and implement a naive prototype that can
provide minimum required features. We will use this prototype to
verify our basic design direction and test integration of core
building blocks.

Our goal on this phase is to build a yet another key-value blob store.
The key-value stores provides minimal features required to run it in
real production environment. We focus on keeping the initial design
simple as much as possible so that we can extend it later easily. We
can say that the goal would be to build a reference key-value store.

### External API

The API we provide is a write and read of a blob associated with
a primary lookup key. 

```
Put(key, blob) error
Get(key, position) (blob, error)
List(keyPattern) (keys, error)
Delete(key) error
```

There is no secondary key or file system like hierarchical namespace.
All written data are immutable. 

Transactional guarantee is minimum. A blob that is being written is
not visible, and it will be eventually become visible.

### Architecture

The system consists a centralized master and storage nodes. The master
manages all metadata (e.g., location of data stripe). The master has N
replicas, and [RAFT](https://raft.github.io/raft.pdf) is used to
manage a leader lease. Only the leader can accept read/write to
metadata. RAFT is also used by The leader and followers to sync their
states (note on statement-based replication and data-based
replication). The backed storage for RAFT is
[RocksDB](http://rocksdb.org/) (note: whether a RAFT leader can simply
be a leader. Otherwise, how we collocate the two leaders.)

The master does minimum monitoring of each storage node. The master
sends period heartbeat for basic health check, and it also collects
node stats like disk limit/usage. Based on the collected data, the
master makes a decision on where data should be allocated and
replicated. It also keeps track of basic failure domain information
(e.g., datacenter, power, switch, rack) so that data replicas are
distributed well (e.g., do not put all replicas in a single rack).

// The master also keeps a list of all storage nodes and does periodic
health check.

When clients write/read, they first talk to the master 

### Master 

The master consists of the following components:

- txn manager
- storage node monitor (this can be easily 
- admin 
- store (append only) wrapper around RAFT


Each component does not 

TODO(kaneda): 
- How to locate masters
- How to avoid mono-architecture for the master

API exposed to clients.

```
OpenTransaction()
CloseTransaction()
List()
```

Admin API

```
AddStorageNode()
RemoveStorageNode()
```

API exposed to other master instances.


Internal state machine

How do we choose a set of nodes We don't care data locality here for
simplicity. Thus, 

- pack the file to the same set of nodes.
- do not 

```
Input:
- primary key
- data size
- replication params

Output:
- 
```

Note: what are pros and cons of allocating all stripes of a given file
to the same set of machines?


```

```


The master's state

- For each primary key, the master keeps maps from primary_keys to blob.
- a collection of machines (see below).
- txns: open, close, associated data (e.g., )


- Whenever a state is updated, the change is recorded, and propagated to RAFT followers.
- We can also view historical changes to the master state.

- A master needs to clean up garbage data that is .


A client locates the master with DNS or other 



### Data Replication

A blob is split into multiple "stripes". Each strip consists of N data
chunks and M code chunks.

An encoding algorithm we use is simply
[Reed-Solomon](https://en.wikipedia.org/wiki/Reed%E2%80%93Solomon_error_correction).

How do we want to send data to all replicas? Can 

Does normal data replication require RAFT? How do we distribute
chunks? I might lack basics of RAID stuff.

- Option 1: just do a simple replication 
  - cons: can be slow. it's not easy to make every node catch up correctly
- Option 2: Create a new RAFT group on every write.
  - do we store RAFT log and snapshot?
  - each write is a data stripe, and there is no mutation. So, we don't really need to take a snapshot.
  - One RAFT for managing the master state. This requires voing
    - Then, we want to form a RAFT group with this leader and other followers (
- Who is a leader? 



### Storage Node 

Each storage node is a key-value store also backed by RocksDB. Keys
are of the form `<primary_key>:<strip_id>:<replica_id>`>. Values are
either data or code. (note: do we need special annotation for coding region?)

- When a storage node goes down, it will trigger data reconstruction. 
- How to Join and leave


### Interaction with Clients

A client 

A read/write request first 

```
Put(blob): 
  repeat until succeed
    start txn
    send metadata to the master
    get info on where should files need to be sent.
    
    for (;;) {
      if err := write; err != nil {
         break and start from the beginning
      }
    }
    close the txn
   }
}
  
```

When it encounters a transient/permanent failure during write, 
the transaction will be aborted, and  a client needs to retry the entire
procedure.


The same story applies to the read path.

TODO(kaneda): Add some more details. What if 


A client read and data reconstruction. Given a list of machines, it
will find file in a nearby location.

### Failure scenarios 

Whenever a planned/unplanned outage happens, we will lose portion of
replicated data, and we will reconstruct. We don't gracefully migrate
data from A to B.



### Summary

Even this naive prototype requires a reasonable amount of engineering
work, and


## Observation on the Naive Design/Implementation 

The above naive design/implementation reveals many shortcomings and
possible areas where we can optimize and improve. Here are some
obvious limitations:

- Overall performance and availability
- All stability coming from various engineering efforts and experience
  from production usage.

- A centralized master can easily be a bottleneck.
- Failure detection and node discovery to reduce configuration
- Planned maintenance should be gracefully handled.
- Automatic retry on failures..
- secondary indexing and other lookup API
- System visibility (
- Security (ACL)
- Data load balancing
- Data locality
- In-memory caching
- Co-processor
- Monitoring and failure recovery
- Encoding schema
- Data compression (if needed)


## An example Use Case

Let's imagine that we have read-pairs of a genome, sequence them
(e.g., compute contigs), and then apply some computation (e.g., check
the quality of the sequencing by comparing it against reference
genome).

One possible implementation of the above workflow with GDB is
following:

1) Upload the read-pairs to GDB storage.
2) Trigger sequencing (manually or via a registered co-processor)
3) Trigger another computation for comparision with reference genome.

Let's look at the individual steps in more details.

### Step 1. Upload

1-1) Create a new namespace for this experiment. Set the ACL.
1-2) Specify a name of the data and attatch tags/labes.
1-3) Upload data.

GDB will partition data and store with erasure coding.

### Step 2. Sequencing

2-1) Upload a Docker image that has a sequence program.
2-2) Specify its resource requiements.
2-3) Send a run command to the job queue.

Here is pseudo code of the genome sequencing

```
parallel_for (...):
  data = readPairs(name, [position_from, position_to])
  contigsData = findContigsLocally(data)
  write(name + "/contigs", contigsData)

[Slave]

```

or maybe actually construct an Eulerian path from a De-brujin grpah.

A naive way to parallelize the workload (let's say with some
master-worker task-steal style) MapReduce or Spark or whatever),

- master specifies 
- workers compare matching of patterns, 
- upload intermediate data to somewhere (probably in-memory, maybe mutable)

- would require fault tolerance obviously.

- maybe it's nice to have some way to explicitly specify an in-memory
  storage, which is not persistent.
  
- I guess the question is how each worker communicates with each other. Do we need assist from GDB?

2-4) Store the result of sequencing in GDB storage.

### Step 3. Sequencing

3-1) 


### Data Characteristics and Access Patterns

We assume here that genome data are generally immutable and the data 
size is huge (e.g., a few TB for storing an entire human genome).

The access pattern of genome data is mostly batch read, and there is
not so much random access like a typical web service (is that true?).

We also don't expect strong support of transactional semantics.


Does read pattern have locality? 
How many 

Donesn't need to opitmize for read-latency. I think it's more
important to optimize data read throughput. Even availablity might be
something we can trade. We don't need to make 

## API



### Upload Genome Data

```
Write(name, genomeData, labels, options)
```


Namespace is 

for HTTP2 (gRPC streaming).

like a typical key-value store.
Primary key 

Upload file format will be
[FASTA](https://en.wikipedia.org/wiki/FASTA_format) or some common
format used for genome data.

labels are just 

Just use a different name when you want to store processed data.

It should be just write. Can create a link to 
the original data using metadata.

Indexing metadata. 

Maybe you can specify expiration time.

### Manage Metadata 


```
Update(name, labels)
```

Can update only labels. Will be index automatically (later).

We can provide batch update if it's helpful.

Does this need to be async? Or it doesn't matter?


### Read / Scan / List

```
(genomData, labels) = Read(key, options) 
```

`key` can be a `name`, `labels`, or a combination of them. This can be
a more SQL-like API.

`options` specify several things, like read only labels, read specific
position/length of data, etc.


```
(names, labels) = List(namePrefix)
```

We can have `Scan`, which returns more than one results.



### Delete


```
Delete(name)
```


### In-memory Pinning And Prefetch



### Coprocessor 


```
run(container, config)
```

basic resource requirements?

Can run on top of Spark

Centralized 

Upload a container image, which is 

Load-balancing across multiple datacenters.

Spark or MapReduce


### Watcher 

Get notification like feed.

Run a registerd callback/coprocessor 

### Data Access Control

SSL/TLS auth 

not sure we want to set permission (maybe per database instance).
Having some namespace is nice to accidental collide.

## Basic Architecture

The design principle of GDB is to be *"simple”*, and it is highly
inspired by [Cockroach DB](http://cockroachlabs.com).

- Stored data are replicated to provide high availability with
  eventual consistency (including cross-datacenter replication).

- Stored data are partitioned and stored on top of a collection of
  distributed machines. Some encoding schema like
  [Reed-Solomon](https://en.wikipedia.org/wiki/Reed%E2%80%93Solomon_error_correction)
  is used.

There is no transactional guarantee, which we don't need given the
nature of the data we store.


So, it's a key value store. in-memory 

what we want to store

- indices
  - (label_name, label_value) -> a list of (data chunk) -> a list of (stripe) -> a list of data location (replicas)
  - (label_name, label_value) -> a list of metadata

Note on RAFT:

- A RAFT group is created for each file write. The master assigns a new ID.
- A writer becomes a RAFT leader.
  - writing *code* is not really a replication.
  - then how do we deal with transient/permanent failues? 
- Once the file write completes, RAFT group will also end. (We don't need to keep it.)
- This is not really RAFT. We just 


TBD:
- Master should be sharded, to store 
  
Metadata should be small enough (or maybe store that in the master instead of applying erasure coding)




- Data locality is based on primary key(?) Or this doesn't matter much if our data chunk size is too large.
-

### Components

- *Gateway*: Frontend for serving requests from clients.
- *Master*: A centralized server for There is a master. Master keeps a map from par
- *Storage nodes*
- *Memory nodes*
- *Compute nodes*
- *Centralized job queue*

Can be separate physical machines or can be logically partitioned on the same machine.

The master's state is persisted in 
It is backed up by
[RAFT](https://raft.github.io/raft.pdf) and [RocksDB](http://rocksdb.org/).

A storage engine has a centralized job queue for managing co-processors.

### Single Master v.s. Hierachical Master?

### 
  

## Data layout 

First, split a given genome sequence into multiple small chunks. For each chunk, apply erasure Coding. 

Address

## Data Spliting



### Node Join and Leaving: How to manage a list of nodes in the 

- Let's have a simple master node (with replication)
- Heartbeat with some hierarchical structure
- Can be Gossip like Cockroach 
- Report available resources on the machine.
- Health check

### Data Splitting and Rebalancing

Maybe we don't need any rebalance?

If we do, is there any chance of race (especially core master data)

Basically we need to shuffle RAFT groups, 

Note that we can do a wild optimization since 

## Replication

Does normal data replication require RAFT? How do we distribute
chunks? I might lack basics of RAID stuff.

- Option 1: just do a simple replication 
  - cons: can be slow. it's not easy to make every node catch up correctly
- Option 2: Create a new RAFT group on every write.
  - do we store RAFT log and snapshot?
  - each write is a data stripe, and there is no mutation. So, we don't really need to take a snapshot.
  - One RAFT for managing the master state. This requires voing
    - Then, we want to form a RAFT group with this leader and other followers (
- Who is a leader? 

## Data Compression 

Are there any known techniques for compressing genome data? 

### Client Side Reconstruction

Does this make sense to always fan out replicas to other multiple DCs?

### Failure Recovery Server Side Reconstruction

## Comparision with Other Storage Systems

There are many existing storage systems. To name a few,

- Large-scale file systems (e.g., Google File System, Colossus, HDFS)
- Key-value stores (e.g., Bigtable, HBase, Cassandora)
- Spanner, Cockroach DB
- Blobstore 
- MongoDB, ElasticSearch

Yes, we might not need yet another storage system, but we believe GDB
is a unique valuable system for the specific purpose, which is genome
data processing. In particular, its blob store focuses on batch
processing, which is not a primary requirement for typical web
services. GDB has coprocessor as first class citizen of its design.

Also, the simple design of GDB will provide an easy of operations and
can be a good reference system.

So, what about MapReduce and GFS/BigTable combination? 

# References

- [Reed Solomon](https://web.eecs.utk.edu/~plank/plank/papers/CS-96-332.pdf)
- [Blob storage discussion on Cockroach DB](https://github.com/cockroachdb/cockroach/issues/243)
- [Suffix array](https://academic.oup.com/bib/article/15/2/138/212729/A-bioinformatician-s-guide-to-the-forefront-of)

http://ieeexplore.ieee.org/document/7347331/ 

[Note on GCE: took 10 days to migrate Everlink's 3PB data.]
307TB per day.
http://www.pcworld.com/article/3167594/data-center-cloud/heres-how-evernote-moved-3-petabytes-of-data-to-googles-cloud.html

