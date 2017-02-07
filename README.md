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

## Data Characteristics and Access Pattern

Here are basic characteristics of genome data we handle:

- Immutable
- Huge data size (e.g., a few TB for storing an entire human genome)

The access pattern of genome data is mostly batch read, and there is
not so much random access like a typical web service (is that true?).

Upload file format will be
[FASTA](https://en.wikipedia.org/wiki/FASTA_format) or some common
format used for genome data.

## Architecture

The design principle of GDB is to be *"simple”*, and it is highly
inspired by [Cockroach DB](http://cockroachlabs.com).

- Stored data are replicated to provide high availability, with
  eventual consistency. It is backed up
  [RAFT](https://raft.github.io/raft.pdf) and [RocksDB](http://rocksdb.org/).

- Stored data are partitioned and stored on top of a collection of
  distributed machines.

There is no transactional guarantee given the nature of the data we store.

A storage engine has a centralized job queue for managing co-processors.


