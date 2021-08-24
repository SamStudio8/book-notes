# Designing Data-Intensive Applications


* Data intensive applications distinguished from compute intensive as the data is the bottleneck, rather than compute cycles
* Raw CPU is rarely the limiting factor for modern applications
* Will explore the building blocks:
   * Storing data (databases)
   * Speeding up reads of expensive results (caches)
   * Searching and filtering (indexing)
   * Asynchronous processing (streams)
   * Crunch accumulated data (batches)


## Foundations of data systems
### Reliable, Scalable, Maintainable
* Tolerating hardware, software and human errors
* Measuring load and performance
* Operability, simplicity and evolvability


* No single tool can now meet all the requirements of a complex system
* Hard to put tools in boxes, Redis is a data store and message queue, Kafka is a message queue with database-like guarantees
* Systems are composites, hidden from users behind an API
* System may provide guarantees such as invalidating or updating caches on application writes such that outside clients will see recent results


The three main concerns of software systems:
* Reliability
   * The system should continue to work correctly even in the face of adversity (hardware, software and human fault)
      * Create the expected output for given inputs 
      * Tolerating user input error
      * Performs reliably for its use cases
      * Prevents unauthorised access and abuse
   * A fault is when something deviates from spec, a failure is when expected service to the user is stopped
   * Think about the type of faults you want to tolerate (you won’t be able to tolerate them all)
   * Job is to prevent faults becoming failures
      * You can increase the rate of faults by triggering them on purpose to help find critical errors - which often come from poor handling of errors (cf. Chaos Monkey)
   * Hardware faults more likely in larger complex systems, and AWS is designed for flexibility and elasticity rather than single machine reliability
   * Software errors can be systematic and more correlated than hardware faults (imagine faults on specific inputs, incorrect validation, runaway processes, leap seconds) or software that relies on other software that returns slow or corrupt responses
      * Failure cascades
      * Such software failures often lie in wait to be triggered by an unusual set of circumstances, often caused by an underlying assumption that turns out to be False one day
   * Configuration errors from humans are the leading cause of error at internet services
      * Need to design software to make it easy to do the right thing
      * Users and developers will circumvent difficult interfaces to do unexpected things
      * Provide a fully-featured sandbox for testing, using real data if possible
      * Set up monitoring of performance and error rates
   * * Scalability
   * As the system grows (data and traffic volume, complexity) there should be reasonable ways to deal with that growth
   * Common cause of scale errors - more users and more traffic
   * Scale is not a ...scale
      * Describe Load in terms of parameters
      * Could be requests per/s, read/write ratios, active users
         * What about operations that fan out (pushing tweets to followers)
      * Describe Performance with respect to Load
         * What happens to service when Load is increased (with no changes)
         * What resources need to be increased to cope with new Load
         * eg. Records processed per/s, response times
            * Such metrics are a distribution, not a single number
            * Percentiles more informative than arithmetic mean
            * p95 (95th percentile) often used 
            * A small number of large requests can cause queuing - “head of line blocking” -- should measure response time from client perspective
            * Parallel requests can be slowed by just one part of the request (the slowest piece still has to finish) -- “tail latency amplification”
            * Don’t forget that munging the percentiles is mathematically meaningless, you need to munge the underlying distributions first
      * Likely you will need to rethink your architecture for each order of magnitude increase in Load
         * Scaling up (more powerful machines) and scaling out (multiple smaller machines)
         * Intense workloads often cannot avoid scaling out, as large single machines remain expensive
         * Elastic systems can flexibly change their resources with Load, can be automated if the Load is particularly unpredictable
         * Distributing stateless services across machines is quite easy, the difficulty lies in distributing stateful data
         * Architecture becomes increasingly specific to the application with scale; problem can be with reads, writes, volume, complexity, response times, access patterns…
      * Better systems have a good idea for which scaling concerns will be common or rare -- allowing for prioritisation
      * Often start-ups must prioritise development on features rather than hypothetical future load
* Maintainability
   * Developers maintaining current and future behaviour must be able to adapt the code productively
   * Goal is to create software that leaves a less painful legacy
   * Operability
      * Make it easy to run smoothly
      * A good ops team…
         *  maintains monitoring and can restore service from bad state
         * Tracks down problems, including degraded performance
         * Keep tabs on correlated systems
         * Capacity planning
         * Defines processes
         * Preserves system knowledge as people come and go
      * A good system …
         * Provides introspection on its internals for monitoring
         * Supports automation
         * Avoids depending on single machines
         * Offers easy operational models “if I do X, I know Y happens”
         * Provides good defaults but allows for overriding
         * Self heals but allows manual control
         * Minimises surprises
   * Simplicity
      * Make it easy for new engineers to understand the system
      * Avoid the big ball of mud that can happen as a system migrates from a small expressive system into something larger
      * Symptoms that contribute include...
         * Inconsistent naming and terminology
         * Coupling, tangled dependencies
         * Special casing (particularly painful in Majora…!)
      * Unexpected consequences caused by changes to a system that people do not fully understand
      * Accidental complexity (Moseley and Marks) arises from the system rather than the user’s problem (Majora models…?) as opposed to essential complexity http://curtclifton.net/papers/MoseleyMarks06a.pdf
      * Finding good abstractions is very hard
   *    * Evolvability (Plasticity, Modifiability)
      * Make it easy for new engineers to make changes to the system
      * Evolvability, a concept of agile at a large scale


### Data Models and Query Languages


* Data models are how we think about the problem we are solving
* Each layer of the system is hiding some complexity below, from the application right down to the magnetic field on a disk
* Different models for data can have drastically different performance for different operations


#### SQL and NoSQL
* SQL model Edgar Codd proposed only as theory in 1970, by mid 1980s was implemented efficiently enough, dominating for some 30 years 
* Relational DBs have seen off a lot of competitors and turned out to be surprisingly general
* NoSQL in 2010s (“NoSQL” is merely a catchy hashtag, nowadays thought to mean Not Only SQL)
   * A need for greater scale (large writes)
   * Specialised queries not supported by the more generic relationship model
   * Rigidity of relational model
* Likely that “polyglot persistence” will be the norm for some time to come - using a mix of SQL and NoSQL approaches to serve different needs of the system
* “Impedance mismatch” between software models and SQL model, requiring a clunky ORM layer
* Some data structures work well as “documents” (eg. a CV), JSON can reduce the impedance between your models and the persistence layer (although JSON notably lacks a schema)
   * Document model has better “locality” than SQL - you can fetch related pieces of information with less effort (no need for multiple fetches or messy joins)
   * “Shredding” documents into multiple tables can be unnecessarily complicated
   * Document model falls over if you wish to normalise (eg. store an ID of a city, instead of a text string), few systems allow joins
   * Systems have a tendency to move towards interconnected data, making it hard for the document model to stay join-free
   * This problem is not even new (see IBM IMS)
   * Document databases are schemaless, no guarantee on what keys/values will be in a given document (“schema-on-read”)
* Relational model counters documents performance with support for joins and many-to-one and many-to-many relationships
   * Keeping denormalised data consistent is hard work in a document system
* Document is more flexible for schema changes, you can just update new fields on the first read. SQL approaches require a migration.
   * MySQL copies the entire table for ALTER which can be slow…
* Documents need to be small as the entire record is usually re-written on update, and the entire document must be fetched even if you need only one piece of information from it
* Locality is not limited to document databases
   * Multi-table index cluster tables (Oracle)
   * Column-family (Bigtable)
* Relational databases now have support for document fields like JSON
* The relational model introduced a new declarative way to access data
* MapReduce paradigm allows for scaled out queries
   * Pure functions (cannot call other functions or request more data) to map a document to some key and value, and reduce the values for a key
   * MongoDB offers a powerful “aggregation pipeline” model to do this declaratively
      * Which looks a bit like SQL in disguise…
* Lots of one-to-many relationships (trees) are good for documents


#### Graph databases 
* Graph databases come in when you have many, many-to-many relationships in the data
   * Vertices and edges can have heterogeneous meaning


* Two main ways to structure graph data
* Property graph
   * Vertex: uid, outgoing edges, incoming edges, key-values
   * Edge: uid, tail vertex, head vertex, label, key-values
   * Edges are schemaless (any vertex can connect to any other)
   * Can efficiently find connected vertices for a given vertex using the edges
   * Edges can be labelled for filtering (allowing one graph to have multiple purposes)
   * Cypher is a declarative language for graph databases
   * Cypher can follow an edge towards some filter 0 or more times, which is an awkward recursive query in SQL
* Triple store
   * Everything stored in a triple (subject, predicate, object); (Jim, likes, bananas)
   * The subject is a vertex in a graph
   * The object is either another vertex or a primitive type
   * Examples of formatting a triple store is Turtle (Notation3) can be converted into less readable formats with Apache Jena
   * Related; RDF (resource description framework) was intended for websites to publish metadata in the same format to make a “database of everything” -- the “semantic web”
   * RDF has the quirk in that everything is expressed in URIs, primarily to avoid clashes with other people’s definitions of things (cf. Sammy’s ontologies)
   * SPARQL is a query language for RDF triple stores
   * Datalog/Prolog pre-date all this stuff and is very powerful but difficult to integrate


#### Summary


* Document databases target self-contained data documents where relationships between documents are rare
* Graph databases target use cases where everything is connected to everything else
* Some problems require very specific databases (eg. sequence similarity) or approaches (eg. full text databases)


### Storage and Retrieval


* There are engines optimised for transactions and others for analysis
* Important tradeoff, indexes speed up reads but slow down writes


* Log-structured vs page-orientated


#### Hash based index


* Consider the most basic database, a key value store backed by a file
   * Appending to a file is generally very efficient so inserts are fast!
      * This concept of fast appending backs the log-structured engines
   * Select is poor, requiring an O(n) scan
      * We must derive additional Index structures from the primary data
      * Indexes slow down writes as the indices must be updated
   * Simplest index maps the key to the byte-offset of the file
   * Almost feasible (Bitcask) if the map can be held in RAM
      * Not too many keys, but many writes per key
      * Append structures are compacted to remove and prune duplicate keys
      * Deleting records requires a special “tombstone” record to signal the key to be removed entirely from the next compaction
      * Bitcask uses checksums to determine if a record is a partial write
      * Bitcask writes snapshots of the segments to disk to speed up recovery
   * Append only writes are faster than random writes
   * Keys must fit in memory
   * Range queries are linear (slow)
* Key-values are similar to hash maps


#### SSTables and LSM-Trees


* Sorted string table
   * Keeps the concept of the hash index but requires the keys are sorted in each segment
   * Merging segments is efficient, using mergesort
   * No longer need to keep all the keys in memory, just store the first key in each segment and the byte location of the start of the segment
      * If all the keys & values were the same size you could use a binary search
   * Segments can be compressed to reduce space and improve I/O
   * Implemented in memory (memtable) with red-black trees or AVL trees
      * “Red–black trees offer worst-case guarantees for insertion time, deletion time, and search time” [wiki]
      * “For lookup-intensive applications, AVL trees are faster than red–black trees because they are more strictly balanced” [wiki]
   * Store in any order, read back in sorted order
   * Once the memtable is too big, write it to disk
   * Maintain a log file of writes, only for restoring the memtable before its written to SSTable
   * * Log-structured merge tree (LSM)
   * Keeping a cascade of SSTables and merging them in the background
      * Continues to work well even if the data does not fit in memory
      * Range queries fast as its easy to scan keys
      * High write throughput as writes are sequential
   * This is how LevelDB and RocksDB work, designed to be embedded into applications
      * Levelled compaction
      * Key range split into smaller SSTables and older data moved to levels
   * Similar engines are in Cassandra and HBase
      * HBase size tiered compaction, Cassandra both
      * Size tiered compaction merges new small SSTables into old large ones
   * log-structured merge-tree (or LSM tree) is a data structure with performance characteristics that make it attractive for providing indexed access to files with high insert volume, such as transactional log data [wiki]
   * Most LSM trees used in practice employ multiple levels. Level 0 is kept in main memory, and might be represented using a tree. The on-disk data is organized into sorted runs of data. Each run contains data sorted by the index key. A run can be represented on disk as a single file, or alternatively as a collection of files with non-overlapping key ranges. To perform a query on a particular key to get its associated value, one must search in the Level 0 tree and also each run [wiki]
   * Bloom filter is often used to determine whether the key is in the database to speed up finding keys (saving unnecessary reads for non-existent keys)


#### B-Trees


* Most widely used indexing structure
   * Introduced in 1970 and ubiquitous by 1980
   * Standard indexing structure in almost all relational (and other) databases
   * Like SSTables, key-value pairs are sorted by key, otherwise totally different
   * SSTables break data into variable sequentially written segments, B-trees break data into fixed blocks (or pages) [traditionally 4KB] and read/write one page at a time
      * Arranged to match underlying disk hardware
   * Pages are identified with a memory address, and pages can refer to each-other
   * One page is the designated root, each successive child is responsible for a range of keys
      * The number of references from one page to a successive child page is the branching factor
         * Based on a few factors but typically several hundred
      * Adding a key looks for spare space in the page, if there isn’t any, the page is split into two half-full pages, and the parent page is updated with the new ranges
         * Deleting is much more involved as balance must be maintained
      * A B-tree with n keys is always O(log n) deep
         * A four level 4KB B-tree with branch factor of 500 can store 256TB
   * In contrast to LSM-trees, pages are re-written in place and the references to the page will always remain intact (unlike LSM which is append only with obsolete delete)
   * Inserts that cause splits are more complex as they require writing new half-full pages and updated parent pages. Runs the risk of orphaned pages.
      * WAL - write-ahead log (or “redo log”) in which B-tree operations are appended in a structure before the tree is updated
      * Allows a B-tree to be updated to consistency after a crash
      * Some implementations use copy-on-write instead of WAL, write all the new pages first, then updating the parents to point to the new pages (this is useful for concurrent control)
   * Latches (lightweight locks) are used to ensure that concurrent reads of the tree do not view pages in an inconsistent state
      * LSM approaches are easier as merging data is a background operation that does not interfere with reading
   * Keys can be abbreviated to save space, only need to store as much as is needed to maintain range boundaries. This increases the branching factor, decreasing levels.
      * Sometimes called a B+ tree (but is so common nobody does)
   * Can be slow to read key ranges sequentially as it is hard to maintain ordering of leaf pages on underlying disk (easier for LSM where segments are written in sorted order)
   * Pages may also point to their siblings and other relevant pages to reduce query time during scans


#### LSM and B-trees


* LSM is faster than B-tree for writes, B-tree is faster than LSM for reads
   * SSTables requires reading of several structures at various stages of compaction
   * Such benchmarks are sensitive to workload, however
* A B-tree must write everything at least twice, once to the WAL and then again to the page
* Write amplification causes one write to lead to multiple writes, and is a concern for SSD-based database storage as blocks can only be written a certain number of times
* In write-heavy applications, storage speed may be the bottleneck
   * LSM-trees generally (but not always) lead to less write amplification
   * LSM-trees sequentially write SSTables which leads to fewer disk seeks on magnet, unlike writing to dispersed pages on B-trees
* LSM-trees can be compressed better and are compacted to save space
* B-trees are fragmented by design (leaving space for inserts)
* LSM compaction can impact performance of read/writes, occasionally leading to worse performance at higher percentiles than B-trees which have more predictable performance
* Configuration of LSM compaction is important to ensure bandwidth between incoming writes and compaction is suitable to avoid runaway slow-compaction running out of disk space
* Locks can be directly attached to keys in a B-tree


#### More on indices


* Keys can point to actual values, or the location of data
   * The latter option points the key to a position in a heap file
      * Avoids duplicating data when multiple secondary indexes point at the same data; all the actual data is kept in one place
      * Updates are easy if the value is not larger, otherwise the data needs to be moved in the heap and all secondary indexes need to be updated (often by leaving a forwarding pointer behind)
   * Extra hop from index to heap can be too expensive
      * Clustered index stores the actual row with the index for speed
      * Covering index stores a subset of columns with the index (the index covers the query)
* Clustered and covering indexes improve read but require storage and write overhead
* Concatenated index combines values of several fields into one key
* Specialist multi-dimensional spatial indices are also a thing
   * Lat, long could be collapsed into one value with a space filling curve to improve 2D query performance on LSM or B-tree
   * Alternatively, R-trees can be used
   * Not just limited to location -- see HyperDex
* Full-text fuzzy searching
   * Lucene keeps an in-memory index with a finite state automaton over the characters in the keys (like a trie), which can be transformed into a Levenshtein automaton for searching with edit distances
   * MySQL LIKE operator is not full-text and uses the B-tree https://stackoverflow.com/questions/25171013/how-does-sql-like-actually-work/25171340


#### Disks and Memory


* We tolerate the performance problems with disks because the data is not lost with power loss and they are cheaper
* As RAM prices have fallen, databases have moved to in-memory such as Memcached
* Writing to disk has many operational advantages -- backups, inspection, analysing
* In-memory databases are often not faster by virtue of being in memory (most disk databases are cached in RAM if your OS has enough anyway)
   * Rather they are faster by avoiding the overhead of encoding data structures to be written to disk
* Redis offers simple versions of priority queues and sets which would be harder to achieve on disk
* Anti-caching allows in-memory databases to use more memory than available by evicting the least used information to disk


#### Transaction pattern and warehouses


* Transaction came from the early days of business processing
* Transactions need not be ACID (atomic, consistent, isolated, durable)
   * Transaction processing just means clients can make low-latency reads and writes as opposed to Batch processing
* Usage pattern
   * Online transaction processing (OLTP) -- looks up a small number of keys, writes a small number of rows based on user input
   * Online analytical processing (OLAP) -- business intelligence queries, scan a huge number of records, perhaps of a few columns and aggregate statistics rather than actually returning the rows
* SQL works well for OLTP and OLAP but a trend towards “data warehousing” for OLAP has emerged
   * Emerged partly because engineers didn’t want analysts touching the OLTP databases as the performance could be harmed by large OLAP style queries
   * Read-only copy of data in the OLTP system, copied by dump or continuous streams
   * Process of ingesting is called ETL (extract-transform-load)
   * Warehousing is typically an enterprise level activity
   * Benefit is the warehouse can be optimised for OLAP access without the expensive of damaging OLTP activities
* Warehouses are usually relational
   * OLTP and OLAP can look similar but trend is towards two different database engines with a common SQL interface
   * SQL-on-Hadoop is becoming a common contender with big name enterprise systems
      * Apache Hive, Spark SQL
      * Some based on Google Dremel
   * Amazon Redshift is a hosted version of ParAccel. MS SQL claims to support both OLTP and OLAP.
* Warehouses have less model diversity (no need to support business cases)
   * Star schema (dimensional modelling)
   * Central “fact” table capturing some activity of interest, such as sales
      * “Facts” are events that have been captured
      * Could be massive table
      * Fact table in the middle of a sprawl of dimension tables 
   * Fact rows have attributes or foreign keys to other “dimension tables”
      * Dimensions represent the who, what, where, when, how and why of the fact
   * Even date and time may be encoded in a dimension table to allow for other information (such as flagging public holidays, or sale events)
   * Star topology can turn into snowflake by normalising the dimension tables (eg. a brands table dimension from the product table)
* Tables are very wide in warehouses
   * Fact tables may have billions of rows and hundreds of columns
   * Dimension tables may have millions of rows and tens of columns
   * SELECT * queries are rarely needed in warehouse as queries are looking to answer some specific question, requiring only a few columns
* Column orientated storage takes advantage of the access pattern, rather than loading entire rows from disk and processing/filtering them in memory, the columns can be accessed directly
   * Columns are stored separately, in the same row-order
   * Columns are ripe for compression, too
      * Run-length encoding
   * Compression even helps with query -- load the bitmaps of the target values and apply the bitwise AND to the column
* Cassandra and HBase have “column families” inherited from BigTable
   * Still mostly row-orientated, all columns from a row stored together with a row key
* Scanning millions of rows not only requires care about the transfer from disk to memory, but also efficiently using the cache between the main memory and CPU cache
   * Avoiding branch mispredictions, CPU instruction bubbles and using SIMD
   * Compressed columns fit better in L1 cache and can be read in tight loops (no functions) rather than reading in rows and processing conditions
   * Bitwise operations can be vectorised to work on column chunks
* Sort order of column-orientated databases don’t just need to use the insert order
   * If many queries rely on date ranges it might make sense to order by date
   * Sorted columns also improves compression performance
* Compression and sorting makes writes to column-oriented store more difficult
   * Many implementations have LSM-trees for writes where elements are sorted in memory and eventually written to disk
   * Queries need to search both the memory and disk writes and combine
* Materialised aggregates keep up-to-date aggregates of raw data to prevent repetitive processing
   * Materialised view is a table-like object representing the result of a fixed query
   * A copy of the data itself (unlike a virtual view)
   * OLTP databases often do not use such views as they are expensive to maintain on upsert
* OLAP cube (or data cube) is a special materialised view
   * Grid of aggregates grouped by dimension, such as a grid of total sales by each product over time -- now you can use the data cube to get total sales across time periods or products (or both) by summing along rows and columns
   * Facts often have more than two dimensions
   * Certain queries are now very fast as the hard work is pre-computed -- no need to scan millions of rows, but the query is completely fixed
* OLAP handles fewer but much more demanding queries than OLTP
* Disk seek often the limiting factor in OLTP, disk bandwidth for OLAP
* OLTP is typically divided into log-structured or update-in-place approaches
   * LSM -- append only, no updates, only merging
   * B-tree -- fixed pages that are overwritten in whole
* Column orientated systems emerging as a means to efficiently encode data to reduce reading from the disk and taking advantage of CPU bandwidth


### Encoding and Evolution


* Whenever you want to send data to another process with which you don’t share memory, you need to encode it as a sequence of bytes
* Schema evolvability and maintaining forward/backward compat is import for system evolvability 
* Services need to support rolling upgrades where new versions are deployed alongside old ones (less downtime, less risk); important that messages are passed in a way that provides backward compat (new code can read old data) and forward compat (old code can read new data)
* Programming language encoding are restrictive and fail to provide compat
* Text formats (XML, JSON, CSV) serve a purpose but lack of schemas can be a hindrance
* Binary schema-drive formats allow for compact and efficient encoding with clearly defined compat, as well as providing tools for documentation and static-typed languages


#### Formats


* Applications inevitably change, so do the data they store
* Need to manage these changes in a way that maintains forward and backward compatibility as best as possible (users may not update clients, servers may have staged rollouts)
* Data typically stored in memory, or encoded for flight
* In memory to byte encoding is known as encoding, serialisation or marshalling, the reverse is called parsing, deserialisation or demarshalling
* Many languages have their own system for serialisation (encoding)
   * java.io.Serializable, python Pickle, ruby Marshal
   * However the implementation is tied to a specific language
   * Restoring data requires the ability to instantiate arbitrary classes which can be a security risk
   * Efficiency and versioning are almost afterthoughts in these standard libraries
      * Java’s serialisation is notoriously slow
* Instead, standard encodings such as JSON and XML are language agnostic, are widely known and widely supported but have their own problems
   * Encoding of numbers (vs. strings that contain numbers)
      * Twitter’s API returns tweet IDs twice (once as a JSON string and once as a JSON number) because JS cannot handle 64 bit integers used to identify tweets
      * Binary strings are often encoded as base64 as JSON and XML do not support this
      * Unicode
   * XML and JSON have schema support but often not used, especially for JSON
      * Handling of edge cases (eg. numbers/base64) is handled in encoding/parsing logic instead
   * CSV has no schema support at all, parsers are not standardized
   * CSV, XML and JSON remain popular as data interchange formats and that is not going away
* Internally, there is less pressure to use something easy to parse/share
   * JSON is less verbose than XML but can become large
      * MessagePack, BSON etc. are binary alternatives to reduce size
      * None are as widely adopted as plain text JSON 
      * Often the small improvement in payload size is not worth the reduction in readability
* Apache Thrift (FB) and Protobuf (Google)
   * Require a schema, fields are tagged with a number
   * Provides tools that read the schema into classes in many programming languages
      * Fields are tagged, saving space as keys do not need to be stored (they are in the schema)
   * Thrift has several encoding options, including CompactProtocol
      * Compacts field type and size tags, as well as numbers
   * Protobuf similar to Thrift CompactProtocol
   * Protobuf does not have an array type, just a repeated tag
   * Encoded records are essentially concatenations of encoded fields
   * Tags refer to fields, even the field name can change, so long as the tag is the same
   * Old code can simply ignore new tags (forward compat)
   * New code can read old tags (backward compat), so long as new fields are not required -- new fields need to be optional or have a default
* Apache Avro is different from Thrift and Protobuf, a subproject of Hadoop
   * Does not encode field data at all, the data is mapped onto the types in the schema to work out their length and decode, no tags
   * Requires the same schema to read as the one that was used to write... 
      * Interestingly, the reader’s schema and the writer’s schema only need to be compatible, rather than the exact same
      * Compat maintained by only adding and removing fields that have default values (or null -- but nulls as quirky in Avro)
   * Large data sets encode the writer’s schema as part of the file, database records will encode the writer schema version with the row, network connections negotiate a schema version with Avro RPC
   * Avro’s benefit is mostly down to better support for dynamically generated schemas
* Tools to generate code for deserializing are less useful in dynamically typed languages as there is no compile-time type checking, and the code is usually seen as an obstacle in just getting to the data
* These tools have a lot in common with ASN.1 (1984), used to define network protocols and is still used for some purposes (X.509 SSL)
* Binary encodings offer better compaction compared to binary JSON and the schemas are a valuable form of documentation, and offers code to check for type-compatibility at compile time for statically typed languages
* Schema evolution offers similar flexibility to schema-on-read JSON databases


#### Dataflow


* Data can flow through databases, service calls and messages
* Database
   * Process encodes query into some storage and reads it sometime in the future
   * Data outlives code
   * Your application changes, your schema changes, but your data could be years old
   * Care must be taken if old code reads from database, may lose (or overwrite) new fields that are not in the application model
   * Avro and Parquet useful formats for writing out archived database dumps (object container files) -- chapter 10
* Services
   * Servers expose API that offer clients a service
   * Service orientated architecture (SOA) now more often called microservices
   * Services are a bit like databases, allowing clients to submit and query data, but using an application-specific API that is predetermined by the application, rather than a query language
   * Goal of microservices is to make the application easier to change and maintain by making services independently deployable and evolvable 
      * Expectation is that old and new versions of clients (and servers!) will be running at the same time and have full back/forward compat
   * Web services are implemented with REST or SOAP
      * REST is a design philosophy, building on HTTP
         * Simple data formats
         * Using URLs for identifying resources
         * HTTP for cache control, auth and content types
         * Has been gaining popularity over SOAP
         * Definition formats such as OpenAPI/Swagger allow for code and docs generation
      * SOAP is an XML-based protocol for network API requests
         * Independent from HTTP and HTTP features
         * Complex (“sprawling”) set of standards
         * Web Services Description Language (WDSL) describes an API
         * SOAP relies heavily on tooling as the WDSL and SOAP messages are not human friendly
         * Still used in many large enterprises but generally smaller and newer services are RESTful
   * Web services are just a new fad in a long line of tech for communication over a network
      * EJB and RMI only work for Java
      * DCOM only works on Microsoft platforms
      * COBRA is too complex and doesnt support back/forward compat
      * All based on RPC (1970s)
   * RPC tries to make calling a remote service as simple as calling a function on your local application (location transparency)
      * Anticipate failed requests (retry)
      * Need a response to distinguish no result from a timeout
      * Idempotence required to ensure multiple requests do not cause trouble
      * Need to account for more variable latency
      * All parameters must be encoded (can’t just use pointers), becomes problematic when parameters are chunky objects
      * Datatypes may differ across languages (JS can’t cope with numbers greater than 2^53)
   * The appeal of REST is it doesn’t have to deal with much of this, as it doesn’t pretend to be anything like a local system call
   * Yet, RPC is still around and supported
      * gRPC is an RPC implementation using Protobuf
         * Also supports streams of requests and responses over time
      * REST is predominantly popular as it is so ubiquitously supported (and can be tested with something as basic as cURL)
      * RPC often used between services owned by the same organisation
   * In terms of evolvability, you can assume servers are updated first, and clients second
      * Only need backward compat on requests (old code needs default)
      * Forward compat on responses (old code ignores new fields)
   * No real consensus on managing API versions
      * RESTful often encode version in the URL, or API keys are versioned
* Messages
   * Async messages, somewhat between RPC and DB
      * Sends messages like RPC, brokered between another process like a DB
   * Message broker has a few benefits over RPC
      * Can act as a buffer for unavailable or overloaded clients
      * Can redeliver messages to crashed processes
      * Sender does not need to know routing information
      * Messages can be broadcast
      * Decouples senders and recipients 
   * However, this is usually a one-way system - a sender normally will not expect replies (and certainly doesn’t wait for one)
   * Previously dominated by enterprise level software like IBM WebSphere
      * Now, open source alternatives like RabbitMQ and Apache Kafka
   * Messages sent to topics and consumed by subscribers
   * Distributed actor model where single-thread applications communicate with each-other by passing messages
      * Akka, Orleans, Erlang OTP


## Distributed Data
* What happens if multiple machines are involved in storage and retrieval of data?
* Reasons to distribute data
   * Scale - spread the load of read/write across more than one machine
   * Fault tolerance/availability
   * Latency - serve requests from geographically closer locations to users
* Higher load can be achieved with vertical scaling (bigger boi), multiple machines can be joined together virtually as one machine (shared memory approach -- cost is non-linear)
   * Care must be taken on NUMA (non-uniform memory arch) where some memory is closer to some CPUs, so processes must be partitioned to avoid performance penalties - even on the same machine
* Or, shared-disk approach -- different machines with shared storage (NAS/SAN)
   * Storage contention and locking limits scaling with shared-disk
* Shared nothing architecture (horizontal scaling) is increasing in popularity, compute nodes are co-ordinated via a network
   * Pick machines with best perf/price ratio; distribute data and reduce latency
   * Requires the most care from developers
   * Most flexible but incurs the biggest penalty for development
* Two common ways to use the shared nothing arch
   * Replication -- keep a copy of the data, can improve performance and reliability
   * Partitioning -- split data into smaller subsets (sharding) for performance
   * Or both


### Replication


* Increase throughput, availability and reduce latency
* Difficulty of replication lies in handling changes in the data
* Single-leader
   * Read scaling, many balanced followers for high read rate, low write rate 
* Multi-leader
* Leaderless
* “Eventual consistency”
   * Stop writing to the database and async followers will eventually catch up


#### Leaders and followers


* A node storing a copy of the database is a replica
* Every write must be stored by every replica
* Leader-based replication (active/passive, master/slave)
* Primary (leader, master) accepts writes and transmits them through a replication log/change stream to followers (secondaries, read replicas, slaves)
* Writes are primary only, reads can be served by any node
* Built in to most database engines (and some messaging brokers too)
* Sync vs. async is an important detail
   * Sync waits for writes to be successful on followers
      * Guaranteed consistency
      * Any sync follower can block writes to the leader
      * Usually one follower is synchronous and the others are async -- “semi-sync”
   * Async does not block        
* Setting up a follower normally involves
   * Snapshotting the leader (innobackupex for MySQL InnoDB)
   * Follower connects to the leader and requests changes since the snapshot, using the log sequence number (Postgres), binlog coordinate (MySQL)
* Follower failure - catch-up recovery (process the write log and catch up)
* Leader failure - failover to sync’d follower (or most sync’d one at least)
   * Gotcha - need to make sure if the leader comes back, it knows it has been forced to step down and act as a follower
   * Can become especially messy with automatically assigned primary keys
   * Split-brain can occur when two leaders accidentally emerge
   * Need to balance timeout period between waiting too long (damaging availability) and too short (triggering unnecessary failover)


#### Replication logs


* Statement based replication
   * Forwards INSERT, UPDATE and DELETE statements to followers
   * Non-deterministic functions like NOW or RAND will generate different values on replicas
   * Autoincrementing statements must execute in exactly the same order
   * Non-deterministic side effects (triggering procedures)
   * MySQL used this until v5.1, and uses row-based replication for non-determinism
* WAL shipping
   * LSM engines use append-only log
   * B-trees engines use write ahead log
   * Postgres and Oracle use WAL
   * Downside is the log is expressed in terms of the underlying storage (which bytes and blocks were changed), requires database software versions to be strictly managed to ensure they read/write WAL the same way
      * This is important as it means rolling releases might not be possible
* Logical / row log replication
   * Similar to WAL shipping but is less coupled to the database engine
   * Contains just enough information to insert new rows, locate and update (or delete) existing rows
   * MySQL’s binlog uses this approach
   * Additionally useful for “change data capture”
* Trigger based replication
   * Application level replication rather than at the database itself
   * Might need to replicate specific data to specific node, or only replicate subsets etc.
   * Triggers allow for custom application code to be executed on write, which can write data to another table to be read by external processes (Oracle Databus, Postgres Bucardo)
   * Less reliable that database level replication but powerful


#### Replication Lag


* Read-after-write consistency (read-your-writes) guarantees that a user will see the result of THEIR writes (there are no guarantees about the writes of OTHER users)
   * Read modified data from the leader (need an idea of what could be modified)
   * Read from the leader where the update time is in the last minute
   * Clients can remember their most recent write and ensure they only make queries to followers that are sufficiently up to date (logical timestamp rather than actual timestamp will circumvent unreliable clocks)
   * How to do this over multiple devices?
*  Monotonic reads
   * Reading from a recent follower then a stale follower, time travel
   * Lesser guarantee than strong consistency but better than eventual consistency
   * Send user reads to same replica (based on user ID hash or some such)
* Consistent prefix reads
   * If writes are not sync’d in order, observers may see inconsistent relationships between data (a twitter user having a conversation with themselves or appearing to answer questions ahead of them being asked)
* Several minutes of lag may be no problem to some applications, otherwise you will need to guarantee the consistency (pretending async consistency is sync is going to fail one day)
   * Eventual consistency is somewhat inevitable compromise in distributed systems (many argue that transactions are too expensive to distribute in terms of performance) -- a little simplistic


#### Multi-Leader Replication (active/active)


* If you can’t connect to the (partition) leader, you can’t write to the database (or partition)
* Active/active
* Multiple datacenter operation (leader in each DC)
* Can geographically spread leaders for lower latency and tolerance of DC loss
* Tungsten for MySQL, BDR for Postgres
   * Tungsten doesn’t try to detect conflicts
   * BDR does not provide causal ordering of writes
* Requires conflict resolution systems for merging writes
   * Sync vs. async conflict detection -- if you want sync you may as well be using single-leader as the main benefit is accepting writes quickly (not blocking)
   * Avoiding conflicts at all is a good approach (leaders are responsible for a given set of users or records)
      * From the user’s point of view the system is single-leader
      * Breaks down if you need to failover
   * Conflicts must resolve in a convergent way (all replicas arrive at the same value eventually)
      * Break ties by giving all writes an ID, timestamp or hash and pick the highest (tossing conflicting writes) - LWW (last write wins)
         *  Prone to data loss
      * Merge values together (if this makes sense)
      * Record the conflict explicitly and present it to be resolved by the user
* Generally avoided as support is often retrofitted (causing trouble with autoincrementing keys, triggers, constraints)
* Clients with offline operations are effectively multi-leader (each device is its own leader, syncing the other devices)
* Collaborative editing has many things in common with this problem
* Custom conflict resolution
   * On write -- database calls a conflict handler script
   * On read -- conflicts are stored and the user is prompted to make a decision
* Automated conflict resolution (complicated)
   * Conflict-free replicated datatypes (CRDT) -- 2 way merges
   * Mergeable persistent data structures -- explicit history tracking and 3 way merges
   * Operational transformation -- Etherpad/GDocs, concurrent editing of ordered lists (eg. characters of a document)
* Multi-leader topology
   * MySQL only supports circular topology by default
   * Writes may need to pass through multiple nodes to reach all replicas
   * Hard to mitigate failure as the path for sharing writes is blocked


#### Leaderless


* Re-popularised by Amazon’s internal Dynamo system
   * Riak, Cassandra and Voldemort use “Dynamo-style” leaderless
* Clients directly send writes to other replicas, or to a controller node (which is not a leader as it does not make any guarantees to write ordering)
* Quorums for reading/writing
   * Send update to n replicas and assume its OK if w reply
   * Read most recent data as reported by r replicas
   * R + W > N
      * N usually odd, with r=w=ceil((n+1)/2)
   * Can be tricky to ensure all reads and writes are quorate, especially if tweaking w+r <= n for performance purposes
* Dynamo-style leaderless
   * Read repair -- multiple replicas reply to a read, stale data is detected and overwritten (works best when all data is read frequently)
   * Anti-entropy -- background process looks for disagreements between replicas (ensures data that is not read often is also replicated correctly)
* Harder to monitor staleness as there is no log for comparisons
* Good for high availability if you can tolerate some stale reads
* Sloppy quorum if you can’t reach all n nodes
   * Hinted handoff -- Once the disruption is over, the temporary nodes write to the “home” nodes 
   * Good for write availability, allowing the database to write if any w nodes are available to a client (rather than w nodes inside the set of usual nodes for some data)
   * Violates w+r > n for reading as you have written to some nodes outside n
* Cassandra and Voldemort also works for quorums per datacentre, Riak achieves multi datacentre through a multi-leader-like background process


#### Last write wins (again) and conflict resolution (again)


* Assume there is a canonical latest record and that replicas can overwrite or discard older values
* May have to enforce an arbitrary order
* Efficient but comes at the cost of durability
   * Even successful writes will be lost if they are replaced by LWW
* Two operations are concurrent if they have no ordering
* Only concurrent updates need conflict resolving (because causal dependent operations can be ordered and the newer replaces the older)
* To avoid data loss, clients have to do the heavy lifting:
* Stamp rows with version number
   * Upload new data with last known version number (read before write)
   * Distinguishes between overwrites and concurrent writes
   * Merge and take the union of concurrent siblings 
   * Deletions need to be marked with a tombstone to prevent deleted items getting added back to union
   * Riak uses CRDTs to try and aid with merging siblings
   * Scaling this to leaderless requires each replica to keep a version number too -- forming a version vector of versions across all replicas that must be merged as siblings
      * Riak 2 uses “dotted version vector” (called causal context)


### Partitioning


* For very high throughput (or very large data) we must break the dataset into partitions
   * Aka: shards (mongodb, elastisearch, solar), regions (hbase), tablet (bigtable), vnode (Riak, Cassandra), vbucket (couchbase)
* Each record (row, document) belongs to exactly one partition
* Each partition is a database of its own, though an operation may touch multiple partitions
* Main goal is scalability
   * Used when storing and processing a dataset on a single node is no longer feasible
* Recently “rediscovered” by NoSQL and Hadoop-warehouse databases
* Everything from Replication applies, usually partitions are replicated across nodes
   * Node may be a leader for one (or more) partitions and a follower of others
* Skew makes partitioning less fair, ideally 10 times more nodes should deliver 10 times more throughput
   * A node with disproportionate load is a hot spot


#### Key-value partitioning


* Simplest approach to avoid skew is to assign records to a partition randomly
   * But need to keep track of which records are on which partitions!
   * Use ranges (like picking the right volume of an encyclopedia) and adapt the boundaries to distribute the data (some volumes will cover a bigger range)
      * Used by BigTable/HBase, RethinkDB, MongoDB (<v2.4)
   * Need to be clever with the key name, if the key is time based then data access will be uneven (eg. writing all of today’s data to the same partition), prefix it with a node or sensor name to distribute it somewhere else
* Many databases use a hash of the key to avoid skew and hot spots
   * Takes skewed key set and makes it even
   * Need not by cryptographically strong
      * MD5 (MongoDB), Murmur3 (Cassandra), Fowler-Noll-Vo (Voldemort)
   * Partitions are assigned a range of hashes rather than a range of keys
   * Loses range queries
      * MongoDB sends request to all partitions to calculate range queries
      * Riak, Couchbase and Voldemort don’t support primary key ranges
      * Cassandra uses a compound key that offers a middleground
   * Particularly hot keys will also have hot hashes, a random number appended to the key can further shard the data (but will need to be collected again to carry out queries)


#### Secondary index partitioning


* Problem with secondary indexes is they can’t map nicely to partitions
   * Avoided by HBase and Voldemort
      * Can implement this at an application level instead (map the secondary search terms to primary keys, but need extreme care to ensure consistency)
   * Minimal support in Riak
   * The “raison d’etre” of Solr and Elasticsearch
* Document-based partitioning
   * Adds primary key to list of “documents” for a given term to use an index
      * Eg. all partitions have a list of red cars and silver cars
   * Index is maintained at partition level (local index), so needs to be collected and merged across all partitions (scatter/gather)
   * Queries prone to tail latency amplification as all partitions must respond
   * Widely used in Mongo, Riak, Cassandra, Elastisearch, Solr, VoltDB
* Term-based partitioning
   * Global index for a given search term is partitioned
      * Eg. one partition has a list of red cars, another has silver cars
   * Term can be partitioned by name or hash, just like the keys
   * Reads are more efficient (no scatter/gather), however writes are made more complicated as terms on multiple partitions will need to be updated
   * Usually updated async


#### Rebalancing partitions


* You will want to add more CPUs to cope with load, or the dataset increases and you need more disk/RAM, or a machine fails and you need to plug the gap
* Moving data from one node to another is called rebalancing
   * Nothing more than necessary should move to keep disk/network I/O as low as possible
   * The database must still accept reads and writes for availability
* Do not assign partitions to nodes with hash % N
   * If N changes, most of the data will need to be moved in a rebalance
   * Makes rebalancing excessively expensive
* Fixed number -- assign more partitions than nodes and randomly assign them
   * New nodes can “steal” partitions from existing nodes and share them out
   * The only thing changing is the partitions:nodes ratio
   * You can also assign bigger nodes more partitions to account for resources
   * Used by Riak, Elastisearch, Couchbase, Voldemort
   * Must balance -- low enough to avoid overhead, high enough to account for future growth (as its hard if not impossible to change the number of partitions later)
* Dynamic -- if you got the boundaries wrong in a fixed setup it would be a disaster
   * HBase and RethinkDB automatically select partitions
   * Number of partitions adapts to data volume
   * Large partitions are automatically split in half and moved to different nodes
   * Pre-splitting conducted (if you know what your key distribution looks like) with Mongo and HBase so new databases don’t sit on one partition waiting to be split
* Fixed number per node 
   * Cassandra and Ketama
   * Adding more nodes makes the partitions smaller
* Automation
   * Couchbase, Riak, Voldemort generate assignments but require manual commit
   * Rebalancing is expensive and unpredictable
      * Potential for cascading failure if combined with inaccurate monitoring 
* Dynamic usually used for key-range partitions, fixed for hash-range


#### Request routing


* Instance of a more general problem called service discovery
* Several solutions
   * Use any node in round-robin
      * Node directly serves or forward the request to the right partition
      * “Gossip protocol” used by Cassandra and Riak
   * Routing tier
      * Tier forwards the request to the right node
   * Client routing
      * Client must be aware of partition layout and contact the right node
* Many systems use another service to keep track of this metadata
   * Apache Zookeeper
      * Used by HBase, Solr and Kafka
      * Mongo similar but uses its own config server
   * Nodes register themselves with Zookeeper which maintains the node mapping
   * Routing tier (or clients) can subscribe to updates from Zookeeper


### Transactions


* Transactions are the tool of choice to simplify 
   * Database software, hardware, application or connection can fail at any time
   * Several clients may write over eachothers data concurrently
   * Race conditions between clients
* Generally follow the model introduced in 1975 by IBM System R
   * NoSQL databases emerged, providing weaker guarantees of transactions
   * Transactions have unfairly earned a bit of a bad name
* Group of reads and writes is one batch that succeeds (is committed) or fails (and is rolled back)
   * Applications do not need to worry about partial failure
   * Safety guarantees that the application no longer worries about
* Not every application needs such safety and may not use transactions to improve performance or availability
* 

#### ACID


* Safety guarantees of transactions are often described by ACID
* Atomicity
   * Cannot be broken into more parts
   * If one of several writes fail, the atomic unit cannot be split and fails as a whole
   * Abortability might have been a better word than atomicity
   * All-or-nothing guarantee
* Consistency
   * There exist invariants that must be true
   * Onus is on the Application to define transactions that maintain the consistency of invariants
   * Not really a component of the database and some argue is only here to make the acronym work...
* Isolation
   * Concurrently executing transactions are carried out in isolation to avoid race conditions (eg. the double counter increment)
   * Classically formularised as “serializability”
      * Database handles the concurrent transactions as if they were in serial
      * Every transaction can pretend it is the ONLY transaction running on the database
   * Serializable isolation is often too much of a performance penalty
      * Weaker isolation levels are often used in practice
   * Other transactions should see all (or no) writes from another transaction
* Durability
   * Promises that the result of any successful transaction will be stored
   * Historically this meant written to archive tape!
* “Devil is in the detail” and one ACID compliant system does not equal another
   * Particularly different interpretations of Isolation
* BASE is “not ACID” -- basically available, soft state, eventual consistency
   * “Good enough for big sharded datastores”, “ACID for noSQL” (Tom)
* Multi-object (ie. writes over multiple rows/tables) need a way to determine writes in the same transaction
   * Can be done with a unique transaction ID
   * Or from the client TCP connection (less ideal)
   * Many non-relational databases don’t have a way to group writes
* Still also applicable to single-object write
   * Network disruption writing a large JSON blob
   * Storage engines almost universally handle this case
   * Atomicity provided with logs
   * Isolation provided with locks
   * Sometimes called ACID but really just marketing trickery
* Databases provide some atomic (serializable) operations such as atomic (as in multi-thread safe) increment to avoid read-modify-write cycles on counters
* Distributed datastores have abandoned multi-object transactions as they are difficult to implement over partitions and can get in the way of performance
* Leaderless replication will often violate atomicity and not undo commits already made
* Retrying a transaction can be risky
   * Need to ensure the application has a way to know the transaction succeeded, so it does not try it again (although ideally requests should be idempotent anyway)
   * If it failed due to a load problem, retries will make the load worse (need exponential backoff and retry)
   * Need a way to determine whether the error was transient or permanent (ie. constraint errors or the like)
   * Side-effects outside the transaction (eg. sending an email) need to be correctly orchestrated -- two-phase commits can help


#### Weak isolation


* Concurrency is hard -- hard to test (requires being unlucky with timing), hard to reason about, any piece of data in a multi-user system can change at any time!
   * Serializable isolation is too much of a performance penalty for most databases
   * Weaker isolation tends to be used instead, which leads to subtle bugs
   * Many “ACID databases” use weak isolation, so watch out!
* Read committed
   * Default in Oracle 11g, Postgres, SQL Server 2012, MemSQL and others
   * When reading, you will only see committed data (no dirty reads)
      * Row locks are impractical as long-running transactions can stop application for reading data
         * Database engine just remembers the old and new values, returning the old value for reads until a transaction is committed
         * Other than IBM DB2 and MSSQL (read_committed_snapshot=off)
      * If a database allows dirty reads, a transaction may view data the data of another aborted transaction before it is rolled back
   * When writing, you will only overwrite committed data (no dirty writes)
      * Transaction must wait for row-level lock before writing
      * Prevents two transactions getting mangled together (eg. the buyer and invoice tables)
      * Does not prevent double counter increment
   * Some databases provide a “read uncommitted” isolation level that allows dirty reads
* Snapshot isolation and repeatable read
   * Read committed appears to do everything needed for a transaction
   * Does not guard against read skew
      * Select balance on two accounts where a transaction happens between the two reads
      * Not a lasting problem but can have consequences
      * Backups, queries and integrity checks are examples of where you want to read the data at a specific point in time, rather than observing rows in different states
   * Snapshot isolation is the common resolution
      * Transactions read from a consistent snapshot
      * If another transaction changes the database during the first transaction, it still sees the original data
   * Good for long-running analysis and backup queries
   * Write locks, all uncommitted reads are stored -- MVCC (multi-version concurrency control)
      * At the start of a transaction, all other transactions in progress are enumerated, their writes are ignored for the duration
      * Aborted transactions are ignored
      * Future transactions are ignored
   * Indexes
      * Postgres has an optimisation where indexes stay the same if multiple versions can fit in the same page
      * Append only B-trees create a new root for each transaction, a given root is consistent for a point in time, old trees are garbage collected
   * Snapshot isolation useful in particular for heavy and long-running read queries
   * Be aware of some naming confusion around “repeatable read”


#### Preventing lost updates


* Read committed and snapshot isolation concerned with what a read-only transaction can see while concurrent-writes are completed, neither covers concurrent write problems
* Best known write-write conflict is lost update (the double counter increment)
   * Two concurrent read-modify-write cycles can lose the modification of one write
   * The latter write clobbers the first write
* Atomic write operations are the best choice
   * Concurrency safe:
      * UPDATE x SET value = value + 1 WHERE y
   * Some document database support editing of part of a document
   * Redis provides atomic operations for data structures
   * Takes an exclusive lock on the object when READ
      * Sometimes called cursor stability
   * Easy to use an ORM to accidentally circumvent atomic writes provided by the database
* Explicit locking
   * If the operation you want to perform is not atomic in your database engine
   * Explicitly lock the objects to be updated
      * Forces a second transaction to wait for the read-modify-write cycle of the first
   * Multiple users moving a character in a game cannot be handled with database level atomic operations as the move will require logic first
   * SELECT FOR UPDATE
   * Easy to forget to add a lock
* Detecting lost updates
   * Allow the concurrent write to execute and catch it 
   * Postgres repeatable read, Oracle’s serializable and SQL snapshot isolation automatically detect and abort on lost update
   * MySQL InnoDB repeatable read engine does not catch lost updates
      * Arguably MySQL does not have snapshot isolation
* Compare and set
   * For databases that cannot provide transactions
   * Allows an update iff the row has not changed since it was read
   * UPDATE x SET value = newvalue WHERE value = oldvalue
   * Care to be taken that you’re reading from the real source (not a snapshot) otherwise the UPDATE may go ahead without value=oldvalue
* Multi-leader or leaderless
   * Locks and compare-and-set don’t work -- there is no one source of data
   * Concurrent writes are allowed to create conflicting “siblings”, and the application (or special data structures) take care of merging and resolving the conflict
   * Atomic operations that are commutative (can be applied in any order on different replicas) are viable (eg. add to set, increment counter)
      * Riak2 specifically offers data structures for this
   * Last write wins is the default with most replicated databases and is prone to loss


##### Write skew


* Two users update a record at the same time, taking themselves off a register
   * Both transactions read from snapshot isolation level, so the invariant is true
   * Both transactions succeed and are committed at the same time and the invariant is false
* Write skew is a generalisation of lost update
   * If two transactions read the same object then update different objects you have write skew
   * If two transactions update the same object, you get a dirty write, or lost update (depending on the timing)
* Application “phantom” (1976)
   * SELECT checks for a constraint
   * Application goes ahead with some operation based on SELECT
   * The UPDATE changes the pre-condition
   * Phantom is where one transaction affects the SELECT of another transaction
* Options more restricted to prevent write skew
   * Atomic single-ops don’t work as they cover multiple objects
   * Detecting write skew requires true serializability which is not the default for most levels of isolation
   * Best option is to use SELECT FOR UPDATE and lock any rows involved in the transaction
   * Only works if there is something to SELECT to lock (ie. if you are checking something can be modified, rather than inserted)
   * Materialized conflicts involve creating dummy tables and rows that can be SELECTED for locking (but do not actually store anything)
      * Pretty ugly as your concurrency control is leaking into the application
      * Last resort
      * Serializable usually preferable
   * Serializable is the only true solution
* Snapshot isolation works for read-only phantoms, but not read-write (as you may read an invariant as true based off old data)


##### Serializability


* The only true solution, promises that if transactions behave when run individually, they will also behave concurrently -- the database prevents race conditions
* Actual serial
   * Completely sidesteps the problem of conflicts
   * Only actually recently became feasible (c.2007)
      * RAM became cheap (data loaded from memory not disk)
      * OLTP transactions are short and OLAP are long (read-only OLAP can be executed on a snapshot in the background)
   * Transactions have to be written differently
      * Interactive multi-statement transactions not possible
      * Cannot lose time waiting for users to make decisions (eg. SELECT followed by UPDATE) -- time lost in network
      * Transactions instead defined as stored procedures that are called
   * Stored procedures have a bit of a bad rep
      * Each DB has its own language for them and are usually archaic
         * Not always true, Redis uses Lua for example
      * More awkward to manage and debug as its not in the application
      * Can cause performance trouble for other applications using the DB
   * Can be a serious bottleneck for high throughput write applications
      * Writes are limited to speed of a single CPU core
      * Read only transactions can execute elsewhere with snapshot isolation
   * If you can partition your data such that reads/writes do not need to span partitions, then each partition can have its own isolated thread for its own writes (VoltDB)
      * Cross-partition procedures must execute in lock-step which adds huge overhead
   * Every transaction must be slow and fast
      * A transaction that needs data from disk is better aborted and restarted once the data is fetched to memory in an async process (anti-caching) 
* Two phase locking (2PL), strong-strict two phase locking (SS2PL)
   * Not to be confused with two phase commit (2PC)
   * Widely used algorithm for serializability
   * Stronger lock requirements
   * Several transactions are allowed to read the same object as long as it is not being updated, otherwise exclusive locks are required
      * If A has read an object and B wants to write to it, it must wait for A
      * If A has written an object and B wants to read it, it must wait for A
      * Writers don’t just block other writers, they block readers (unlike snapshot isolation where readers never block writers and writers never block readers)
   * 2PL protects against lost updates and write skew
   * Used by MySQL InnoDB serializable isolation level
   * Readers lock the object in shared mode, writers must wait to obtain an exclusive lock. A reader can become a reader during a transaction but must wait for an exclusive lock.
   * Lock is held until the end of the transaction (two phase: query execution and transaction end)
   * Deadlocks can occur if A waits for B, and B waits for A
      * Database detects and randomly kills a transaction in deadlock
   * Significantly worse performance than weak isolation
      * Primarily from overhead of acquiring and releasing locks on every row
      * Reduces concurrency, by design a transaction must wait to read anything being written
      * Queues can form to access particular objects
      * Queries that access many objects can cause instability
      * Slow at high percentiles
   * Types of locks
      * Predicate lock
         * Locks a set of rows matching a predicate
         * A must acquire a shared mode predicate lock on all objects that match its query, and must wait if B has an exclusive lock on any one of those objects
         * If A wants to insert, update or delete, it must check whether the old or new value matches a predicate lock and wait
         * Predicate lock applies to objects that don’t exist yet, covering the case of read-write phantoms
      * Index locks
         * Predicate locks do not perform well
         * Most 2PL implementations use index locking (next-key locking)
         * Uses approximations to lock a wider set of objects
            * Eg. instead of locking the predicate Room 123 between 12pm 1pm, you just lock all of Room 123, or all rooms between 12pm 1pm
         * Approximation of the search condition
         * Lock more objects but have lower overheads
         * Effective protection for phantoms and write-skew
         * Can fallback to locking the entire table if there is not a sufficient index, which can be terrible for performance
* Optimistic concurrency control (serializable snapshot isolation)
   * Concurrency control looks a bit bleak; 2PL is too slow and true serial doesn’t scale, weak isolation leaves us open to lost updates, write skew and phantoms
   * SSI first described in 2008 but has potential to be default in many engines in future
   * 2PL is pessimistic as it assumes anything could go wrong, preventing any situation that could cause a conflict (catching many non-conflicting situations), serial is extremely so
   * SSI is optimistic (optimistic concurrency is not a new idea) and allows transactions to occur concurrently and checks if anything bad happened afterward
   * Previously performed badly as transactions needed many retries (due to aborts) in systems with high contention for object access
   * SSI takes old idea and adds snapshot isolation and a means to detect conflicting writes
   * SSI aims to detect when a value used as a premise (for doing something in the application) is outdated
      * Detecting reads of a stale MVCC (multi-version concurrency control) object version (uncommitted write before read)
         * Snapshot isolation ignores uncommitted writes when transaction has started
         * A will update an object that B reads (before A has committed), but the premise for B’s transaction may longer be correct
         * SSI keeps track of writes that have been ignored during the current transaction, if they are committed before the end of the current write transaction, the current is aborted
         * Avoids unnecessary aborts (the second transaction may just be read-only, or the first might fail)
      * Detecting writes that affect prior reads (committed write after read)
         * Uses similar mechanism to index locks, but without blocking
         * When a transaction is written, it checks the index “locks” to see if another open transaction has read the object -- acts as a tripwire instead of a block
   * Granularity of tracking level and abort rate determines the performance
   * Big advantage compared to 2PL is the lack of blocking waiting for locks, especially appealing for read-heavy loads which do not require locks at all (read from consistent snapshots)


##### Summary
* Dirty reads -- one client reads the writes of another before they are committed, read committed isolation level protects
* Dirty writes -- one client overwrites the data of another before it is committed, almost all transaction implementations prevent this
* Read skew -- a client sees different parts of the database at different points in time, prevented with snapshot isolation
* Lost updates -- two clients before read-modify-write, one overwrites the other without incorporating the changes of the other (the change is lost); some snapshot isolation implementations guard against this, others require locks (SELECT FOR UPDATE)
* Write skew -- A transaction reads something, makes a decision and writes to the database. However between the read and write, the premise became false. Only serializable isolation prevents write skew
* Phantom reads -- A transaction reads objects that match a search condition, another client makes a write that affects the result of the search. Snapshot isolation prevents read-only phantoms, but read-write (write skew) phantoms require index range locks or other treatment


### The trouble with distributed systems


* Time to dial up the pessimism and think about “the harsh winds of reality”.
* Distributed systems offer lots of new and exciting ways for things to go wrong
* A node cannot know anything for sure, it can only make guesses based on received messages. Can only know about other nodes state/store by exchanging messages over an unreliable, variable network.
* Our job is to build applications that do what the user wants despite everything going wrong
* Computers hide the fuzzy physical reality of their implementation and provide a model system that operates to mathematical perfection; if an internal fault occurs the system crashes completely to avoid returning confusing bad data
   * A distributed system does not operate on this model, and we must confront the messy reality of the real world
   * Partial failures are possible
   * Distributed systems behave nondeterministically in partial failures
* HPC supercomputers try to behave more like a single computer (eg. jobs will be stopped if one node fails), escalating a partial failure to total failure if necessary
   * HPC nodes are in specialised network topologies such as Torus to ensure reliability for particular usage patterns
   * Internet services cannot work like this, they must be online all the time!
* Must accept the possibility of partial failure and build fault-tolerant software
   * Unwise to assume faults are rare and hope for the best
   * “Suspicion, pessimism and paranoia pays off”
   * Although safe to assume faults are non-Byzantine (ie. your systems arent lying)
* Unreliable network, unreliable clocks, process pauses can all cause partial failures
* Faults must be tolerated by first detecting them
   * Nodes can’t even agree what time it is, so quorums allow nodes to come to a consensus on failures and mitigations
* Hard real-time guarantees and bounded delays are possible, but too expensive and have lower utilisation efficiency, making them poor choices for non safety-critical operations


#### Unreliable networks


* If you send a request and expect a response, many things can go wrong
   * Request lost (unplugged network cable)
   * Request waiting (stuck in a switch queue)
   * Remote node crashed with your request on it
   * Remote node busy and isn’t responding (yet)
   * Remote node processes your request but the response is lost
   * Remote node processes your request but will tell you later
* In an asynchronous packet network it is not possible for the sender to distinguish what has failed, the only information is you have not received a reply
   * Even with timeouts, you still don’t know if the remote node got the request or not
   * Unbounded delay means there is no upper limit on how long a packet can take to be delivered
   * Unlike synchronous networks (eg. telephone, ISDN, where bandwidth is reserved end-to-end for the duration of the connection), no queueing, bounded delay
   * Asynchronous packet switching networks best for burst traffic (and less wasteful), other than the odd keepalive packet, a TCP connection uses no data when idle
      * Asynchronous transfer mode (ATM) was a competitor to Ethernet in the 1980s but didn’t really catch on outside of telephone core switches
* Redundant networking gear doesn’t help so much as the main reason for failure is configuration changes
* Handling faults does not mean tolerating them
   * You can just report messages to users saying you’re out of service
   * Regardless of what you do, you need to be sure your software will do the right thing
   * Makes sense to trigger these circumstances to make sure
* Hard to tell whether a node is working or not
   * If you can reach the machine but the process is dead, the OS will return `RST` or `FIN` in its reply, but you have no idea how much data of your request it did process
   * The node operator may provide a mechanism to alert other nodes of the crash (HBase works like this)
   * Routers may respond with ICMP Destination Unreachable
   * Even if TCP reports the packet was delivered, the application may crash before reading it
* Networks are like roads, they can get congested and travel times can be variable
   * Nodes all sending packets to the same destination can clog up the network as packets can only be pushed one-by-one to the recipient
   * Once read off the wire, the destination’s CPU will queue the data until the application can handle it
   * VMs will be paused while other VMs access the CPU core, during this time the paused VM cannot access the network (VM monitor must buffer the packets)
   * TCP congestion avoidance (backpressure) will queue packers at the sender to prevent overloading a busy receiver 
   * TCP considers a packet lost if a reply is not received before a timeout, requiring the timeout delay + retransmission time
   * These problems worsen the closer the system is to capacity
      * Noisy neighbours in a data centre can saturate links even if you don’t
* UDP is good choice for cases where delayed data is worthless
* You can only trust replies from the application itself
* Dynamically allocating packets on a wire allows for cheaper per-byte transfer than static partitioning (the wire has a fixed cost and you can use it better). This is the significant motivation for VMs (using CPUs more efficiently).
   * Dynamic resource partitioning affords better utilisation but has variable delays
   * Delays in networks are not a law of nature, but a cost/benefit trade off
* There is no correct value for timeouts, they need to be determined experimentally
   * Phi Accrual Failure Detector (used in Akka and Cassandra) can monitor response time and jitter for selecting timeouts dynamically


#### Unreliable clocks


* Clocks are used for measuring durations, and points in time.
* In a distributed system, time is a tricky business
   * Clocks seem simple and easy, but have many pitfalls
   * Easy for CPU faults to be noticed, not so for clocks
   * Can be hard to work out the order of events when multiple machines are involved
   * Each machine will have its own quartz crystal oscillator with its own imperfection
   * Clocks can be kept in check through something like NTP which uses more accurate time sources such as GPS
* Computers have two clocks, a monotonic clock and a time-of-day clock
* Time of day
   * Returns the wall-clock time according to some calendar
   * Usually synchronised with NTP
      * Can go backwards in time
   * Leap seconds
   * Bad for elapsed time
* Monotonic clock
   * Always guaranteed to move forward
   * Absolute value of the clock is meaningless only the difference between two values
   * Monotonicity not necessarily guaranteed across multiple CPUs
   * NTP adjusts the frequency that clock moves forward (slew) a small amount
* Clock problems
   * Skew and drift
      * Quartz not very accurate (Google services assume a 200ppm drift which will lose 17 seconds a day)
      * Skew - difference between two clock values
      * Drift - difference between two clock rates
   * Refuse or Reset
      * Large drifts may be ignored, or forcibly reset (allowing a clock to go back in time)
   * Firewalls, partitions and network delay
      * Clocks may be blocked from accessing NTP
      * NTP only as good as the network delay, busy networks can yield poor sync
   * Leap seconds
      * Can make a minute last 59 or 61 seconds, possibly breaking assumptions
      * Some NTP systems “smear” the second over the day
   * VMs
      * VM clocks are virtualised, so appear “paused” when other VMs are running
   * Embedded or mobile devices
      * Clock may be garbage
      * Users may have inadvertently or purposefully messed up their clock
* You can have an accurate clock if you want it enough (Precision Time Protocol) but requires expense and expertise and usually isn’t needed
* If you rely on accurate timekeeping, you must carefully monitor their synchronicity
   * Eg. Toss nodes out of your system if their clock is out of range
* If two clients write to a database, who got there first?
   * Clocks generated by clients can lead to cases where transactions are replicated in reverse order (if the first update is done with a fast clock)
   * Last write wins (LWW) needs causality tracking mechanisms such as version vectors to prevent violations of causality from clock skew
   * It is possible for two clients to generate writes with the same timestamp depending on the resolution of the clock, in these cases LWW normally randomly breaks the tie (leading to a lost update)
* How does a database know it is still the leader?
   * Keeps a lease from the other nodes that expires at a given time
   * But any number of things can preempt and resume a thread, meaning a lease can expire while the thread is paused and carry on regardless later
      * What if the Java GC runs (“stop-the-world GC” can last for minutes)
      * VMs can be suspended and resumed in the middle of something
      * Threads are paused in context-switching, or waiting for disk (or network, or network disk!)
      * SIGSTOP and SIGCONT
* RTOS systems use “real-time” in the sense that the system is carefully designed and promises to meet specified timing guarantees in all circumstances (eg. a garbage collector won’t stop the airbag in your car deploying)
   * RTOS simply not economical or appropriate for server-side data processing
   * Usually lower throughput and much more restrictive environments
* Definition of “most recent update” (LWW) relies on a correct time-of-day clock!
* Even with NTP sync’d clocks, a packet could technically be sent before it arrives (according to the timestamps on the sender and receiver)
* Logical clocks are safer
   * Do not measure time of day, or elapsed time
   * Incrementing counters
   * Allows determination of event ordering
   * Lamport timestamps and version vectors https://towardsdatascience.com/understanding-lamport-timestamps-with-pythons-multiprocessing-library-12a6427881c6
* Reading a clock is less like getting a point in time and more like reading a most probably point in time (eg. a system is say, 95% confident the time is between two boundaries)
   * Microsecond precision for time of day is basically useless
   * Google’s TrueTime API in Spanner reports the earliest and latest time
* Global snapshots need to be a monotonically increasing ID
   * Hard to generate, requires coordination across data centers
   * Twitter’s Snowflake generates approximately monotonically increasing IDs
   * Spanner uses TrueTime, waits for the length of its time confidence interval before committing a read-write transaction
   * Google has a GPS or atomic clock in each DC to keep the confidence interval as small as possible


#### Knowledge, truth and lies


* Nodes cannot really be trusted to know their own state either (imagine a node waking up from a minute-long stop-the-world GC)
* Quorums are often used to spread decisions across multiple nodes
* Majority rule is easiest, there can only be one possible majority vote
* Frequently a system requires there to be one of something (a database leader, a lock holder), but how to make sure that a false leader cannot disrupt the system
   * Fencing -- attach an incrementing token to the lease, expired tokens are rejected from potentially disruptive operations
   * Zookeeper can be used as a lock service wherein the transaction zxid or node cversion are guaranteed to be monotonically increasing
   * Requires the application to check the token of course
   * A node could just circumvent this by sending the wrong token on purpose (but we assume nodes are unreliable but honest)
* Assume nodes are playing by the rules of the protocol to their best
   * Distributed systems become much more difficult to work with if nodes are allowed to “lie” -- Byzantine faults
   * A system is Byzantine fault tolerant even if it operates correctly with nodes that do not follow the agreed protocol (eg. aerospace, blockchain)


#### System model


* Three models of timing assumptions
   * Synchronous
      * Bounded network delay, process pauses and clock error
      * Not very realistic
   * Partially synchronous
      * System behaves synchronously mostly of the time
      * Sometimes exceeds bounds for pauses, delay and clock drift
   * Asynchronous
      * All bets are off
      * Assume there is no clock at all
* Three models for system models
   * Crash-stop
      * A node can only fail by crashing
      * Nodes may suddenly stop at any moment and never come back
   * Crash-recovery
      * Nodes can stop at any time and will try and return
      * Disk storage persists but in-memory data is lost
   * Byzantine
      * Nodes may crash and do anything, including attempting to deceive other nodes that they are still working
* Partially-synchronous crash-recovery is the most common model
* Algorithms must be shown to be correct by describing properties with respect to the chosen model
   * Safety
      * “Nothing bad happens”
      * If violated, you can point to a particular point in time
      * The violation cannot be undone
      * Should always hold regardless of circumstances
      * eg. No request for a token should return the same value
   * Liveness
      * Often includes “eventually” in the description
      * Should always hold in best circumstances
      * eg. A request for a token is eventually responded to
* Sometimes despite models and proofs of algorithm correctness, there is no better option that exiting and letting a human clean up the mess
   * “The difference between computer science and software engineering”


### Consistency and consensus


* Packets can be lost, reordered, duplicated and delayed; clocks are merely approximate at best and nodes can pause or crash at any time
* If bringing the system down and showing an error is unacceptable, we must be able to tolerate these problems
* Find general-purpose abstractions, implement them once and let applications rely on the guarantees afforded (eg. transactions mean an application doesn’t have to worry about crashes [atomicity], concurrency [isolation] and storage reliability [durability])
* Consensus is one of the most important such abstractions -- getting nodes to agree on something
   * Need to avoid split brain (multiple leaders in a single leader database system)
* Most replicated databases provide eventual consistency
   * Inconsistency is temporary and the replicas eventually converge
   * Weak guarantee, as you don’t know when “eventually” is
   * Easy to think of a database as a variable that you can guarantee read/writes from, but its much more complicated!
   * Bugs only tend to appear with high concurrency or a network fault (eg. the Majora phantoms during network slowdowns)
* Strong guarantees have a higher performance penalty or be less fault-tolerant


#### Linearizability


* aka Atomic consistency, strong consistency, immediate consistency
* Would be nice if the database could give the illusion that there is only one replica
* Regular register allows reads to return old and new values if they are concurrent with write
* Recency guarantee ensures once a new value has been written (or read), all subsequent reads will see the same value
* Easily confused with Serializability
   * Serializability is an isolation property of transactions, it guarantees that transactions behave as if they had executed in *any* serial order (not necessarily the order they were actually run)
   * Linearizability is a guarantee of recency of reads and writes of a register (an individual object), does not prevent write skew or phantoms
   * A database can provide both -- known as strict serializability or strong one-copy serializability (strong 1SR)
   * SSI (serializable snapshot isolation) is not linearizable as it does not include reads more recent than the snapshot, so reads from the snapshot cannot be linearizable
* Locks must be linearizable
   * All reads and writes determining ownership of the lock must agree
   * Zookeeper and etcd provide linearizable writes but reads can be stale
      * etcd has quorum reads for linearizable reads
      * Zookeeper requires a sync() call before the read
      * Apache Curator builds recipes on top of Zookeeper
   * Linearizable storage service is foundational for coordination tasks
* Unique indexes or constraints (balance > 0, stock > 0) must be linearizable
   * Similar to a lock or an atomic “compare and set”
* Cross-channel timing dependencies
   * Imagine a file uploader and resizer service
   * Risk that the resizer is run before the file has been replicated
   * Two different communication channels
   * Need the recency guarantee of linearizability (eg. reading your own writes)
* How to handle linearizability
   * Single leader replication (no)
      * Has the potential to be linearizable, but usually isn’t by design (eg. using snapshot isolation)
      * Using the leader for reads means you need to know who the leader is
      * Async replication means you could even lose data on failover which is neither durable or linearizable
   * Consensus algorithms (yes)
      * Similar to single leader replication but with measures to prevent split brain and stale data
      * Zookeeper and etcd
   * Multi leader replication (no)
      * Concurrently process writes on multiple nodes and rely on async replication
      * Allow writes with conflicts that require resolving
   * Leaderless (probably no)
      * Potential for strong consistency with quorum reads and writes (w+r >n)
         * However, network delay can yield race conditions
      * LWW not likely to be linearizable
      * Sloppy quorums and hinted handoff unlikely to be linearizable
* Not even RAM is serializable, as CPU cores have their own caches and writes are applied async to RAM from the cache
   * Dropped linearizability for performance (and fault tolerance not necessary as its not expected that multi-core CPUs will lose connection to each other!)
* If you want linearizability, the response time of reads and writes is proportional to the delays in the network
   * Weaker models of consistency are faster
   * Physically impossible for linearizable to be faster (Attiya and Welch)


##### CAP


* If you require linearizability, and some replicas are disconnected...
   * ...they must wait until the network is fixed or return an error
   * Unavailable
* If you don’t require linearizability, and some replicas are disconnected…
   * ...they can process requests independently
   * Available, at the cost of linearizability
* Applications that don’t need linearizability can be more tolerant of network issues
* CAP “pick two of three” a little misleading, better to think
   * Consistent or Available when (network is) Partitioned
* CAP is “mostly of historical interest today”




#### Ordering guarantees


* Ordering is important
   * Determine the order of writes in a multi-leader system
   * Serializability ensures transactions behave as if they executed in some order
   * Timestamps and clocks are used to determine ordering of real-world events
* Ordering preserves causality
   * Causal dependency between reads of a question and its answer
   * Rows must be created before they can be updated
   * Determining if writes A and B are concurrent (and not causal linked)
   * Snapshots must be consistent with causality (if they contain an answer, it must also contain the question), read skew violates causality
   * Ticking items off a list is causally dependent with what is on the list
   * The example of an image not being replicated before resizing
   * A message is sent before it is received
* Total order
   * Any pair of elements can be compared and ordered
   * Natural numbers have total order
   * Sets are not totally ordered (some are incomparable)
   * Linearizability offers total order -- all operations can be ordered
   * Causal consistency (partial ordering) offers a weaker guarantee, two events are ordered if they are causally related (one happened before the other), otherwise they are concurrent and incomparable
   * ie. There are no concurrent operations in a linearizable data store
      * There is a single timeline of events
      * Concurrency means the timeline branches and merges again later
      * eg. git
   * Linearizability implies causality
* Causal consistency is the strongest model you can achieve that does not have impacted performance and remains available during network partitions
   * Promising direction for future systems
   * http://www.bailis.org/blog/causality-is-expensive-and-what-to-do-about-it/
* Capturing causal dependencies
   * Clients read lots of data before writing, its hard to know what is actually causal to the write
   * Capturing all this data is expensive
   * Sequence numbers or timestamps can be used to order events
      * Compact and offer total order
      * Note its possible to create a total order that violates causality, such as lexicographic ordering of UUIDs
      * Concurrent events can be labelled arbitrarily, so long as causally related events are orderable
      * Replication log of a single leader application defines the total order of writes that is consistent with causality
      * A replica following the order of the writes in the log will be causally consistent at all times (even if its lagging the leader)
   * Noncausal sequence numbers
      * Nodes generate their own numbers (eg. one odd, one even)
         * Doesn’t maintain the relationship between causal operations
      * Timestamps attached to numbers
         * Clock skew
         * Google Spanner does achieve this but its difficult
      * Blocks of numbers assigned to node
         * Causal ordering is violated as operations in the past can have higher numbers that ones generated by another node in future
   * Lamport timestamps
      * Causality-preserving sequence generator
      * One of the most cited papers about distributed systems (Lamport 1978)
      * A pair of (counter, node)
      * Every node and client keeps track of the maximum counter value they have observed
      * Provides total ordering and maintains causality
      * Sometimes confused with version vectors
         * Version vectors determine whether events are concurrent or causal
         * Lamport enforces total ordering but cannot determine what is concurrent or causal
         * Lamport timestamps are more compact
      * Lamport timestamps can retroactively organise a total order but does not help if you need to make a decision on a given node, right now
         * eg. “is this username taken?”
   * Total order broadcast
      * Not sufficient to have total ordering for uniqueness constraints, need to know when the order is finalised to be able to make decisions
      * Cannot have a fault-tolerant system if timestamp ordering is to determine uniqueness as all nodes must be available
      * A protocol for exchanging messages between nodes, must ensure:
         * Reliable delivery -- a message is delivered to all nodes without fail, even if there is a fault (ie. the system will ensure messages are delivered after the network is repaired)
         * Totally ordered delivery -- messages arrive at all nodes in the same order
      * Implemented in Zookeeper and etcd
      * Total order broadcast used for “State machine replication” which is used in database replication
      * A way of generating logs, sending a message is effectively appending to the log
      * Useful for fencing tokens (managing locks)


##### Implementations of linearizability


* Linearizable storage with total order broadcast
   * Linearizable read-write register easier than total ordering
   * Total order broadcast is equivalent to consensus which has no solution in a crash-stop model, whereas read-write registers do
   * Although compare-and-set or increment-and-set operations on a register require a total order broadcast like solution…
   * Works like:
      * Imagine a linearizable register for the resource you’re trying to protect
      * Execute a compare-and-set on the register
      * Append a message to the log
      * Read the log and wait for your own message to be played back to you
      * If no other message set the register, you can continue
   * Ensures linearizable writes but not linearizable reads
   * Provides sequential/timeline consistency (weaker than full linearizability)
      * It is possible to read from a source that is behind the head of the log
   * You can enable linearizable reads by…
      * Appending prospective read to the log, read the log, and perform the read once it is returned to you (ie. the message’s position in the queue defines when the read happens)
         * Quorum reads in etcd work like this
      * If you can fetch a position from the log in a linearizable way, you can query that position and wait for all entries up to that position to be delivered and perform the read
         * Zookeper sync()
      * Synchronous replication


* Total order broadcast with linearizable storage
   * Imagine an atomic compare and set
   * For every message you want to send through total order broadcast, increment the linearized integer counter, and attach it as a sequence number to the message
   * Forms a sequence with no gaps (unlike Lamport)
      * A node therefore knows which messages it has missed and must wait for them to be resent
         * A key difference between total order broadcast and regular ordering
* Linearized compare-and-set (or increment-and-get) are total order broadcast are equivalent problems to consensus (and can be used to generate solutions for eachother)


#### Consensus


* An important and fundamental problem in distributed computing
* The goal is to get a group of nodes to agree on something
* Has taken decades to refine the meaning on consensus
* Important for nodes to agree on things such as
   * Leadership elections
      * Consensus required to avoid split brain in a single leader situation
   * Atomic commit
      * A transaction may fail on one node and not others, which violates ACID if left unchecked (all or nothing)
      * Nonblocking atomic commit is a harder problem than consensus
* FLP result (Fischer, Lynch, Paterson) prove there is no algorithm that can reach consensus in a system where there is a risk a node could crash
   * Only actually proved for the async system model where you cannot rely on clocks or timeouts (as timeouts can be used to ignore nodes)


##### Two phase commit (2PC) or blocking atomic commit


* Atomicity prevents the database from getting littered with half-finished state
* In a single node system, atomicity hinges on the disk controller that writes the commit record
* What if multiple nodes are involved? Most NoSQL databases don’t handle this, it’s not as simple as sending the commit to all nodes as they must fail or succeed as one
   * A node should only commit once it is certain that all nodes will commit
* 2PC is an algorithm for achieving atomicity in a multi node system
* Not to be confused with 2PL (two phase locking)
* 2PC uses a “coordinator” known as the transaction manager (normally implemented as a library into the application requesting the transaction)
* The coordinator asks all nodes if they are prepared to commit, and sends a signal to commit (if all agree) or abort (if any disagree)
* In more detail
   * The application requests a globally unique transaction ID from the coordinator
   * Application begins a single node transaction on all nodes
   * The coordinator sends a prepare signal with the transaction ID, if any request fails or times out, the abort signal is sent
   * When a participant (node) receives a prepare request, it makes sure the transaction will committed under all circumstances, checking for any conflicts or constraint violations
   * Once a participant votes yes, it surrenders the right to abort
   * When the coordinator receives all the votes, it writes its decision to disk in case the coordinator itself crashes, this is the commit point
   * The commit signal is sent, if a commit fails or timesout, it must retry forever
* If the coordinator crashes between the prepare and commit/abort, the transaction is “uncertain”
* 2PC is protected by a single node atomic commit on the coordinator (ie. writing the log)
   * 2PC is blocking as nodes can get stuck waiting for the coordinator 


##### Three phase commit (3PC)


* A proposed solution to the blocking element of 2PC
* Assumes a network with bounded delay and response times, not practical in most cases
* Requires a perfect failure detector (hence the need for bounded delay)


##### Distributed transactions in practice
 
* 2PC has a bit of a bad rep (mostly performance and promising too much)
   * Distributed MySQL transactions are allegedly 10x slower than single node
   * Most of this is down to the required fsyncing required to ensure logs are on disk as well as the network round trips
* Two types of distributed transactions
   * Database internal
      * All nodes run the same database software and have their own support for distributed transactions
   * Heterogeneous  
      * Nodes use a mixture of two or more technologies (different databases, brokers etc.), must ensure atomic commits even if the systems under the hood are different
      * Each system must not have side effects that violate the 2PC agreement (eg. a transaction that sends an email should only send on commit)
* Database internal works better as they only have to work internally, heterogeneous systems are much more challenging
* X/Open XA (extended architecture) is a standard for implementing 2PC across heterogenous systems, widely adopted since 1991
   * Postgres, MySQL, DB2, SQL Server, Oracle
   * ActiveMQ … IBMMQ etc.
   * Not a network protocol, but a C API for interfacing with a transaction coordinator
   * Implemented in the application and exposes APIs to manage transactions
   * All communication via the application/client library
* Stuck transactions cause trouble because of locking
   * Row level exclusive locks (for the case of read committed isolation), 2PL would lock any row read by the transaction
   * These locks cannot be released for transactions in doubt
   * No writes and possibly not even reads, holding up other transactions
* Orphaned uncertain transactions can occur
   * Must be manually fixed 
   * Proper implementations of 2PC will maintain locks over reboots!
   * Heuristic decision making is permitted in XA
      * Will most likely break atomicity and should be avoided until everything has failed
* Coordinators usually have bad support for replication
   * Including the coordinator in the application can make it harder to deploy
   * XA has limited support for different database isolation levels (because it must support all possible systems that implement its API)
* Distributed transactions tend to amplify problems
* Should we give up?... Find out later


##### Fault tolerant consensus


* One or more nodes propose a value, then a consensus algorithm decides on one
* A uniform consensus algorithm (on an async system with unreliable failure detection) must satisfy the following:
   * Uniform agreement -- No two nodes decide differently
   * Integrity -- No node decides twice, or changes its mind
   * Validity -- If a node decides on v, v was an option proposed by a node
   * Termination -- Every node that does not crash eventually decides, so long as half of them are still working (quorum) with non Byzantine errors
* Any algorithm that must wait for a node to recover does not satisfy termination
* The first three properties are safety properties and are usually always met


##### Consensus and total order broadcast


* Best known algorithms are Viewstamped Replication (VSR), Paxos, Raft and Zab
* Not advisable to do this yourself “it’s hard”
* These algorithms generally decide a sequence of values (rather than a decision of a single value), making the total order broadcast algorithms
* Effectively deciding which message to send next
* Agreement (all nodes agree to the message order), integrity (messages are not duplicated), validity (messages are not corrupted or fabricated), termination (messages are not lost)
* Manually deciding the leader is a consensus algorithm with a dictator
   * Does not satisfy the termination property as nodes cannot progress without a human
* Need to avoid split brain
* How to pick the first leader?


##### Epoch numbering and quorums


* Consensus protocols usually have a leader but does not guarantee the leader is unique
* An epoch number (Paxos ballot number, view number VSR, term number Raft) provides a weaker guarantee, within each epoch, the leader is unique
* Epochs are totally ordered and monotonically increasing
* Decisions from the leader are sent to other nodes who vote on the proposal, taking care to ignore decisions from nodes with a lower epoch (ie. an old leader)
* Two rounds of voting, one to pick the leader, one to make a decision
   * These votes must overlap, at least one node that votes for a decision must have also voted in the leadership ballot
   * If a decision vote does not reveal a higher epoch number, the leader can safely assume no election has taken place without their knowledge
* Similar to 2PC but
   * 2PC coordinator is not elected and requires a unanimous vote
   * Consensus algorithms additionally ensure a recovery process to regain consistency after an election is followed


##### Limitations


* Concrete safety properties to uncertain systems
* Provide total order broadcast and therefore linearizable atomc operations
* Async leader elections can lose data on failover but are still used for performance
* Consensus systems require a majority to operate
   * 3 nodes required to tolerate 1 failure
   * 5 nodes required to tolerate 2 failures
   * If a network partition that cuts off some nodes from the rest, only the majority side can function
* Can be harder to add/remove nodes from the cluster as elections are often fixed
* Highly variable delays can trigger elections even if the node is fine (especially so in geographically dispersed systems)
* Single unreliable network links can inadvertently trigger repeating election cycles


#### Membership and coordination services


* Zookeeper and etcd are often described as distributed key value stores or coordination and configuration services
* Likely relying on Zookeeper through related projects (eg. HBase, Hadoop, Nova and Kafka)
* Zookeeper modelled on Google’s Chubby Lock service
* Zookeeper implements
   * Linearizable atomic ops
      * Atomic compare and set locks
      * Locks are actually leases, a lock with a timeout
   * Total ordering
      * Fencing tokens to control access to leases to guard against process pause
      * ZK uses an increasing zxid and version cversion
   * Failure detection
      * Heartbeats check other nodes are alive
   * Change notifications
      * Clients can subscribe and hear about changes in membership
* Useful for leadership elections for single-leader databases, job schedulers etc
* Assigning nodes work, or ownership of partitions
* Hard to set up but usually less disastrous than doing everything yourself
* Designed for infrequently, slow changing properties such as “the leader of the database is on IP x”, not intended for any application state
* Service discovery to provide a phonebook of services to applications
* Membership services for attempting to collect groups of nodes that are alive


#### Summary


* Linearizability is a consistently model whose goal is to make replicated data appear as if it was one copy, and for operations on it to act atomically
   * Makes a database behave like a variable in a single threaded program
   * Slow, and depends on network delay
   * All operations on a single totally ordered timeline
   * Concurrent operations are not possible
* Causality, imposes an ordering on events in a system (what happened before what)
   * Branching and merging timeline
   * Concurrent operations are possible
   * Less overhead than linearizability, less sensitive to network delay
* Linearizability and causality do not help for uniqueness and constraint checks, requires consensus
   * Nodes need to know another node isn’t in the process of changing a unique object
   * Irrevocable decision making process that a node cannot back out of after deciding
* Wide range of problems are reducible to the problem of finding consensus
   * Linearizable compare and set registers (decide to set a value)
   * Atomic transaction commit (decide to commit)
   * Total order broadcast (decide on message order)
   * Locks and leases (decide lock acquisition)
   * Membership (decide which nodes are alive)
   * Uniqueness constraints (decide which request registers the unique item)
* All these problems are straightforward in a single node system, or rather if the decision making is left to one node
* In a multi leader system…
   * Wait for the leader to return and lose availability in the meantime
   * Manually fail over and reconfigure the system
   * Algorithmically decide a new leader with a consensus algorithm
* Zookeeper outsources consensus, failure detection and membership
   * Hard to use, but more likely to fuck it up on your own
* Not every system requires consensus
   * Maybe you can tolerate branching and merging timelines in your history


## Derived Data


* No one database can solve all needs
* Applications will use a combination of different data stores, indexes, caches, analysis and have mechanisms to shift data from one source through to another
* Integrating disparate systems is often overlooked and in reality is one of the most important things you must do in a non trivial application
* Systems that store data can be divided into:
   * Systems of record
      * a source of truth
      * holding authoritative versions of data
      * usually where data is written into existence first
      * Usually normalised (only exists once)
   * Derived data systems
      * Transforming and processing existing truth and storing it somewhere else
      * If lost, it can be recreated from the source
      * Caches, denormalized views, indexes
      * Redundant data
      * Often denormalised for performance


### Batch processing


* Different types of systems
   * Online services
      * Waits for a request to arrive
      * Tries to handle it as quickly as possible and replies
      * Availability and response time are the primary performance measures
   * Offline batch services
      * Large input
      * Runs a job (often takes a while)
      * Jobs are usually scheduled rather than sent by users
      * Primary performance measure is throughput (time taken to crunch)
   * (Near) real-time stream processing
      * Somewhere between online and offline 
      * Operates on events shortly after they happen
      * Lower latency to respond than batch processing
* MapReduce, a batch processing algorithm published in 2004 was called “the algorithm that makes Google scale” and has been implemented in Hadoop, CouchDB and MongoDB
   * MapReduce is a low level model compared to parallel processing systems developed for datawarehouses before
   * Importance of MapReduce is declining (apparently)
   * Bears an uncanny resemblance to the electromechanical IBM card-sorters of the 1940s (“history has a tendency of repeating itself”)
* Sorting vs. in-memory aggregation
   * Sorting approach makes efficient use of disks
   * Similar approach to SSTables/LSM-Trees
   * Mergesort has access patterns that perform well on disks
   * UNIX `sort` automatically handles larger-than-memory data by spilling to disk and automatically parallelising sort across multiple cores
      * Bottleneck will therefore by disk I/O
      * Better than most library sorting implementations


#### UNIX philosophy


* Doug McIlroy, the inventor of the UNIX pipe (1964)
   * “...connecting programs like [a] garden hose”
* 1978 philosophy
   * Do one thing well - “build afresh rather than complicating existing programs”
   * Expect a program’s output to be another’s input - don’t clutter output with extra information
   * Design and build to be tried early, don’t be afraid to start over
   * Use tools in preference to unskilled help, even if you have to detour to build the tool
* Automation, rapid prototyping, incremental iteration, experimentation
   * ...sounds like the Agile and DevOps movements of today!
* Uniform interfaces
   * File descriptors
      * BSD socket interface deviated from this style, requiring netcat or curl to treat network connections like a file
   * URLs/HTTP
      * Not like old BBS where a phone number and modem settings were the way to “link” to other boards
* Separation of logic and wiring
   * UNIX tools assume stdin and stdout
   * Program doesn’t know or care where data is coming from or going
* Transparency
   * Input files usually treated as immutable
   * Piping to less good for basic debugging
   * Output can be written to a file to pick up the next stage without restarting
* Biggest limitation was they only ran on one machine… enter Hadoop


#### MapReduce and distributed filesystems


* Like UNIX tools but distributed
* MapReduce reads and writes files on a distributed filesystem
   * HDFS -- Hadoop Distributed File System
      * An open source implementation of Google File System (GFS)
   * Other distributed filesystems exist (GlusterFS, QFS) and Object storage systems (Amazon S3, Azure Blob, OpenStack Swift) are similar -- although HDFS is unique in that jobs can be run on nodes where the data is (objects stores usually keep storage and compute separate)
      * “Putting the computation near the data”
         * Reducing network load, increasing performance
      * Although if using Erasure Coding, this locality advantage is lost as the file fragments need to be combined from multiple machines
* HDFS based on the shared-nothing architecture
   * Contract to the shared-disk architecture of NAS/SAN requiring specialized hardware
   * Shared nothing approach can run on commodity hardware
* HDFS exposes a daemon on each machine, allowing other nodes to access files stored on that machine
   * A central server called a NameNode keeps of track of which blocks are where
   * Effectively presents one large file system that has the capacity of all disks in the system
   * File blocks are replicated on multiple machines to tolerate machine and disk failures
   * Erasure coding like Reed-Solomon allows for recovery of lost data with lower overhead that full replication
   * Similar to RAID but for a distributed file system
* MapReduce is a programming framework to process large datasets in a distributed filesystem like HDFS -- pattern:
   * Read a set of inputs into records
   * Call the mapper to extract a key and value from each record
      * Mapper called once for each record
      * Can generate zero, one (or more) key-value pairs from the record
      * Records are handled independently 
      * Prepares data for sorting
      * Number of mappers determined by the size of the input data
   * Sort the key-value pairs by key
      * Partitioning by reducer, sorting and copying data partitions from mappers to reducers is called the shuffle
   * Call reduce over the sorted key-value pairs
      * Reducer operates over the values for a given key
      * If the keys are duplicated they are now sorted and can be combined if appropriate
      * Process the sorted data
      * Number of reducers determined by the author
* The main difference from UNIX tools is that MapReduce parallelises across many machines without having to explicitly handle the parallelism
* Hadoop MapReduce, the mapper and reducer are Java classes implementing an interface
* Mongo and Couch, the mapper and reducer are JS functions
* The problems that can be solved with one round of MapReduce is quite limited
   * Common for jobs to be chained together into workflows
   * No way to do this with Hadoop, usually achieved by writing the results of one MapReduce job to a particular HDFS directory, and read that dir from the second
   * Less like UNIX pipes, more like independent commands
   * Various schedulers for Hadoop have been developed
      * Oozie, Azkaban, Luigi, Airflow, Pinball
      * Required for large collections of jobs
   * Higher level tools for Hadoop set up workflows and wire them together
      * Pig, Hive, Cascading, Crunch, FlumeJava


#### Reduce-side joins and grouping with hadoop


* Common for records to have an association with each other (foreign keys, document references, edges), a join is required when access is needed to both sides of the association
* Equi-joins (the most common type) but some databases allow other joins (eg. using LTE, GTE)
* Imagine joining clickstream or event data with a User ID to the Users table (like a star schema where the event table is the fact table and the user profile is a dimension)
   * Matching each event to its user obviously going to be poorly performing, especially if the user table is not local
   * Better approach would be to (extract transform load) ETL the user database into HDFS
* A user activity and user data mapper would map user keys to activity and profile data and the reducer would collate them per user id
   * Secondary sort can ensure the profile data appears first and URLs after so the reducer can output deterministically
   * Sort merge (or reducer-side) join (mapper output sorted by key for reducer, reducers merge both sides of join)
   * Keys act like an “address” that will route all key-values pairs for the same key to the same mapper
   * GROUP BY works similarly (eg. sessionization of user activity streams)
* Hot keys (linchpin objects) can cause skew and keep one reducer much busier than others
   * Skewed join (Pig) tries to check which keys are hot and random sends key-values for hot keys to random reducers (the other side of the join is sent to all reducers for a key)
   * Sharded join (Crunch) works similarly but needs an explicit list of hot keys
   * Skewed join (Hive) requires hot keys to be specified in table metadata and stores data for those keys separately (and uses a map-side join, not sort-merge join)
   * Two stage sort merge join needed to pull key-values from multiple mappers in order to provide sorted single input to reducers


#### Map-side joins


* Map-side joins do not make assumptions of the input data, but are much more expensive
* Often requires several stages of MapReduce (writing a lot to disk in each cycle)
* If you can make some assumptions of the input data, a map-side join skips sorting and reducing and writes outputs to one block of the file system immediately
   * eg. joining a very large table to a very small table (broadcast hash join)
   * eg. joining a very large table to a partitioned small table (partitioned hash join)
* Imagine if the user database from the previous section could fit in memory (and be loaded into each mapper)
   * Each mapper will scan the activity fact table and output the user profile information by looking it up in a hash - broadcast hash join (Cascading, Crunch)
   * aka replicated join (Pig), mapjoin (Hive)
   * Alternatively, the small table can be written to a read-only index on disk to take advantage of OS-level page cache (providing RAM-like lookup speeds without fitting the dataset into RAM)
* If the inputs can be partitioned, you can additionally split the work down by partition (partitioned hash joins)
   * eg. Partition user IDs by their last digit and each mapper loads hashes of the small table (eg. the profiles) for only the relevant objects
   * aka bucket map join (Hive)
   * If the input data is also sorted, this collapses to a map-side merge and it doesn't matter whether the input can fit in memory as the inputs can be read sequentially
* Output of a reduce-side join is sorted by the join key, whereas the output of a map-side join is sorted in the same way as the large input
   * Important to know the layout of a dataset when joining
   * This metadata is catalogued in HCatalogue / Hive metastore


#### Batch workflow output


* Recall distinction between transaction processing (OLTP) and analytics (OLAP)
* Batch processing typically scans over large inputs, more akin to the OLAP paradigm
* However MapReduce jobs output some sort of structure, rather than a report (like OLAP)
* The original use for MapReduce was building indexes at Google for search results
   * no longer used for this purpose, but still good for Lucene/Solr indexing today
* Briefly recall Lucene is a file (term dictionary) that offers fast lookup of keywords to documents (posting list)
   * Simplistic example doesn’t cover relevance ranking, misspellings, synonyms
   * Batch process works effectively for this sort of work; mappers partition documents, reducer builds index for its partition and index results are written to the filesystem
   * Indexes can be built incrementally (Lucene uses segment, merging and compaction)
* Batch processes also build models for ML classifiers (spam detection, anomaly detection, image recognition) and recommendation engines (people you know, products you want, related searches)
   * Output for such jobs is often another database
   * Making network requests inside jobs will be an order of magnitude slower
   * Making concurrent requests to the same database will make things worse
   * Using another database breaks Hadoop’s “all or nothing” results system
   * Solution is to build a database inside the job, written as immutable files in the distributed fs
   * Voldemort, Terrapin, ElephantDB and HBase bulk loading
* Hadoop uses the UNIX philosophy of immutable inputs and experimenting with outputs
   * Provides human fault tolerance (reverted code can revert data), minimising irreversibility, transient errors are automatically tolerated, separates logic from wiring (a team could focus on the hadoop tasks while another worries about deploying)
* Avro and Parquet are structured input formats that provide schema-based encoding unlike unstructured text files
#### Hadoop vs. distributed databases


* The sushi principle “raw data is better”
   * Hadoop opened the possibility of indiscriminately dumping data into HDFS for processing later, whereas databases require data structured according to their model
   * Making data available quick can sometimes outweigh formatting it nicely in a database, simply bringing data to one place can be valuable (eg. warehousing)
   * Makes standardizing the data the consumer’s problem, especially helpful where the producers and consumers are different teams
   * There may not even be one single suitable data model
* MapReduce and Hadoop allowed code (as well as queries) to scale to large data sets, beyond SQL
   * Not all processing can be sensibly expressed with SQL (although Hive does allow for SQL to be used too)
   * Saves having to move data around to specialised systems for specific queries
   * HBase and Impala can exist in the same ecosystem despite very different approaches
      * HBase OLTP
      * Impala massively parallel processing database (MPP)
      * Neither use MapReduce but both leverage HDFS
* Designed for fault tolerance
   * Batch processes are less sensitive to faults as users are not immediately affected
   * Databases encountering a fault will require users to submit their query
   * MapReduce writes to disk partly for fault tolerance (and also as it is unlikely to fit in RAM anyway), databases very keen to keep everything in memory for performance
   * Jobs run for so long, they are likely to experience at least one task failure
   * In its original design, MapReduce tasks had to run in the knowledge that their work could be preempted in favour of higher priority prod tasks at any time


#### Beyond mapreduce


* MapReduce is one of many distributed programming models
* This book focuses particularly on MapReduce as its a nice model to understand (a simple abstraction on top of a distributed filesystem) although it is not necessarily easy to use
* Pig, Hive, Cascading and Crunch are higher level abstractions on top of MapReduce to make it easier to use
* Some problems are not solved with an extra layer of abstraction
   * MapReduce fully materialises inputs of one job to the next (unlike UNIX pipes that stream), which can be useful if an intermediate data set is published for use in its own right, but is wasteful if not
   * Commands connected by UNIX pipes all start at the same time and operate immediately on any input they receive, whereas MapReduce jobs in one cycle must all finish before the next mapreduce can begin (prone to skew)
   * Mappers are often redundant, reading back the same file just written by the previous reducer, in some cases reducers could have just been chained together without mappers at all
   * Intermediate files are often replicated, which is unnecessary
* Spark, Tez and Flink are the most well known execution engines for distributed computation
   * These systems are often called dataflow engines
   * Like MapReduce these engines take a user-defined function to process a record in one thread, parallelize work with partitioned inputs and copy inputs over the network to become outputs to other functions
   * Unlike MapReduce, the workflow is handled as one job, rather than independent subjobs; the functions do not strictly alternate between maps and reduces and offer more flexibility (operators)
* This style of processing is based on research systems (Dryad, Nephele)
   * https://dl.acm.org/doi/10.1145/1272998.1273005
   * https://dl.acm.org/doi/10.1145/1646468.1646476
   * Expensive work (sorting) is only performed when needed (unlike between every map/reduce)
   * No unnecessary map tasks (as the mapper work can be incorporated into the reducer)
   * Joins and dependencies are explicit, so a scheduler has advance knowledge of which data will be needed where (eg. avoiding copying files over the network if jobs can be run on the same machine)
   * Intermediate data can be kept in memory or written to local disk (not HDFS)
   * Operators can start executing as soon as input is ready (like a pipe)
   * JVM threads can be reused instead of starting the JVM for each task
* Pig, Hive and Cascading can be switched from MapReduce to Spark with configuration changes
* Tez is a fairly thin library, whereas Spark and Flink are big frameworks with their own network layer, scheduler and APIs
* Spark, Flink and Tez do not fully materialize intermediate state, making it less robust to faults than MapReduce, intermediate output must be recomputed from available data
   * Spark uses the resilient distributed dataset (RDD) abstraction for tracking ancestry of daa
   * Flink checkpoints state of operators
   * Downstream applications using non-deterministic data must be killed if an intermediate state is lost
   * Sometimes cheaper to just materialise the intermediate state (eg. highly compute intensive work)
* Better to make output deterministic to avoid a cascading fault
   * Easy for nondeterminism to creep in (no guarantees on iteration of hashes, random number generation, clocks)
* Sorting operations need to consume their entire input, accumulating a large state; ideal to avoid wherever possible as other workflow steps can be pipelined more efficiently 


#### Graphs and iterative processing 


* Batch processing of an entire graph
* Traverse an edge, joining vertices to propagate information and repeating until a condition is met (no more edges to follow, some sort of metric converges)
* MapReduce does not allow for “repeating until done” as it only performs a single pass, requiring an iterative workflow, where an external scheduler checks if a stopping condition is met and runs another iteration (works but inefficient)
* Bulk synchronous parallel (BSP) model has become popular as an alternative
   * Implemented by Apache Giraph, Spark GraphX, Flink Gelly
   * aka the Pregel model (Google Pregel paper popularised the model)
* Pregel “sends a message” from one vertex to another (along edges, typically)
* In each iteration, a function is called for each vertex, processing the messages from the vertex (much like a mapper “calling” a reducer), however the vertex persists its state between iterations (only processing new messages)
* Message passing allows Pregel to work more efficiently than querying vertices directly (as messages can be batched), Pregel guarantees messages are processed exactly once
* Pregrel periodically checkpoints the state of all vertices to durable storage for fault tolerance
* “Thinking like a vertex”, one vertex processed at a time
   * graph is often partitioned arbitrarily (optimising this step is hard)
   * Graph algorithms often have a lot of unavoidable cross-machine communications
   * If the graph can fit in memory, it will usually work better than a distributed graph algo
   * GraphChi could also work if the graph can be processed on one machine


#### High level APIs and languages
 
* Physically operating compute jobs at scale is considered somewhat “solved” and attention has turned to the programming model, improving efficiency and broadening the problems that can be solved
* MapReduce is quite laborious, Hive, Pig, Cascading and Crunch try to bridge the gap
* Dataflow APIs (Tez, Spark, Flink) use relation-style building blocks to express computation
   * Such high level interfaces require less code and make it easier to explore data in a REPL like shell
   * Another advantage of using high level building blocks means that the workflow executor can determine the most efficient method for performing low level tasks (eg. joins), Hive, Spark and Flink have cost-based query optimisers to do this (and can change the order of the joins to improve intermediate state)
   * Declarative joins mean the executor can worry about the laborious work of using the right join
* MapReduce built around function callbacks, for each record (or record group) a user defined mapper (or reducer) is called, with the advantage that a wide library of existing functions exist in the ecosystem
   * Differentiates MapReduce from MPP databases which offer less flexibility for running user code (often cumbersome and do not offer package management)
* Filtering operations can also take advantage of declarative workflows, for example using column-orientated storage to read only the required columns from disk in the first place
* Hive, Spark DataFrames and Impala also support vectorised execution, iterating over data in a tight inner loop that takes advantage of CPU caches
   * Spark generates JVP bytecode
   * Impala uses LLVM to generate native code for looping
* Many common cases where processing patterns keep recurring and being able to run arbitrary code is less important than having simple ways to handle common use cases
   * Mahout implements algorithms to support machine learning on top of MapReduce, Spark and Flink
   * MADlib implements similar functionality inside MPP database Apache HAWQ
   * K-nearest neighbours useful for similarity search in multi-dimensional space; approximate string search also important for genome analysis
* High-level declarative operators and MPP databases are looking more alike


#### Summary


* Design philosophy of UNIX is carried into MapReduce and dataflow engines
   * Immutable inputs, outputs are intended to become inputs, complex problems solved by chains of commands doing one thing well
   * Uniform interface of files and pipes replaced with a distributed filesystem
   * Dataflow engines build on top of this with pipe-like mechanisms to avoid materialising intermediate state
* Partitioning
   * Mappers are partitioned according to file input blocks
   * Output of mappers are repartitioned, sorted, merged into reducer partitions
   * Bring all related data (records with the same key) to the same place
   * Dataflow engines try to avoid sorting unless it is really required
* Fault tolerance
   * MapReduce writes to disk frequently to make it easier to recover from task failure, but incurs a time and overhead penalty (especially as HDFS is replicated)
   * Dataflow engines avoid materialisation and keep more data in memory, requiring more compute if a node fails
* MapReduce Joins
   * Sort-merge - mappers extract join key; partitioning, sorting and merging ensures records with the same key are sent to the same reducer
   * Broadcast hash - one side of the join is small, so is loaded into a hash table and allows the reducer(?) to match records to the small side of the join by querying the hash table
   * Partitioned hash - if the two inputs are partitioned in the same way (same keys, hash and partitioning) then the broadcast hash can be extended to additionally partition the hash table too
* Model
   * Deliberately restricted programming model
   * Callbacks (eg. mappers and reducers) are assumed to be stateless and have no external side effects, allowing the framework to handle some of the hard problems of distributed systems (node crashes, network issues, discarding partial outputs, retrying tasks)
   * Users don’t have to worry so much about coding for fault tolerance, the framework guarantees the output will work as if each task was executed once (even if failures had been tolerated under the hood)
   * Such “reliable semantics” are stronger than services that handle user requests and write to a database as part of the request
* Distinguishing feature of batch processing job is that the input is read and an output is created -- the output is derived from the input
   * The input is bounded -- it has a known, fixed size, meaning it is possible for a job to know when it has finished reading the input, ensuring jobs can be completed
   * Next chapter concerns unbounded inputs (never-ending data streams)


### Stream processing


> A complex system that works is invariably found to have evolved from a simple system that works. The inverse proposition also appears to be true: A complex system designed from scratch never works and cannot be made to work. John Gall, Systemantics (1975)


* Batch processing is derived data, it can be recreated by running the process again, this all relies on the assumption that the inbound data is bounded
   * eg. sorting operations central to MapReduce are able to determine they have found the lowest key once scanning through the input data to the end
   * Most systems produce data continuously, often batch processors divide the data into some chunk of duration (eg. today, yesterday, tomorrow)
   * These processes only reflect change once a day, which may not be suitable in cases where action is required immediately
* A stream refers to data that is incrementally available over time
   * stdin, stdout, lazy lists, Java FileInputStream, TCP connections, delivering audio/video over the internet
* Event streams as a data management mechanism


#### Transmitting event streams


* Records are more commonly known as events, small self-contained immutable objects
* May be appended to a file, inserted into a relational table or document database, or sent over a network to be processed by something else
* In batch processing a file is written once and can potentially be read by multiple jobs, likewise events are generated by a producer (publisher, sender) and potentially processed by a number of consumers (subscribers, recipients)
* Events are grouped together into a topic, or stream
* Basic stream processors can write events to a file and subscribers can poll and read changes
* Polling is too expensive for systems that require continual processing with low delays, better for consumers to be notified on new events
* Relational databases offer triggers which can react to changes but are limited in what they can do (and are somewhat an afterthought to most databases)
* As a result messaging systems have emerged as a means to notify consumers of events


#### Messaging systems


* TCP connection or UNIX pipe between producer and consumer would be a simple way to implement a messaging system
* Most systems expand on this model, as a TCP connection or pipe can only connect one producer to one consumer
* Variety of approaches to the publish/subscribe model, aimed at answering:
   * What happens if producers send messages faster than consumers can read them?
      * Drop
      * Buffer -- what happens if the queue grows, do you spill to disk?
      * Backpressure (block the producer) -- same approach as UNIX pipes and TCP
   * What happens if a node crashes?
      * Are the messages lost?
* An occasional missing data point may or may not be critical
   * Missed messages can lead to bad metrics, or wrong counters
* Harder to provide strong guarantees compared to batch processing model
* Direct messaging from producers to consumers
   * UDP multicast (used by stock markets for low latency feeds), application-level protocols responsible for handling missing packets, consumers can specifically request missing packets from producers
   * Brokerless messaging (ZeroMQ, nanomsg) use TCP or IP multicast
   * StatsD and Brubek use unreliable UDP for collecting machine metrics
   * Webhooks expose a callback endpoint for producers to send events
   * Generally requires the application to be aware of and handle potential loss of events, assumptions break down if the consumer is offline
* “Traditional” message brokers
   * Widely used alternative
   * A “database” for handling event streams, centralising events in the broker
   * Runs a server with producers and consumers connecting as clients
   * Central design makes it easier to tolerate clients coming and going as durability problems are offloaded from clients (and their applications) to the broker
   * Some brokers can be configured to write messages to disk so events are not lost and can support unbounded queueing for slow consumers
   * A consequence of the model is messages are async, the event sender only waits to confirm that the broker has buffered the message (rather than delivered the events to their destinations)
   * Brokers can use two-phase commit (with XA or JTA) like a database
   * Brokers are generally not used for long-term storage and will delete messages once it has been delivered to its subscribers
   * Working state is assumed to be small (unlike a database) meaning that performance can become degraded when there are many undelivered messages
   * Unlike a database, brokers provide much simpler means to filter events
   * Brokers do not support arbitrary queries but can notify clients when data has changed (the inverse of a database)
   * This model standardised in JMS and AMQP
   * Implemented by RabbitMQ, ActiveMQ, HornetQ, Qpid, TIBCO, IBM MQ, Azure Service Bus, Google Cloud Pub/Sub
* Handling multiple consumers reading the same topic
   * Load balancing (shared subscriptions)
      * Deliver event to one consumer so they share the work of processing
      * Arbitrary selection
      * Use pattern when events are expensive to process
      * JMS shared subscription, AMQP does this automatically for multiple subscribers to the same topic
   * Fan out (independent subscriptions)
      * Deliver to all consumers
      * Independent consumers “tune in” to receive the same messages
      * JMS topic subscription, AMQP exchange binding
* Acknowledgements and redelivery
   * Consumers can crash at any time
   * Clients must tell the broker when they have finished processing a message so it can be removed from the queue
   * It may be that the message was processed but the acknowledgement was delayed, requiring an atomic commit protocol if you want to handle this scenario
   * Messages are generally processed by consumers in the order they are sent by producers but redelivery can cause messages to be processed out of order if using load balancing (as a message from a failed consumer can be delivered out of order to another consumer after a timeout)
      * Not an issue if messages are independent but use case might mean that load balancing is not suitable
* Transient messaging mindset leads to a need for log-based message brokers
   * Receiving a message is a destructive operation (it is deleted from the broker), unlike batch processing where one can repeatedly experiment with immutable input data
   * New consumers only receive new messages, unlike batch processes that can process old files any time in the future
   * Why can’t we have both? Low latency messages with durable storage? Introducing log-based message brokers!
   * Effectively an append-only log on disk, consumers can read the log from start to end and then wait for new messages to indicate the log has been updated (think tail -f)
   * To scale this, logs can be partitioned to different machines
   * Total ordering can be achieved as the log is append only and each message can be given a monotonically increasing number
   * Apache Kafka, Amazon Kinesis, Twitter’s DistributedLog are log-based message brokers that work in this fashion
   * Google Pub/Sub is architecturally similar but exposes a JMS-style API
   * Despite writing to disk, these systems can process millions of messages a second through partitioning across multiple machines and replicating messages for fault tolerance
   * Trivially supports fan-out messaging and clients can read the log independently
   * Reading messages does not damage the log
   * Load balancing can be achieved by partitioning the log and assigning clients to different partitions, generally parallelism is increased by further subdividing the log (rather than adding more threads to a consumer, or adding consumers to the same partition)
   * A difficult to process message will slow all consumers reading that message from the log (head-of-line blocking)
      * In cases where messages are expensive to process and ordering is not so important, it may be preferable to use JMS/AMQP style brokering
   * Logs are only totally ordered within one partition, so good selection of partitioning key is important
   * Importantly, brokers no longer need to keep track of acknowledgements as consumers know which messages they have read from the monotonically increasing message offset
      * Broker only needs to occasionally check the consumer’s current offset
      * Similar concept to the log sequence number used in single-leader replication
      * If a consumer in a group fails, another consumer can be assigned the log partition with the last known offset
      * Offset can be changed to allow for replaying of old messages
   * To save disk space, the log is divided into segments; old segments are deleted or moved to cheaper archival storage -- technically a bounded queue but in practice the bound is large
      * As the consumer offsets are known, it is possible to raise an alert if a consumer is falling behind the log so badly that it might miss messages that are soon to be deleted (or archived)


#### Databases and streams


* Log-based message brokers take database ideas and apply them to message brokers, what about the other way around?
* Writes to a database ARE events, this event can be captured, stored and processed
* Shocker, a replication log is therefore a stream (produced by the leader carrying out transactions)
* There is no single system that can satisfy storage, querying and processing needs. How to keep data in sync between an OLTP system for user requests, common request caching, full-text indexing and data warehouse ETLs?
   * Dual writes too slow and introduce a host of concurrency issues and would require complex atomicity to get right
* Problem is that database replication logs were long considered an internal implementation detail rather than a public API, many databases did not provide a nice way to obtain information about changes to them
   * Growing interest in change data capture (CDC) where changes to a database are observed and extracted for replication in other systems
   * If the log of changes is applied in the same order, you can expect derived data sets to be consistent with the primary database
   * Through CDC the primary database is a single leader and all derived data systems are followers
   * Log-based message broker perfect use case as it can maintain order of messages
   * Database triggers can assist in CDC by registering triggers to append to a changelog (with performance penalties), alternative is to parse the replication log itself (which can be challenging)
   * Best to collapse the log into snapshots periodically such that the entire log does not need to be replayed from the start
   * LinkedIn Databus, Facebook Wormhole and Yahoo Sherpa use this idea at scale
   * BottledWater implements CDC for Postgres, decoding the WAL
   * Maxwell and Debezium parse the MySQL binlog
   * Mongoriver reads the Mongo oplog
   * GoldenGate provides CDC for OracleDB
   * Kafka Connect offers CDC connectors for various databases
   * RethinkDB, Firebase, Couch and Volt provide change feeds that are designed for integration into applications
* CDC logs can also be compacted, removing old events for data that is overwritten in future (effectively only keeping the most recent relevant event), logs therefore grow with the size of the database contents rather than the number of writes
   * Allows for keeping a durable log of the database without necessarily taking snapshots
   * Supported by Kafka
* CDC draws some parallel with DDD event sourcing
   * Idea of storing all changes to an application as a stream of events
   * Main difference is the database is not necessarily aware that its events are being sourced, whereas event sourcing requires specific choices to be made in an application’s design -- events reflect application domain changes, rather than low-level changes
   * Event sourcing is powerful for domain modeling, as events are recorded more meaningfully in the language of the application’s domain rather than low-level changes of the database (can answer why a change was made)
   * Harder to compact logs for event sourcing (as events do not necessarily supercede earlier events the same way as CDC)
   * In event sourcing a user command becomes an event if the conditions to carry out the action of the command are satisfied, and events are essentially facts, facts remain true even if a user performs a new action later that reverses the state of the first event (because it was still true that a user held that previous state for some period of time)
* If you are mathematically inclined (a past life maybe), you might say the application state is what you get when you integrate an event stream over time; and a change stream is what you get when you differentiate the state with respect to time
   * The state is always a sequence of events that caused the state to exist
   * If the changelog is stored durably, then so its the state (as its reproducible)
   * The real truth of the system is in the log, “the contents of the database hold a caching of the latest values of the log” - Pat Helland
* Event immutability is useful for auditing, analysis and experimentation
   * Can derive multiple read-orientated representations of the same log
   * Can be useful to keep applications evolving over time, if events sit between the application and database changes, new events can be routed to a new database without a migration, once all events have been migrated the old database can be shut down
* Storing data is much more straightforward if you don’t have to worry about how it will be accessed
   * CQRS (command query responsibility segregation) separates how data is written to how it is read
   * Complexities of schema design, indexing and database storage engines is mostly orientated around supporting certain query and access patterns
   * Database normalisation concerns itself with the idea that pieces of information should only be stored once, but it's entirely reasonable to keep a read-only denormalised table if you can keep it up to date (consistent) with an event log
   * Read optimised state
* Concurrency control is important for asynchronous event streams
   * A client may not be able to read their own write back from the log immediately
      * Read view could be updated when appending to the log, but ties the event log and read view in the same system (to support atomicity)
   * However, logs can make atomicity simpler in multi-object cases
      * Now applications need only write the single event to a log which is much easier to make atomic
      * If the event log and application state are partitioned in the same way (such that updates to state and their logs are mapped in the same way) then a single threaded log consumer can handle the event serially without concurrency control on writing
      * Log provides a serial order of events (for a partition)
* Limitations of immutability
   * Logs cannot really be compacted like CDC and can become prohibitively large
   * Administrative or legal reasons may require that previous events (or events with personal, or incorrect data) are removed -- in which case you must rewrite history to pretend the event never happened
      * Dataomic excision operation, Fossil VCS shunning


#### Processing streams


* Three main models for processing
   * Write events to derivative data sources (databases, caches, indexes) for later querying by other clients
   * Push events to users (emails, push notifications) or dashboards
   * Process events into a new derived stream
* A stream operator (or job) is related to UNIX pipes and MapReduce jobs with the main difference that the job never ends
   * Sorting does not really make sense, ruling out sort-merge joins
   * Fault tolerance considerations change, a stream job that runs for years cannot simply start over from the beginning
* Stream processing typically used in scenarios where an organisation wishes to monitor something and generate alerts
   * Fraud detection
   * Trading systems
   * Manufacturing reliability systems
   * Intelligence systems
   * Scientific experiments
* CEP (complex event processing)
   * Particularly aimed at pattern matching
   * Often uses high-level declarative language to describe the target patterns
   * Processing engine performs required matching and emits a complex event
   * Role reversal to a database, data is streamed and queries are stored
   * Implemented in Esper, IBM InfoSphere, Apama, TIBCO StreamBase, SQLstream
* Analytical processing
   * Blurred line with CEP but as a general rule, analytical processing is more about aggregation of metrics over a large number of events -- rates, rolling averages, detecting trends or deviations, often over fixed time intervals
   * Often employ probabilistic approximations
      * Bloom filters for set membership
      * Hyperloglog for cardinality estimation
      * t-digest or HDRHistogram for percentile estimation
   * Probabilistic algorithms are better thought of as optimisations rather than approximations
   * Apache Storm, Spark Stream, Flink, Concord, Samza, Kafka Stream
   * Google Cloud Dataflow, Azure Stream Analytics
* Materialised views
   * Any stream processor could be used for maintenance of materialised views (eg. data cubes) but keeping a full history of all events can run counter to a chosen analytical framework
   * Samza and Kafka support this usage, building on Kafka’s log compaction
* Stream searching
   * Full-text searching for media monitoring 
   * Turns conventional search engine on its head, instead of indexing first and performing queries; queries are stored and documents pushed through
   * Elasticsearch Percolator
* Message passing and RPC
   * Stream processing may look similar to the actor frameworks but actors are a mechanism for managing concurrency and distributed execution, not data management
   * Communication between actors is usually one-to-one and ephemeral


#### Reasoning about time


* You’d think “the last five minutes” is unambiguous but time is tricky
* Batch processing relies on historical timestamps in the raw data
   * Still has problems reasoning about time, but the problem is more noticeable in stream processing as we’re looking at real time events
* Stream processing often uses the local machine clock to determine windowing of events which can break down if there are delays on the local machine
   * eg. queuing, network faults, performance issues, consumers restarts, reprocessing events can all lead to delayed processing of new messages
   * Worse still is delays in the network can lead to messages arriving at the broker out of order (and downstream to the consumers) -- the processing time is inconsistent with the true event ordering
   * Eg. imagine a restart of a consumer that updates a dashboard to show request metrics, an artificial spike may appear as the restarted consumer processes the backlog of messages that arrived while it was restarting -- even if the request rate was actually constant
* When can you declare you’ve finished processing data for the current minute? What if straggler messages are stuck in the broker somewhere? Two options:
   * Discard stragglers (you can additionally count dropped events as a metric)
   * Correct published results
* Which clock to use for events?
   * Events from offline apps that periodically sync could appear as extreme stragglers
      * Can you trust the user’s clock anyway?
   * Events are often logged with three timestamps
      * Event time on device clock
      * Event broadcast time on device clock
      * Event received time on server clock
   * Now you can determine an approximate offset between the server and user device
* Windowing
   * Tumbling window -- a fixed length and events belong to exactly one window
   * Hopping window -- a fixed length with some fixed overlap, events can belong to more than one hopping window (eg. a 5 minute window with a 1 minute hop), can be achieved by calculating 1 minute tumbling windows and aggregating over groups of five, provides some smoothing compared to tumbling windows
   * Sliding window -- contains all events that occur within a fixed interval of each other, achieved by buffering events sorted by time and expiring them as they fall out of the window
   * Session window -- no fixed duration, sessionises events from a user until some inactivity is detected


#### Stream joins 


* Stream-stream joins (window join)
   * Imagine a search feature on a website and you want to detect trends in searches
   * You can log searches and log clicks
   * To calculate the click-through you must join the searches and clicks by session
   * It’s not enough to capture the search data in the click, as that only gives you insight to the searches with click-through
   * Stream processor needs to be capable of keeping state (eg. searches in the last hour, indexed by session), emitting a search click through event if a pair of events are seen before one expires
* Stream-table join
   * Thinking back to user activity and user profiles
   * Enriching the activity stream with selected profile information
   * User profile table can be copied to be local to the streaming processor to avoid round-trip times for searches on user ids, like in the batch processing case
   * Unlike batch processing, the user profile table can be updated (as its not merely a point in time replica), change data capture can solve this problem and the stream processor can subscribe to changes in the profile table as well as user activity
* Table-table join (materialised view maintenance)
   * Imagine the twitter example wherein
      * User u sends a tweet and the tweet is appended to the timeline of all u’s followers
      * Deleting a tweet removes it from all timelines
      * When u1 followers u2, recent tweets from u2 and added to u1’s timeline (and when u1 unfollows u2, those tweets are removed)
   * This is a cache maintenance problem, requiring access to streams for new and deleted tweets, following (and unfollowing)
   * The processor will need a database of the follower set for each user to know which timelines should be updated for a new tweet
   * Effectively a materialised view between a tweet table and followers table (p.475)
   * Both streams are database changelogs and the result of the stream is changes to a materialised view of the join between the two target tables
* Order of events to determine joins is important (following then unfollowing is important)
   * How to ensure events on different streams are ordered appropriately?
   * Can lead to non-deterministic joins when using historical data
   * Slowly changing dimension (SCD) is overcome by adding a unique ID to the version of a joined record (eg. sales tax rate at a point in time)
      * Makes log compaction more difficult (if not impossible)


#### Fault tolerance


* Batch process jobs can be restarted, transparent retry is made possible as inputs are immutable and outputs are only visible after success
   * Exactly-once semantics (or effectively-once semantics)
* Microbatching and checkpointing
   * Stream can be broken into small blocks and treated as a batch job (used in Spark Streaming), batch size is very small (microbatch) eg. one second
      * Implicitly a tumbling window
   * Rolling checkpoints could be written to durable storage (used by Flink)
   * Within the stream processor these approaches offer exactly-once semantics, but once the result is published it is not possible to revert a failed task (as the side effect of writing the data somewhere has occurred)
* Atomic commit revisited
   * To give proper exactly-once semantics, results must be published iff the processes were successful, this even includes acknowledging the messages to the broker
   * Atomic commit implemented in Google Cloud Dataflow, VoltDB and Kafka
      * Keep transactions internal by managing the state and messages in the stream
* Idempotence
   * Discarding failed output is one method to handle failed tasks
   * If the operation can be performed multiple times without causing the side effect to happen more than once (eg. writing a key value, but not incrementing a counter)
   * Non-idempotent messages can be made idempotent by keeping track of which messages have been seen/processed
   * A good way to achieve exactly once semantics with less overhead
   * When failing over, token fencing may be needed to prevent interference from a node that is not dead
* Rebuilding state
   * State could be kept in another data store and replicated (performance penalty)
   * State could be periodically replicated (less penalty)
   * Flink captures operator state to HDFS periodically
   * Samza and Kafka replicate state changes by sending them to a dedicated Kafka topic
   * Volt replicates state by redundantly processing each input on several nodes
   * It may not even be necessary to replicate the state if it is fast enough to replay the input events (eg. for small windows)
   * Ultimately depends on performance characteristics of your disks and network


#### Summary


* AMQP / JMS brokers -- ideal for task queues where the exact order of messages is not important and there is no need for reading old messages
* Log-based brokers -- parallelism achieved through partitioning, consumers checkpoint progress with message offset numbers, messages are retained for playback, similar to replication logs
* Representing databases as streams opens up powerful opportunities, derived systems can subscribe to changes and fresh views can consume the log from the start of time to right now




### The Future of Data Systems (a subjective look)


“If the highest aim of a captain was to preserve his ship, he would keep it in port forever”


* There are usually several solutions to a problem, each with their own pros and cons
   * Log structured storage, B-trees, columns
   * Single leader, multi-leader, leaderless
* There is unlikely to be one piece of software suitable for all circumstances in which your data will be used (eg. Majora), so you will inevitably end up cobbling different pieces of software together
* Common to integrate OLTP data with full-text indexing to handle arbitrary queries for keywords
   * Postgres has a full-text search feature, but more sophisticated alternatives may be needed for complex applications
   * Search indexes obviously not a durable system of record, so both are needed
* Becomes harder to keep derived data in sync as it becomes used in more places
   * Search index, data warehouse, batch/stream processors, caches, denomalised read-only views, machine learning models, notifications
   * Need to be clear about the inputs and outputs of each system
   * Which representations are derived from where?
   * How does data reach the right place, in the right format?
   * Easier to employ a database of record and a means of communicating changes (CDC, events) rather than allow for writing to multiple derivatives (total order broadcast)
   * Making this update deterministic and idempotent reduces the difficult in fault recovery
* Derived data vs distributed transactions
   * Distributed transactions require locks for mutual exclusion (two-phase locking) aiming for single atomic changes
   * CDC and event sourcing use a log for ordering, relying on determinism and idempotence (rather than transactions for atomicity)
   * Transactions over linearizability (allowing reading of your own writes) whereas derived systems are usually updated async and cannot offer the same timing guarantees
   * Author suggests XA in practice is poorly fault tolerant and that log based derived data is your best bet for integration across systems
   * Author suggests “eventual consistency is inevitable” is not good advice
* Total ordering limitations
   * Totally ordered event log feasible in small systems
      * Evidenced by abundance of single leader database systems
   * Requires all events to pass through a single system for ordering
      * Logs can be partitioned to increase throughput, but the ordering guarantee only applies within each log partition (not across partitions)
   * More difficult to order events across geographically dispersed data centres
      * Undefined ordering of events that occur in different data centres
   * No defined order for events generated by different microservices that do not share state
   * Clients that support offline working can cause difficulty in determining total ordering
   * Deciding on a total order is a consensus problem, usually designed for cases where a single node is sufficient for processing the entire event stream
   * Extending total order broadcast to multiple nodes is an open research problem
   * In cases where the events are not causally linked, the lack of total order is not a big problem (concurrent events only need arbitrary ordering)
   * Some cases are also easy to handle, an object updated multiple times can be totally ordered by ensuring the events are routed to the same log
      * Subtle exceptions however -- consider a user unfriending someone and then posting a rude message about them, those events need to be processed causally
* Batch and stream processing
   * Batch processing has functional programming flavour, immutable inputs without side effects, deterministic pure functions, explicit outputs
   * Deterministic functions with defined inputs and outputs makes it easier to reason about the flow of data through a system and organisation
   * Asynchrony is what allows a distributed system to be robust (as an error in one part of the system doesn’t rollback transactions across the board)
   * Batch and stream are two sides of the same coin in some ways; stream processing allows input changes to be reflected in derived views with low delay whereas batch processing allows large amounts of historical data to be reprocessed to provide new views
   * Being able to reprocess existing data provides a good mechanism for maintaining a system and evolving new functionality
      * Without being able to do this, you’re limited to schema migrations
      * Derived views allow for gradual evolution, the particular beauty of which is that every step of the transition is reversible as the old view still exists
   * Lambda architecture -- runs both batch and stream processing at the same time and records incoming data in an always-growing immutable event log
      * Read optimised views derived
      * Stream processor (eg. Storm) consumes events and produces approximate updates to the read view, the batch process later consumes the same set of events and produces a more accurate version
      * Significant additional effort to run both batch and stream systems
      * Views from batch and stream need to be merged somehow to handle queries
      * Reprocessing the entire history for the batch job is expensive, leading people to configure the batch to handle a more recent slice of time which runs counter to having a batch and stream processor
      * Liquid attempts to unify the idea of Lambda processing
      * Unified systems can replay events from HDFS or log-based brokers, exactly-once semantics for stream processors, tools for windowing by event time (not processing time)
* At an abstract level, databases and filesystems store data and allow that data to be queried
   * A filesystem won’t cope well with a directory of 10 million files but a database table with 10 million rows is normal
   * UNIX fs presents programmers with a logical but low level abstraction of storage whereas relational databases offer a high level abstract that hides the reality of data structures, concurrency, recovery etc.
   * NoSQL appears to be a movement towards the filesystem abstraction for OLTP problems
* Composing a data storage technology
   * Features
      * Secondary indexes (allow efficient searching of records based on a value)
      * Materialised views (precomputed query cache)
      * Replication logs
      * Full text indexing
   * There is a parallel between these features and the batch and stream processors
   * Creating an index is effectively snapshotting the data, reprocessing the data set to create a new derivative dataset (the index), catching up to the latest changes and then listening to new events to update the index -- just like starting a follwer replica
   * “The meta-database of everything” (sounds like the original idea behind Majora!), all batch, stream and ETL processes are keeping indexes and materialised views in one big database up to date
      * Batch and stream processors are elaborate triggers and stored procedures
   * Future
      * Unified reads through federated databases (sounds like Majora 3 so I must be on the right track!) or polystores provide a unified query interface to a variety of underlying storage engines
         * Postgres foreign data wrapper
      * Unified writes through unbundled databases, allowing for plugging and playing with CDC and event logs to sync writes over disparate technologies
   * Unified writes is a harder problem
      * Distributed transactions is the traditional approach
      * Async event log with idempotent writes is more robust
      * Ordered log of events is a simpler transaction that disparate dev/prod teams can get behind (unlike transactions)
      * Lack of standarised transaction protocol
      * Log approach allows for loose coupling
      * Log approach can buffer events in case of a failure (unlike failed transactions which escalate minor faults to large failures)
      * Teams can focus on doing one thing well and provide well-defined interfaces to other teams systems, event logs can capture strong consistency (as they can be ordered) and are general enough to use for all types of data
   * Author suggests this will move the complexity of databases to running the unbundled infrastructure rather than inside the database itself (reminder that building for scale you don’t need is wasted effort)
   * Unbundling will not compete with databases to improve performance on particular work, but to increase the range of workloads that can be done
   * The advantages will only come into play once you have outgrown a piece of software that does everything you need
   * Missing component is a high level equivalent to plug databases and other components together with the simplicity of a shell (`mysql | elasticsearch`)
* Unbundling databases by composing specialised storage and processing systems with application code is becoming known as “database inside-out”
   * A lot of learn from dataflow languages such as Oz, Juttle and functional reactive programming (FRP) languages such as Elm, logical programming languages such as Bloom
   * Jay Kreps -- “unbundling”
   * “Spreadsheets have dataflow programming capabilities miles ahead of most mainstream programming languages”
      * https://www.youtube.com/watch?v=TMIBfzSqguQ Spreadsheets are Code
* Databases struggle with custom code, outside secondary indexing things are inherently more complicated
   * Application specific transformations applied to derivative data
* Databases have turned out to be poorly suited for modern development, don't fit well with dependency and package management, version control, monitoring, integration with other services
   * But does make sense for a component to focus on durable data storage and the application worry about application code
   * Many web apps are deployed stateless (the database holds the state of the app) to allow more servers to spin up or tear down, the database is somewhat a mutable shared variable to an otherwise immutable system
   * Databases however have inherited a passive approach to mutability, unlike a spreadsheet, you need to poll the database to find if the value has changed
      * Subscribing to changes (as discussed in event streams) is only emerging as a feature now
* Instead, imagining the dataflow between application and its state lends more to a model of messaging passing but must keep in mind
   * The order of state changes is important for maintaining derived data (event log must be processed in order by all actors to be consistent), important when thinking about undelivered or redelivered messages (logs perhaps better than messages)
   * Fault tolerance essential, losing a single message will forever desync the derivative data (messages should not be kept in memory)
   * Stable message ordering and fault tolerance are hard work but less expensive and more robust than distributed transactions
   * Application works on a stream of changes and emits its own stream of changes as a result, application code is effective a transformation applied by a stream operator
* Composing of stream operators into a “dataflow system” is a lot like microservice architecture
   * Main difference is communication is unidirectional and async in a dataflow
   * One operator can subscribe to changes in another and keep a local database, negating a need for a component to make a synchronous network request to another service -- effective a stream-table join
   * Dataflow system can be faster and more robust (the fastest and most reliable network request is one without a network at all)
   * Time dependent joins must be addressed of course (eg. joining historical data)
* Stream-based updating of derived datasets gives a write path, but applications have a read path too, write path is a precomputed journey (eagerly processing input whether or not it will be read), the read path only happens if a user makes a request on that data (somewhat like lazy evaluation)
   * the derived dataset is in the middle and must balance the work to be done at write and read time
   * Shifting the boundary is work to be done on the write/read paths is key to caches, indexes and materialised views
   * Obviously not practical to precompute all possible queries, but can make decisions about effective precomputing and caching of common queries
* Modern development is contesting the status quo of stateless clients and central servers
   * JS apps can now keep a local database in the browser if they want and many user actions don’t require a round trip to a remote server to handle a request
   * Offline-first applications that occasionally sync to a remote becoming popular, opening new models for applications
   * On-device state is therefore an opportunity to push state changes to cache remote state
      * eg. websockets, eventsource server-sent events allow a browser to keep a TCP connection open and receive messages
      * “Write path” can therefore be extended all the way to a user’s device
      * Consumer offsets neatly solves the problem of catching an offline device up with missed messages
   * Facebook’s chain of React, Flux and Redux handle internal client-side state by subscribing to events of user input and responses (similar to event sourcing)
   * Assumptions of stateless clients is wired in to many databases, libraries, frameworks and protocols
      * Need to move away from request/response and more towards publish/subscribe
* “Reads are events too”
   * Subscribe to reads as a stream-table join and emit the response as a stream
   * Knowledge of reads would allow you to reconstruct what a user saw in the application (perhaps prior to making a decision)
   * Allows insight to causal dependencies
   * May add overhead but you’re probably logging the request anyway
   * Adds possibility for distributed execution of complex queries
   * Internal query systems of MPP databases may provide this rather than rolling your own stream processors to do it, but treating queries as streams may still provide interesting avenues for your system
* Very difficult to determine whether it is safe to run an application at a particular isolation level or replication configuration, leading to subtle bugs
   * If your application can tolerate occasional corruption or lost data, everything is easy
   * If not, serializability and atomic commits come at a cost
   * Of course, if the application has a bug that writes bad data or delete records, isolation levels are not going to save you
      * Argument towards immutable append-only data that can always be corrected
   * Immutability isn’t a cure-all, exactly-once semantics necessitate idempotence (so accounts aren’t charged twice, or metrics aren’t inflated)
   * Need to ensure requests are idempotent through several hops of network
      * What if the POST times out, what if the TCP connection is dropped, connections between app and database, database and client, client and user
      * Retries may be viewed as a separate request
      * End-to-end flow of request must be uniquely identified (eg. hash of inputs, hidden form field)
   * https://dsf.berkeley.edu/cs286/papers/quicksand-cidr2009.pdf
   * End to end argument -- Saltzer, Reed and Clark 1984
      * Applies to duplicate suppression (end-to-end transaction ID), integrity checks (end-to-end checksums), security( end-to-end encryption)
      * Low level reliability features are not sufficient in of themselves for end-to-end correctness
* Transactions are a useful abstraction but the author believes it is not the right one
   * End up quite expensive especially in distributed systems, ends up with fault tolerance needing to be built into application code
* Uniqueness constraints are a consensus problem
   * Solved if all requests can go through a single node to determine consensus, single leader is in charge of making decisions (if you need to tolerate the leader failing, you have another consensus problem on your hands)
   * Uniqueness checking can be scaled out by ensuring requests for a particular ID go to the same node
      * Async multi-master is ruled out as you can’t immediately reject writes that would violate a constraint without sync
   * Logs ensure consumers see messages in the same order
      * Clients can send a request and watch an output stream for a success/fail event to be emitted -- similar to implementing linearizable storage with total order broadcast
      * Potentially conflicting writes are routed to the same partition and read in a total order
   * What about cross-partition logs? (request stream, send/recv account stream)
      * Stream processor reads transaction log stream
      * Uniquely identifiable transaction added
      * Processor emits debit and credit events
      * Other streams read debit and credit events (checking for duplicates) and apply changes
      * Avoids the need for distributed transaction, log is durably written and the debits/credits are derived
      * Crashes are deterministic as each processor knows its message offset, and transactions are uniquely identified so will not be double-handled
      * Additional checks could be added at the start before credit/debit events are emitted (eg. checking balances)
      * End-to-end correctness achieved without atomic transactions
         * All changes are collapsed into single object commits
* Transactions are nice in that they are linearizable, a writer waits until the transaction can be committed and then its write is immediately visible to all readers
   * Unbundled operations are however async by design, so a writer cannot wait to determine if its write has been finished by all consumers
   * Clients can however wait for messages in a stream (eg. the username request)
   * Author argues that consistency conflates two issues:
      * Timeliness -- ensuring users observe up-to-date state
         * CAP cites consistency in the sense of linearizability
         * Weaker timeliness properties like read after write can be useful
         * Waiting and trying again is the worst that will happen if timeliness is violated
      * Integrity -- absence of corruption
         * No data loss, contradictory or false data
         * An index must be a full index
         * Explicit checking and repair is needed if integrity is violated
         * In ACID, consistency is some application-specific notion
      * Violations of timeliness are “eventual consistency” whereas violations of integrity are “perpetual inconsistency”
      * Violations of timeliness are merely annoying and confusing
   * ACID attempts to address timeliness and integrity at the same time
   * Event based dataflow systems decouple timeliness and integrity
      * Fault tolerant delivery and exactly-once semantics can achieve integrity without guaranteeing timeliness
      * Representing the write operation with one single atomic message (CDC, event sourcing), deriving all state from that single message with deterministic functions, using end-to-end request IDs to enable idempotence, immutable messages and fully recoverable derivative data
* Uniqueness may not be as strong as you need -- maybe an apology is fine
   * These cases still require integrity (dont’ want to lose orders) but timeliness checking on the checking the constraint is not needed 
   * Compensating transaction -- allow double booking, but message the user and ask them to adjust their transaction
   * Apology workflow -- if you sell more items than you have, order more and apologise for the delay (this is the same thing that would happen if stock was damaged in the warehouse)
   * In some cases double booking is part of many business models (flights, theatres) with compensation processes in place to handle the case of demand exceeding supply
   * Banks can charge overdrawn accounts rather than avoiding them, by limiting the total withdrawals in a day the bank can reduce risk
* ...so it seems we can maintain integrity on derived data without atomic commits, linearizability and synchronous co-ordination… and many applications are fine with loosely enforced constraints as long as integrity is preserved
   * That is, there is no need to pay the cost of co-ordination or enforcing strict constraints for parts of the application that don’t need it
   * Coordination and constraints reduce the number of apologies you have to make as a business at the expense of performance and availability (potentially increasing apologies…)
* Our system model makes some assumptions about things that don’t go wrong (eg. CPU calculates things correctly, data is not lost after fsync)
   * Tend to take a binary view of what things will and won’t happen whereas its more a question of probabilities
   * If you have enough users then eventually you will see the rarest problems
      * Rowhammer -- pathological memory access patterns flipping bits in other bits in memory… “hardware isn’t the perfect abstraction it may seem”
   * Even widely used software has bugs (MySQL failing to update an index) and that’s assuming you’re using the database correctly (another source of faults)
   * Consistency only works if your transaction is actually consistent (eg. using the right database configuration for isolation)
   * Auditing is not just for finance! Trust, but verify...
      * HDFS and S3 constantly check disks and assume they are not trustworthy to avoid silent corruption
      * Use backups from time to time
      * Author fears claims of ACID has caused people to blindly trust databases without auditing
   * Event based systems provide much better auditing than mere transaction logs (how to determine why an INSERT or DELETE happened from the DB log alone?)
      * Events can be checksummed, derivative data can be regenerated and compared
      * Easier to work out why a system did something in the first place
      * Checking a data pipeline end to end therefore implicitly checks disks, networks, external services and the algorithms inbetween
      * Auditing should mean you can be more confident in making changes, and assist evolvability
   * Cryptographic auditing relies on Merkle trees
      * Cryptocoins are garbage (emphasis mine) but the integrity checking aspects are interesting
      * Trees of hashes that can efficiently determine if a record exists in a dataset
      * Certificate transparency uses Merkle trees for TLS/SSL
* We talk about data in the abstract, but that data is about people and should be treated with respect, users are humans too!
   * ACM code of ethics rarely discussed, let alone applied
   * Engineers cannot ignore the consequences of the applications they build
   * Sentencing people to “algorithmic prison” by predicting their behaviour (rightly or wrongly), algorithms can assume someone is guilty without proof or appeal
   * “Machine learning is like money laundering for bias” -- consider postcodes and IP addresses as proxies for race. if the past is discriminatory, so is the training data
   * Who is accountable for incorrect decisions made by a machine?
   * A blind believe in the supremacy of data for making decisions is not only delusional, it is positively dangerous
   * When services become good at predicting content users like, you end up with a feedback loop (cf. politics on facebook)
   * Systems thinking -- does this system reinforce and amplify existing differences between people?
   * When user’s input data for a service, the relationship with data is clear. What happens when that user is tracked and logged as a side effect? The service has interests of its own (potentially conflicting with the user’s own interest)
   * As a thought experiment, replace the word “data” with “surveillance”...
      * In our rush towards the internet of things there is a microphone in every room
   * Users have little idea what data they are giving away and how it is retained and processed and cannot give informed consent
      * Worse still when data is combined from multiple sources
   * There can be a social cost to not using services and it's a privilege for people who can afford to miss out on personal or professional opportunities, for some the surveillance is inescapable
   * Privacy is a spectrum for people to decide for themselves, but when data is extracted from people that decision is transferred to the data collector
   * Behavioural data sometimes framed as “data exhaust” to make it seem like waste that is harmless to collect, in many causes its actually the fuel
   * User data should be treated like a hazardous chemical, with processes for reporting and handling spills
   * The transition to the information age is somewhat like the industrial revolution before safeguards
   * Bruce Schneier -- data is the pollution problem of the information age and protecting privacy is the environmental challenge
   * Purged data can run counter to immutability, but one solution is cryptographic control (rather than policy)
* Data integration problem can be addressed by using batch processing and event streams to *flow* between different systems
   * Designated systems of record
   * Other data is derived through transformations on the system of record
   * Recasting the idea that dataflow applications are unbundling the concept of a database
   * Derived state updated by observing changes, which in turn can emit changes that can flow all the way to an end-users device
   * End-to-end idempotent, async requests can achieve integrity
      * Clients can wait to hear back or go ahead and apologies if a constraint turns out to be violated
      * Avoiding most coordination trouble, even in the presence of faults
   * Auditing for verification and evolvability
   *
