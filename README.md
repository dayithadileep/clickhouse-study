# ClickHouse: A Database Case Study

> An in-depth study of **ClickHouse**, an open-source, column-oriented OLAP database, covering its architecture, key features, use cases and limitations, with a comparison against **MySQL** and a hands-on demo.

---

## 1. Project Summary

ClickHouse is an open-source, column-oriented OLAP database developed at Yandex for analyzing large volumes of data. It improves analytical performance through columnar storage, MergeTree storage engines, sparse primary indexes, data-skipping indexes, compression, vectorized execution and parallel processing. It supports distributed deployments, replication, SQL analytics, security features and integrations with other technologies.

It is useful for web analytics, logs, monitoring, IoT and large-scale business analysis, but frequent transactional updates are not its main strength. Compared with MySQL, which is commonly used for transactional workloads, ClickHouse specializes in analytics. Organizations can therefore use MySQL to manage everyday transactions and ClickHouse to analyze the resulting data and create reports and dashboards.

## 2. Objectives

- Understand what ClickHouse is and why it was built.
- Study the internal techniques that make it fast for analytical queries.
- Identify the workloads where it performs well, and where it does not.
- Compare ClickHouse with MySQL on design, workload and performance characteristics.
- Demonstrate ClickHouse with real SQL: analytical queries and basic CRUD operations.

## 3. Background

| Item | Detail |
|---|---|
| Type | Column-oriented DBMS for OLAP |
| Origin | Developed at Yandex to power large-scale web analytics |
| License | Open source |
| Query language | SQL (with ClickHouse-specific extensions) |
| Typical scale | Billions of rows, queried interactively |

Traditional databases are built to store and update individual records quickly. Analytical systems have a different job: scan huge numbers of rows, but only a few columns, and aggregate them. ClickHouse was designed specifically for that second job.

## 4. Core Concepts: OLTP vs OLAP

| | OLTP (Online Transaction Processing) | OLAP (Online Analytical Processing) |
|---|---|---|
| Purpose | Run day-to-day operations | Analyze data to support decisions |
| Typical operations | Insert, update, delete single rows | Aggregate and scan millions or billions of rows |
| Query shape | Short, many per second | Long, fewer, complex |
| Storage layout | Row-oriented | Column-oriented |
| Example | Placing an order, updating a balance | "Total sales per region per month" |
| Example systems | MySQL, PostgreSQL | ClickHouse |

## 5. How ClickHouse Works

### 5.1 Columnar Storage

Each column is stored separately on disk. A query that sums listening time reads only the `listen_seconds` column and ignores every other column. This greatly reduces disk I/O for analytical queries, which usually touch a few columns out of many.

```
Row-oriented (MySQL)                 Column-oriented (ClickHouse)
[id | name | country | seconds]      id:      [1, 2, 3, ...]
[id | name | country | seconds]      name:    [a, b, c, ...]
[id | name | country | seconds]      country: [IN, US, UK, ...]
                                     seconds: [210, 180, 95, ...]
```

### 5.2 MergeTree Storage Engines

`MergeTree` is the main table engine family.

- Data is written in immutable **parts**, sorted by the table's `ORDER BY` key.
- A background process **merges** smaller parts into larger ones.
- Tables can be **partitioned** (for example by month) so old data can be dropped or skipped quickly.
- Variants add features: `ReplacingMergeTree`, `SummingMergeTree`, `AggregatingMergeTree`, and `ReplicatedMergeTree` for replication.

### 5.3 Sparse Primary Index

Unlike a MySQL B-tree index that points to every row, ClickHouse stores one index entry per **granule** (a block of rows, 8192 by default). The index is small enough to stay in memory, and lets ClickHouse skip whole granules that cannot match a filter on the sorting key.

### 5.4 Data-Skipping Indexes

Secondary indexes that store summaries per block of granules, so blocks can be skipped without being read. Common types:

- `minmax`: min and max value of a block
- `set`: the distinct values in a block
- `bloom_filter`: probabilistic membership check

### 5.5 Compression

Because each column holds values of the same type, it compresses very well. ClickHouse supports codecs such as `LZ4` (default, fast) and `ZSTD` (higher ratio), plus specialized codecs like `Delta` and `DoubleDelta` for timestamps and counters. Smaller data means less disk space and less I/O.

### 5.6 Vectorized Execution

Instead of processing one row at a time, ClickHouse processes **blocks of column values** at once, making efficient use of CPU caches and SIMD instructions.

### 5.7 Parallel Processing

A single query is split across multiple CPU cores, and, in a cluster, across multiple servers. Throughput scales with the hardware available.

### 5.8 Query Flow (Simplified)

```
SQL query
   |
   v
Parse and plan
   |
   v
Use sparse index + skipping indexes to pick which granules to read
   |
   v
Read only the needed columns (decompress)
   |
   v
Vectorized, parallel execution across cores / shards
   |
   v
Merge partial results -> final result
```

## 6. Key Features

| Feature | Description |
|---|---|
| Columnar storage | Reads only the columns a query needs |
| MergeTree engines | Sorted, partitioned, mergeable data parts |
| Sparse primary index | Small in-memory index that skips granules |
| Data-skipping indexes | `minmax`, `set`, `bloom_filter` and others |
| Compression | LZ4, ZSTD and specialized codecs |
| Vectorized execution | Block-wise processing for CPU efficiency |
| Parallel processing | Multi-core and multi-server execution |
| Distributed deployment | Sharding across nodes via the `Distributed` engine |
| Replication | `ReplicatedMergeTree` for fault tolerance |
| SQL analytics | Aggregations, joins, window functions, and many built-in analytical functions |
| Security | User and role-based access control, quotas, encrypted connections |
| Integrations | Connectors for databases, message queues, object storage, BI and visualization tools |

## 7. Use Cases

- **Web analytics:** page views, sessions and funnels (the original use case at Yandex).
- **Log analysis:** searching and aggregating large volumes of application and server logs.
- **Monitoring and observability:** time-series metrics and events.
- **IoT and sensor data:** high-volume, append-only telemetry.
- **Business intelligence:** dashboards and reports over large datasets.
- **Streaming and media analytics:** listening or viewing behavior, which this project demonstrates.

What these have in common: data is mostly **appended**, rarely changed, and queried with **aggregations over many rows**.

## 8. Limitations

- Frequent single-row `UPDATE` and `DELETE` are not its strength. They run as background **mutations** that rewrite data parts.
- Not designed for multi-statement ACID transactions like a traditional OLTP database.
- Point lookups of single rows are not as cheap as in a B-tree indexed row store.
- Works best with a well-chosen `ORDER BY` key and denormalized or pre-joined data; very join-heavy workloads need more care.
- Operating a cluster requires planning (sharding, replication, coordination).

## 9. ClickHouse vs MySQL

| Aspect | MySQL | ClickHouse |
|---|---|---|
| Primary workload | OLTP (transactions) | OLAP (analytics) |
| Storage model | Row-oriented | Column-oriented |
| Indexing | Dense B-tree indexes | Sparse primary index + skipping indexes |
| Read pattern | Few rows, all columns | Many rows, few columns |
| Updates and deletes | Fast, row-level | Slow, run as background mutations |
| Transactions | Full ACID support | Limited |
| Aggregation speed on large data | Slower | Very fast |
| Compression | Moderate | High, due to columnar layout |
| Best for | Applications, e-commerce, banking | Reporting, dashboards, log and event analysis |

**Takeaway:** the two are not competitors. They are built for different jobs.

## 10. Using MySQL and ClickHouse Together

A common architecture:

```
  Application
      |
      v
   MySQL  (everyday transactions: orders, users, payments)
      |
      |  replicate / export / stream
      v
 ClickHouse  (analysis of the resulting data)
      |
      v
Reports and dashboards
```

MySQL stays the system of record for transactions. ClickHouse receives a copy of the data and answers heavy analytical questions without slowing down the application database.

## 11. Demo

The demo SQL is in [`queries.txt`](queries.txt) and has two parts. This section explains what each one does.

### 11.1 Music Streaming Analytics

Tables used: `listening_events`, `tracks`, `artists`, `users`.

Design notes: `listening_events` is the large fact table, which would be partitioned by month and sorted by time to suit the time-based analysis. The other three are small lookup (dimension) tables.

| Analysis | What it does | ClickHouse features shown |
|---|---|---|
| Artists with the highest listening time | Joins events to tracks to artists. Reports total listening time, number of plays, skips and skip rate per artist, and returns the top 10. | Multi-table joins, `sum`, `count`, conditional aggregate `countIf` |
| Countries generating the most listening time | Aggregates listening behavior per country: number of events, total and average listening time, and skipped events. Useful for regional analysis. | `GROUP BY`, `avg`, `round`, `countIf` |
| Most popular genres by month | Groups plays and listening time by month and genre to show how genre popularity changes over time. | `toYYYYMM` date function; partition pruning when filtered by date |

### 11.2 Library Management: Basic CRUD

A small example showing ClickHouse's basic operations and how they differ from MySQL.

| Step | What happens |
|---|---|
| Create | A `library_management` database and a `books` table (`book_id`, `title`, `author`, `category`, `available_copies`) using the `MergeTree` engine, ordered by `book_id` |
| Insert | Five books are added in a single `INSERT` |
| Read | All books are selected in order of `book_id` |
| Update | `available_copies` of one book is changed using `ALTER TABLE ... UPDATE` |
| Delete | One book is removed using `ALTER TABLE ... DELETE` |

| Operation | MySQL | ClickHouse |
|---|---|---|
| Update | `UPDATE` statement (immediate, row-level) | `ALTER TABLE ... UPDATE` (background mutation) |
| Delete | `DELETE FROM` statement | `ALTER TABLE ... DELETE` (background mutation) |

Mutations are asynchronous and rewrite whole data parts, so the result may not be visible the instant the statement returns. This is why ClickHouse suits append-heavy analytics, not frequently changing records.

## 12. Setup and Installation

### Option A: Docker (recommended)

```bash
docker run -d --name clickhouse \
  -p 8123:8123 -p 9000:9000 \
  clickhouse/clickhouse-server

docker exec -it clickhouse clickhouse-client
```

### Option B: Native install

Follow the official installation guide for your operating system at <https://clickhouse.com/docs/en/install>.

### Running the demo

1. Open `clickhouse-client` (or the web UI at `http://localhost:8123/play`).
2. For the library demo, run the library section of `queries.txt` in order.
3. For the music demo, create the four tables and load sample data first, then run the three music queries from `queries.txt`.

## 13. Repository Structure

```
.
├── README.md        # Project documentation (this file)
├── ClickHouse.pptx  # Presentation: concepts, analysis, use cases, demo
└── queries.txt      # SQL used in the demo
```

## 14. Conclusion

ClickHouse is purpose-built for fast analytics on large datasets. Columnar storage, MergeTree engines, sparse and skipping indexes, compression, vectorized execution and parallel processing together let it scan and aggregate billions of rows quickly. Its trade-off is weaker support for frequent row-level updates and transactions, which is exactly where MySQL is strong. The practical approach is to use each for what it does best: MySQL for transactions, ClickHouse for analysis, reports and dashboards.

## 15. References

- ClickHouse documentation: <https://clickhouse.com/docs>
- MergeTree table engine: <https://clickhouse.com/docs/en/engines/table-engines/mergetree-family/mergetree>
- MySQL documentation: <https://dev.mysql.com/doc/>
