
Reading 1 - 'What Goes Around Comes Around' [pdf](https://people.cs.umass.edu/~yanlei/courses/CS691LL-f06/papers/SH05.pdf)
	Sections 1 - 4
- Later proposals of data models are similar to earlier proposals (hence the name)
Part I
- There are nine 'eras', ending (in this book) in 1990s being XML
- standard example: supplier, part, supply
	- supplier: (sno, sname, scity, sstate)
	- part: (pno, pname, psize, pcolor)
	- supply: (sno, pno, qty, price)
Part II
- IMS - released ~1968
- hierarchical data model
- notion of a **record** type - collection of named fields with their associated data types
- each **instance** of a record type is forced to obey the data description (schema)
- some subset of record fields must uniquely specify a record instance
...
skipping ahead to this reading: [pdf](https://dsf.berkeley.edu/papers/fntdb07-architecture.pdf)
- on DBMS - closer to what I'm interested in studying
# DB Architecture
### 1.1 - p.3
- RDBMS are the most widely used DBs today (relational database management systems)
- 5 main components - seen by the life of a query
- ex: (at an airport) agent clicks on the button to submit a form to request the passenger list of a flight
	1. **COMMUNICATION MANAGER**: the client calls API that communicates over the network to establish a connection with the *Client Communications Manager* of a DBMS. 
		- In some cases, the connection is established directly (via ODBC or JDBC connection protocols)
			- this is a 'two-tier' protocol (just meaning 'client-server').
		- In other cases, there is a middle-tier server, called a 'three-tier' system.
			- this might be a web server, transaction monitor, or so on
			- There may even be another layer (application server)
		- the DBMS needs to be able to handle these many different protocols
		- the base goal is the same: 
			- establish and remember the connection state for the caller (client, middleware, etc), 
			- respond to SQL commands from the caller, 
			- and to return both data and control messages (result codes, error codes, ...) as appropriate. 
		-> In this case, the communication manager would establish the security credentials of the client, set up the state to remember the details of the connection and the current SQL command across calls, and forward the client's first request deeper into the DBMS for processing
	2. **PROCESS MANAGER**: Upon receiving the first SQL command, the DBMS must assign a 'thread of computation' to the command. 
		- It must also make sure that the thread's data and control outputs are connected via the communications manager to the client. 
		- The most important decision the DBMS makes at this stage wrto the query is *admission control*. 
			- whether the system should begin processing the query immediately, or defer execution until there are enough resources available for the query
			- (section 2 for more detail)
	3. **RELATIONAL QUERY PROCESSOR**: Once admitted, the query can begin to execute. It invokes the RQP.
		- this set of modules checks that the user is authorized to run the *specific query*, and compiles the SQL query text into an internal *query plan*.
		- Once compiled, the query plan is handled via the *plan executor*
			- which consists of a suite of 'operators', (relational algorithm implementations) for executing any query.
			- typical operators implement tasks like joins, selection, projection, ...
			- also implements calls to request data records from lower layers of the system
		- a small subset of these operators are used for our example query
		- (section 4 for more detail)
	 4. **TRANSACTIONAL STORAGE MANAGER**: at the base of the query plan, one or more operators exist to request data from the database. These fetch data from the TSM, which manages all data access/(reads) and manipulation/(create, update, delete...) calls.
		 - TSM includes algorithms and data structures for organizing and accessing data on disk ('access methods') such as tables and indexes
		 - includes a buffer management module that decides when and what data to transfer between disk and memory buffers
		 - in our example, in the course of accessing data, the query must invoke transaction management code to ensure 'ACID' properties (section 5.1). 
		 - Before data access, locks are acquired from a lock manager to ensure correct execution in the face of other concurrent queries.
		 - if the query involved updates, it would need to interact with the log manager to ensure that the transaction is durable if committed and fully undone if aborted.
		 - (section 5/6 for more detail)
	 5. **UNWINDING THE STACK**: At this point, data has begun to be accessed, so it is ready to compute results for the client. This is done by 'unwinding the stack' of activities described above.
		  - Access methods return control to the query execution operators, which orchestrate the computation of result tuples from database data
		  - as result tuples are generated, they are placed in a buffer for the Client Communications Manager, which ships the results back to the caller.
		  - for large result sets, the client will typically make additional calls to fetch more data incrementally from the query, resulting in multiple iterations through the communication manager, query executor, and storage manager. 
		  - in our example, at the end of the query the transaction is completed and the connection is closed.
			  - this results in:
				  - transaction manager cleaning up state for the transaction, 
				  - process manager freeing any control structures for the query
				  - and communication manager cleaning up communication state for the connection
- This example covers many components of the RDBMS, but not all of them
- There are a number of shared components and utilities used throughout the operation of a full-functioning DBMS
	- catalog and memory managers are utilities during a transaction
		- catalog: used in authentication, parsing, query optimization
		- memory: used whenever memory needs to be dynamically allocated or deallocated
	- the remaining modules are utilities running independent of any particular query, keeping the database as a whole well-tuned and reliable
	- (discussed in section 7)
### 1.2 - p.8
- scope: the core fundamentals of DB functionality
- not comprehensive on DB algorithms
- minimal discussion regarding extensions of dbs
- topics of interest may be noted however
- starting by investigating the overall architecture of database systems
- in any server system architecture we want to review it's overall process structure and discuss some viable alternatives
	- first for uniprocessor machines
	- and then for parallel architectures
- then we go into the domain-specific components of a DBMS
- then storage architecture and transactional storage management design
- finally, some shared components and utilities with DBMS
## 2 - p.9 Process Models
## Preface
- Any multi-user server requires early decisions on the execution of concurrent user requests and how these are mapped to operating system processes or threads.
- these decisions have a profound influence on the software architecture of the system, and on it's performance, scalability, and portability across operating systems
- assumptions: good OS support for threads, only targeting uniprocessor systems (for now)
	- OS: defines private address spaces and control threads for processes
	- OS Thread: program execution unit without additional private OS context and without private address space - has full access to memory of other threads within the same multi-threaded OS process; commonly called 'kernel threads' - scheduled by kernel
	- LWT Lightweight Thread Package: application-level construct supporting multiple threads within a single OS process. Unlike OS threads, lightweight threads are scheduled at the application level thread scheduler and the kernel has no knowledge or personal involvement
		- faster context switch (no OS kernel mode), but blocking operations (I/O,..) of one lightweight thread will block ALL threads in that process
		- avoided by only issuing async I/O requests, and not invoking OS operations if possible
- some DBMS implement their own LWT packages
- DBMS client is the software component that implements the API used by application programs to communicate with a DBMS. 
	- some example database access APIs are JDBC, ODBC, OLE/DB.
- there is a wide variety of proprietary API sets
- Some programs use embedded SQL, a technique mixing programming language statements with database access statements
	- first with IBM COBOL and PL/I, later with SQL/J (Java+SQL)
	- Embedded SQL is processed by preprocessors that translate the SQL statements into direct calls to data access APIs.
	- these protocols that are used are often undocumented
- DBMS worker thread is the Thread of Execution in DBMS that does work on behalf of the DBMS client. A 1:1 mapping exists between a DBMS worker and DBMS client: 
	- the worker handles all SQL requests from a single client
	- the client sends SQL requests to the DBMS server, and the worker executes those requests
### 2.1 - p.12
- Uniprocessors and lightweight threads
- this forms the basis from which discussion can occur about current generation production systems
- each leading system is at it's core an extension or enhancement of these core model/s
- Two simplifying assumptions
	1. OS thread support: a process can have a large number of threads, memory overhead is small, context switches are inexpensive (generally actually true today, but this wasn't always this case)
	2. Uniprocessor hardware: we are designing for a single machine with a single CPU - this is unrealistic, but it simplifies initial decision making
- in this context, a DBMS has three natural process model options (simplest -> most complex)
	1. process per DBMS worker
	2. thread per DBMS worker
	3. process pool
- Although these are simplified, all are used by commercial DBMS systems today.
#### 2.1.1 - p.13
- process per DBMS worker
- used very early on, still used by many commercial systems today
- relatively easy to implement (workers map directly onto OS processes)
- The OS scheduler manages the timesharing of these workers and the DBMS programmer can rely on OS protection to isolate standard bugs like memory overruns
- Various programming tools like debuggers are well-suited in this model
- what makes it complicated
	- in-memory data structures shared across DBMS connections including the lock table and buffer pool (section 6.3, 5.3 respectively)
- must be explicitly allocated in OS-supported shared memory accessible to all DBMS processes
- This requires OS support (widely available) and some special DBMS coding
	![[Pasted image 20250917150043.png]]
	- each DBMS worker is implemented as an OS process
- in practice, the required extensive use of shared memory reduces some advantages of address space separation given that a lot of 'interesting' memory is shared across the processes
- In terms of scaling, process per DBMS worker is not a very attractive model. Scaling issues arise because a process has more state than a thread and consequently consumes more memory
	- A process switch requires switching security context, memory manager state, file and network handle tables, and other process context
	- not needed in a thread switch
- nonetheless, the process per DBMS worker is popular, and is used by IBM DB2, PostgreSQL, and Oracle.
	- some reading on this wrt Postgres [reddit](https://www.reddit.com/r/PostgreSQL/comments/t5ahe9/why_does_postgres_use_1_process_per_connection/) [wiki](https://wiki.postgresql.org/wiki/Todo#Features_We_Do_Not_Want)
	- note: wiki entry removed so may have changed
	- tldr; eliminates process protection, in modern systems thread creation is similar in overhead to process creation, MySQL and DB2 have demonstrated many issues caused by the thread-per-worker model
#### 2.1.2 - p.14
https://dsf.berkeley.edu/papers/fntdb07-architecture.pdf p 14