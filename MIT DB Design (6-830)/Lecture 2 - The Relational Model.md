
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
## DB Architecture
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
- 
