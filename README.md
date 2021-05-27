# Designing Data Intensive Applications
Notes from the book by Martin Kleppmann

Chapter - 1: Foundations of Data Systems
--------------------------------------------------
### 3 design principles of software applications
- Operatability (Make the system easy to operate)
- Simplicity (Make the system easy to understand)
- Evolvability (Make the system extensible)

**big ball of mud** => Software built with much complexity

### Simplicity
---------------------
#### Avoid

- State space explosion
- tight coupling of modules
- tangled dependencies
- inconsistent naming
- hacks
- special-caseing to work around issues elsewhere.

#### Use

- Abstraction
	- Programming languages are abstractions that hide machine code
	- SQL abstracts away complex on-disk and in memory data structures.

### Evolvability
--------------

- TDD and Refactoring
- Measure of ease to change system design from one approach to another.
- Agility


CHAPTER 2 -- Data Models and Query Languages
============================================

**Clean data model** => a layered approach where each layer above hides the complexity of the layer below it.

- Relational Data Model (SQL)
- Document Data Model (noSQL)
- Graph Data Model

**SQL** => Relations => Tables in SQL. => unordered collection of tuples called Rows.

**NoSQL** => Not-only SQL.
- Greater scalability then RDBs.
- Used for huge datasets
- Very high write throughput.
- Specialized queries not supported by RDBs.
- More dynamic and expressive data models.

**Polygot Persistence** => using both RDBs and DDBs in tandem.
**Impedence Mismatch** => Using RDBs with Object-oreinted programming. => RDBs returns data that needs to be converted into objects. => ORMs help to reduce it.

- 1-to-Many relationships in RDBs => Using unique values in one table and and creating separate tables for the many values and referencing the unique value using a Foreign key in the many tables.
- With JSON it becomes easier to store these 1-to-many relation in a single object. For example a persons resume can be represented with a single object.
- Store non changing data like locations as IDs. It is easier to reference and remain consistent throught the application. Also allows search via dropdowns.
- Use IDs to reference data. That way we just need to change the item the ID references in one place instead of everything that is referencing the ID.
> - REDUCE DUPLICATION OF DATA => REDUCE WRITE OVERHEADS => REDUCE INSCONSISTENCIES
> - NORMALIZATION => DE-DUPLICATION OF DATA. => one-to-many or many-to-one.
> - DE-NORMALIZATION => DUPLICATION OF DATA. => Needed sometimes for complex many-to-many relationships
- Normalization requires creating many-to-one relationships => Many people live in one place. Many people work in a particular industry. => Works well with RDBs (Joins) not DDBs (weak Joins).
- As application grow eventually data transform into many-to-many relationships.


**RDBs** => Query optimzer => Automatically decides which parts of the query to execute in which order, and which indexes to use. => Access path.
**Cardinality** => Uniqueness of data in a column. Lower the cardinality lower the uniqueness. For example, columns that contain booleans have a cardinality of 2. UUID columns have high cardinality. 
It makes no sense to *Index columns with low cardinality*.

**FOREIGN KEY** => Used for many-to-one or many-to-many relationships to reference data in different columns based on a unique identifier.
- For RDBs, it's called Foreign key.
- For DDBS, it's called document reference.

#### DDBs Pros:
- Schema Flexibility
- Better Performance
- Object Oriented.
- Preferred when our app has a tree of one-to-many relationships.

#### RDBs Pros:
- Better JOIN support.
- Better many-to-one and many-to-many relationships.
- MySQL is really bad as DB migrations. The alter commands copies the entire table.
- MySQL does not support XML.


Databases use Declarative query language.
SQL is declarative also called relational algebra.
Means, you tell it what results you want, but not how to achieve the results. Allow the DB to optimize on its own.
CSS and HTML are also declarative.

Normal programming languages are imperative. You have to tell the code what to do and how to do.


Graph Databases
---------------------

- Specialized for many-to-many relationship models.
- A graph consists of two kinds of objects:
	- vertices (also known as nodes or entities) 
	- edges (also known as relationships or arcs). 

#### Graph models
- Property graph model
- Triple store model
- and more....

**Declarative query language for graphs.**
- Cypher (Created for the Neo4j graph database). Has nothing to with cryptography.
- SPARQL
- Datalog

Graph database are highly extensible. They have good evolvability


CHAPTER 3 -- Storage and Retrieval
==================================

## Log Structured Indexes

#### Hash Indexes
- Lookups based on key value pairs called hash maps.
- Hash maps are stored in memory. => applicable when we have a small number of distinct keys. 
- Indexing increases read speeds but reduce writes speeds because the data has to br written twice.
- Adding additional indexes results in storage overhead. => can be minimized by splitting into segments of specific size.
- These segments can then be compacted, and merged (based on similarities).
- After merging segments redundant segments can be deleted.
- Hash index's map keys to byte offsets. 
- Range queries in hash maps don't work.

#### Storage Engines
- Bitcask storage engine is useful when the value for each key is updated frequently but the key does not.
- Like the view counter of a video.
- Key: url, Value: count.
- Because if Bitcask we need to store the keys in memory and there are not a lot of keys so it is feasible. 

### SSTables and LSM-Trees
--------------------------------------
**SSTable** => sorted string tables
**memtable** => In memory table.
**LSM - tree** => Log structured mega tree.

^ these are all log structered file systems.
Storage engine based on segmentation, and then compacting and merging are often called LSM storage engines.

Lucene is an indexing engine used by ELK and Solr uses a similar engine.

LSM-tree algo is slow when looking up keys that don't exist. Everything has to be searched starting from latest to oldest.
Bloom filters can be used to make this faster^

Sequential vs Random writes to disk => sequential is better. high write throughput.
Random writes follow => write-seek-write-seek-pattern. each seek takes 10ms. 
sequential writes takes 30ms to write 1mb of data. Sequential is faster.

## B-Trees
- Most widely used indexing structure.
- Standard for almost all RDBMS and NoSQL DBs.
- It is on disk not in memory.
- B-trees keep key-value pairs sorted by key, which allows efficient key-value lookups and range queries.
- B-trees break the database down into fixed-size blocks or pages, traditionally 4 KB in size (sometimes bigger), and read or write one page at a time.
- ^Similar to how data is written to disks.
- Pages are lined to each other based on references originating from the root page. Each page can have multiple pages linked to it.
- The number of pages that can be linked is known as "Branching Factor". It is usually in the hunderds.
- A B-tree with n keys always has a depth of O(log n)
- Most databases can fit into a B-tree that is three or four levels deep, so you don’t need to follow many page references to find the page you are looking for. 
- A four-level tree of 4 KB pages with a branching factor of 500 can store up to 256 TB.

### B-Tree Resilience
- If a page is being written and the DB crashes the index can become incosistent and we would have a dangling page on our hands with no ref.
- In order to make the database resilient to crashes, it is common for B-tree implementations to include an additional data structure on disk: a write-ahead log (WAL, also known as a redo log).
- This is an append-only file to which every B-tree modification must be written before it can be applied to the pages of the tree itself. When the database comes back up after a crash, this log is used to restore the B-tree back to a consistent state
- Also careful concurrency control must be implemented when multiple threads are accessing the B-Tree. 
- The tree is protected by "latches" in this case that are light locking mechanisms.

>Generally, 
Faster Reads == B-Trees
Faster Writes == LSM-Trees

