**A brief overview of transactions, ACID properties and the challenges we would face in our project**

Whenever we are making a system that involves booking (flights, trains, movies) etc, we face some challenges like:

1. Same seat selection: If two different users are booking the same set of seats concurrently, it can lead to problems.
2. Race condition for a seat: If let's say only one seat is remaining, and there are two different users who are concurrently booking, it can lead to race condition, as to who will get the seat booking.

There can be certain other types of scenarios as well, discussed below:

![Challenges in Booking System](./Challenges_in_Booking.PNG)

Situation 1: The booking service initiates a request from its end, but the request couldn't reach the payment gateway due to some reasons.

Situation 2: The booking service initiates a request to the payment gateway, and the payment fails at the payment gateway due to some reasons.

Situation 3: The booking service initiates a request to the payment gateway and the payment is successful at the payment gateway. The payment gateway intiates a successful response, but it couldn't reach the booking service due to network issues or malformed reponse object. There is the chance that the user might reinitiate the request in such case, and the money will be deducted twice.

Situation 3: The booking service initiates a request to the payment gateway and the payment is unsuccessful at the payment gateway. The payment gateway intiates a successful response, but it couldn't reach the booking service due to network issues or malformed reponse object.

**Let's see how some of the existing booking systems handle transactions and solve concurrency issues**:

If we simultaneously open a booking service in two different browsers, and try to initiate a booking for the same set of seats, we can see that when we initiate the booking from the first user, the booking service temporarily holds those seats for the first user. This temporary booking will be held for some duration of time (let's say 10 minutes), depending on the business logic. If at the same time, the second user tries to initiate the booking, those seats will be shown as unavailable to that user.

If the first user makes the payment and the booking succeeds, then those seats are permanently blocked for the first user. However, if the first user doesn't make the payment within the stipulated time, then the seats are no longer held for the first user and are made available to all users.

![Concurrent Bookings](./Concurrent_Bookings.png)

![Temporary_lock_acquired_on_bookings](./Temporary_lock_acquired_on_seats.png)

![Session_timeout](./Session_timeout.png)

![Transaction_state_diagram](./Transaction_state_diagram.png)

**What are database transactions?**

In real life situations, we need to execute a series of queries in order to accomplish a task. We might be doing a bunch of CRUD operations. These series of steps comprise a single unit of work and are referred to as a transaction.

During the transaction execution, the DB might go through a lot of changes and can be in an intermediate inconsistent state.

In order to handle transactions, we have ACID properties.

**Let's understand how race condition can occur in transactions, with the help of an example**

Let's say we have the following situation:

A and B both have 1000 cash in their account.  A wants to transfer 100 cash to his friend B. Also, the transaction to credit 6% interest in A's account and B's account is also to be initiated.

Now these two transactions can be run in different ways: serially, or with different interleaving methods.

Let's see initial state of the database: A's balance: 1000+6% interest= 1060
                                         B's balance: 1000+ 6% interest= 1060
                                        Total balance: 2120

First, let's see what happens if these two transactions are run serially:

![Serial execution of transactions](./Serial_execution_of_transactions.PNG)

In case 1, A's final balance is (1000-100) * (1.06)=954, and B's final balance is (1000+100) * (1.06)= 1166, total balance= 2120, so the database is in a consistent state before and after the transaction.

In case 2, A's final balance is (1000 * 1.06-100) = 960 , and B's final balance is (1000 * 1.06 + 100) = 1160, total balance= 2120, so the database is in a consistent state before and after the transaction.

If all the transactions in a database are run in a serial manner, then the database will always be in a consistent state. But it's not feasible to run all the transactions in serial manner, because the number of transactions in real life systems are so large that we need to have parallel processing on multiple cores of the CPU.

Let's consider some cases of parallel execution:

![Parallel Execution of transactions](./Parallel_execution_of_transactions.PNG)

In case-1, A's final balance is going to be (1000-100) *  (1.06)=  954 and B's final balance is going to be (1000+100) * (1.06)= 1166, hence the total balance will be 2120. Hence the database is in a consistent state before and after the transaction.

In case-2, A's final balance is going to be (1000-100) * 1.06 = 954 and B's final balance is going to be 1000 * 1.06+ 100= 1160, total balance is 2114. The database is not in a consistent state in this case after during the transaction.

Reason why database is not in a consistent state in this case: The 100 cash which was to be transferred from A to B, neither A nor B got interest on that amount.

**Let's talk about the different execution anomalies**

- Read Write conflict:  

It is also known as unrepeatable read. In this type of conflict, a transaction reads the value which was written by some other committed transaction. As a result, the transaction on reading, finds a different value than what it got, by reading before.

![Example](./Read_Write_Conflict.png)

- Write Read conflict:

It is also known as dirty read problem. In this type of conflict, a transaction reads the value which was written by an uncommitted transaction. 

![Example](./Write_read_conflict.png)


- Write Write conflict: 

It is also known as lost update problem. In this conflict, one of the transactions overrides the value written by other transaction.

![Example](./Write_write_conflict.png)

**Atomicity**

A transaction is a bunch of statements that intends to achieve one final state. When we are attempting a transaction, we either want to complete all the statements or none of them. We never want an intermediate state. This is called as atomicity.

A transaction can only be three states: BEGINNING, COMMITTED OR ROLLBACK. (Beginning is when the transaction is just initiated, Committed is when the transaction is successful and we are saving the changes in the database, Rollback is when the transaction is unsuccessful and the progress made so far is undone).

**How do databases ensure atomicity?**

There are two ways by which databases ensure atomicity:

1. Logging: In this method, the DBMS logs all the actions that it is doing. The DBMS can later on undo those actions in case the transaction is not successful. These logs can be maintained in the memory or the disk, depending upon the type of DBMS.

Logging is the more preferred approach than shadow paging. As an example, MySQL uses logging for ensuring database atomicity.

2. Shadow Paging: In this method, the DBMS creates copy of the database. All the instructions of the transaction are performed on this copy. If the transaction is successful, then this new copy is kept, else the original copy is used. Example:  CouchDB, Open LDAP use shadow paging.


**How does MySQL ensure atomicity, in particular?**

-> As a relational database, MySQL ensures that the database is in consistent state after every COMMIT or ROLLBACK.

-> Handling COMMIT is easy, not a big deal.

-> In order to handle ROLLBACK, MySQL uses two types of logs: undo logs and redo logs.

-> First things first, let's understand that ROLLBACK happens because of two reasons: either something was wrong in the transaction or there was a system crash. Similar to how an employee can be terminated because of misconduct or organsiation restructuring.

-> Undo logs: This log contains information about how to undo the last changes done by a transaction. As an example, if we created table during transaction and in the mid of the transaction, we need to rollback, then redo logs will have the instructions to drop the table.

-> Redo logs: These logs are useful in the case of a system crash. They store the information to execute the instructions of transaction till the point it had executed, before the system crashed. These instructions can be executed again when the system boots up. 

Redo logging is a disk based data structure, which is used for crash recovery to correct data written by incomplete transaction. The changes which could make it to the files before the crash or any other reasons are replayed automatically during restart of system after crash.

As a simple example, if I type 'Hello World' in MS Word application, undo logs would store the instructions to delete these characters one by one from 'Hello World', whereas the redo logs would store the instructions to type the text 'Hello World'.

**Let's talk about isolation in the terms of databases**

Referring to the wikipedia page, Isolation refers to the degree to which the transaction integrity is visible to other users and systems.
 
 A lower isolation level increases the ability of many many users to access the same data at the same time, but it can lead to concurrency effects such as dirty reads (read write conflict) or lost update problem (write write problem)

 On the other hand, a higher isolation level reduces the the concurrency effects, but it requires more system resources amd increases the chance that one transaction will block another (because one transaction will block another transaction from reading/writing).

Let's talk about some of these concurrency issues:

1. Dirty Read (in simple terms, when a reader is reading a dirty value) : When a transaction reads the uncommitted value from another transaction, it is called as dirty read problem.

2. Lost update problem (in simple terms, the updates from one of the transaction is lost due to being overwritten):  An update done to a data item by a transaction is lost as it is overwritten by the update done by another transaction. 

3. Unrepeatable read problem (In simple terms, the read can't be repeated because on different reads we get different values of attributes of a record) : A non-repeatable read occurs when a transaction retrives a row twice and that row is updated by another transaction that is committed in between.

4. Phantom read problem (In simple terms, a phantom (ghost) row appears between two consecutive reads in the database ): A phantom read occurs when a transaction reads the set of rows twice, and new rows are inserted or removed from the database by another transaction, which commits in between.

See this wiki for examples: [Wiki on Isolation](https://en.wikipedia.org/wiki/Isolation_(database_systems))

Important differences:

Q. What is the difference between unrepeatable read and phantom read?

A. Unrepeatable read is when the attribute in a record changes upon the next read. In case of a phantom read a new row is created or deleted upon the next read.

Q. What is the difference between dirty read and unrepeatable read?

A. In dirty read, the transaction is reading the uncommitted changes from the other transaction. However, in the case of a unrepeatable read, the transaction is reading committed changes from another transaction.

Let's discuss about the different isolation levels (again from the wiki):

1. READ UNCOMMITTED isolation level: 

- This is the lowest level of isolation. 
- A transaction can read the latest uncommitted value from another transaction.
- Dirty reads are possible in this isolation level.
- READ UNCOMMITTED is suitable for usecases, if there are frequents write to a table than the reads (an example can be a user registration table). The probability of reading same rows in such database would be extremely low.

2. READ COMMITTED isolation level:

- In the READ COMMITTED isolation level, there is a guarantee that any data read will be committed data only, and hence the problem of dirty read doesn't occur.

- However, there is no promise that the subsequent reads will give the same result, hence the problem of unrepeatable read and phantom read is still there in the READ COMMITTED state.

3. REPEATABLE READ isolation level:

- In this level, the READ operations can be repeated over time. This is possible because of the fact that a snapshot of the SELECT command is used throughout the transaction when the same SELECT is executed.

- However, this isolation level doesn't ensure that new records won't be created or deleted from the database, hence the problem of phantom read can still occur.

4. Serializable isolation level:

- Highest level of isolation. It completely isolates the effect of one transaction from another one. 

- Peformance of the system would be low.

Let's talk about durability in the context of DBMS.

Durability: The effect of the committed transactions will persist in the database, even if the system crashes or fails.

Discussion about storage engine: 

Q. What is a storage engine?

A.  Storage engine is a software module that a database management system internally uses to create, read, update data from a database. There are two types of storage engines in MySQL: transactional and non-transactional. For MySQL, the default storage engine is InnoDB.

InnoDB is an ACID compliant storage engine.

**Consistency**

Consistenncy in InnoDB involves protecting data from crash and maintaining data integrity and consistency.

In order to ensure consistency, InnoDB has two mechanisms:

1. Double Write buffer
2. Crash Recovery

Before we can understand double write buffer, we should understand some fundamental terms first:

1. Page (in the context of databases) : Page refers to how much data of the database is transferred from the disk (hard disk) to the memory (RAM) in one go. A page can contain 1 or more rows. If the data is too large to fit in one page, then InnoDB stores a pointer to the required data.

2. Flush (in the context of databases): It  refers to writing contents to a disk.

When we write something to the database, it is not written instantly, dor performance reasons in MySQL. It instead stores that either in memory or temporary disk storage. These contents are later on flushed to the disk.

Let's try to understand Double Write Buffer:

Problem: If there's a power failure when writing the pages from InnoDB to the disk, it will lead to half-written pages.

Solution: Write the pages from InnoDB to the double write buffer first. Only when the page is safely flushed from the buffer to the disk, will the InnoDB write the page to the disk.

If in case there is a power failure, then the page written to the disk and the page stored in the buffer are compared.

Reference: [Documentation](https://mariadb.com/kb/en/innodb-doublewrite-buffer/)

Coming to the crash recovery, InnoDB maintains logs to ensure consistency in the database.

**Let's discuss how we will be ensuring ACID properties in our project**

In order to handle transactions, Sequelize provides us with two types of transactions: Managed transactions and unmanaged trsanactions.

Reference: [Documentation](https://sequelize.org/docs/v6/other-topics/transactions/)

Managed transactions are the ones which are managed by the Sequelize ORM, i.e. if there is any error or exception in the transaction, it will automatically rollback. Else if there are no errors, the transaction will commit. The programmer doesn't need to explicitly commit or rollback.

In unmanaged transactions, the programmer has to explicitly define the logic for commit and rollback of transactions.

**How does MySQL prevent race condition?**

MySQL uses locking mechanisms to prevent race condition. Some of the locks which MySQL provides are:

1. Shared locks: A shared lock is a type of lock which allows multiple transactions to read at the same time, but doesn't allow any of them to write. (Multiple readers are allowed as R-R is not a conflict but writers are not permitted because it can lead to R-W or W-W conflict). It is called "shared" because the data is actually shared between multiple readers.

2. Exclusive locks: It is more strict than shared lock. It prevents multiple readers or writer to read the data at same time.
So, only one reader or writer can access the data at a time.

3. Intent locks: It is a type of lock which announces the intent of a transaction to read or write on some data. The benefit of using intent lock is that if a transaction has already announced its intent to read or write on a specific data, then another transaction can avoid acquiring a lock on that buffer.

4. Row level locks: This allows transactions to acquire a lock on only a specific row, as compared to the shared or exclusive locks (which acquire lock on the entire database). For our usecase, we will be using row level locks in the Booking Service, so that a lock can be ensured on a particular record in the 'Flights' table. This prevents problems like two different people booking the same seat.

The good part about MySQL is that it provides for MVCC (Multi Version Concurrency Control). What this means is that MySQL can actually allow multiple transactions to read or write on the same data without much conflict. 

Now the question might be: How does MySQL ensure that it can read and write on the same data without being into conflict? The answer to this question is that,  MySQL sort of captures the data it is about to modify at start of transaction and writes the changes in an entirely different version of data. This allows transaction to continue working on the original data without conflict. 

**Types of concurrency control systems**

A system like a Booking System necessitates the need for a concurrency control system, in order to prevent race condition. 

There are two types of concurrency control systems:

1. Pessimistic concurrency control systems: Pessimistic concurrency control systems assumes the worst i.e. it by default assumes that two users will want to update the same record at the same time, and thus it prevents that possibility by using a row level lock, no matter how unlikely the conflict is.

2. Optimistic concurrency control systems: In optimistic concurrency control systems, the system assumes that the possibility of conflicts is very less. So, it only checks for the possibility of conflicts before committing.

Example of optimistic concurrency control systems:

Suppose that there is only one seat left for a movie booking, and there are two users who are lookign forward to make the booking, and we are using an optimistic concurrency control system:

Transaction:

let any user enter

decrement number of available seats

if (available number of seats<0) rollback;

else commit;

Using the query `SELECT * FROM condition FOR UPDATE`, we can acquire a lock on the rows returned by the SELECT query. This lock will be held until the time the transaction (Which the SELECT query is a part of) commits. Any other transaction attempting to update these rows are placed into a time based queue to wait, and are executed chronologically after the first transaction is completed.

As an example consider, 

Transaction T1

---remaining statements

SELECT * FROM condition FOR UPDATE;

--remaining statements

COMMIT;

When the transaction T1 acquires a lock on the result set of the SELECT statement, no other transaction is allowed to update those rows (which is good for us, as it ensures concurrency). This lock will be held until the time transaction T1 commits. Any other transaction which attempts to update the rows while the lock is held, is added to a queue.

There can be different ways in which two services communicate, it can be through an HTTP API request, can be through RPC (Remote Procedure Call) or message queues. Message queues are particularly useful in case the scale of the producer application is much more than the consumer application.

One thing to keep in mind is that we shouldn't be using an `UPDATE` query to update the seats in case of a successful booking, as it can lead to concurrency issues. Instead, we should use the increment functionality provided by Sequelize, which helps us to achieve these operations without running into concurrency issues.

There's one more problem that we solved in the Booking service: which is if the user by mistake makes the API call to payment service twice, the money is going to be deducted twice. In order to solve this problem, we make the Payment API as an idempotent API. What this means is that, for every API request, we will generate a unique idempotency key. If that particular idempotency key has already been served by the payment API, the request will not be allowed. Else, we are going to serve that request, and if that request is successful, we will store this idempotency key in some storage mechanism like Database, caching mechanisms like Redis or an in-memory data structure.

Accessing from a database is a slow operation, so either we can go for a cache or in-memory data structure for faster access. In-memory data structure will lose the content once the server is restarted, but that shouldn't be a problem because we anyways need to flush the old idempotency keys.

In order to prevent misuse of this idempotency key by some bad actor, we can generate a UUID or have some regex check on the server side, so that the client can't play around with the idempotency key.

Idempotency as a concept in Maths or Computer Science is the property of certain operations where they can be applied multiple times without changing the result beyond the first successful operation.