---
title: Lecture 6 - Transactions and concurrency
enableToc: false
---
[[adsi-06-transactions.pdf|Lecture 6 - slides]]

### Index

- [[#Index|Index]]
- [[#"We want performance"|"We want performance"]]
	- [[#"We want performance"#Transaction Concept|Transaction Concept]]
	- [[#"We want performance"#Main issues|Main issues]]
	- [[#"We want performance"#ACID|ACID]]
	- [[#"We want performance"#Transaction State|Transaction State]]
	- [[#"We want performance"#Conflicting Instructions|Conflicting Instructions]]
		- [[#Conflicting Instructions#Conflict equivalent|Conflict equivalent]]
		- [[#Conflicting Instructions#Conflict serializable|Conflict serializable]]
	- [[#"We want performance"#Recoverable Schedules|Recoverable Schedules]]
	- [[#"We want performance"#Cascading rollback|Cascading rollback]]
	- [[#"We want performance"#Cascadeless schedules|Cascadeless schedules]]
- [[#Levels of Consistency in SQL|Levels of Consistency in SQL]]
	- [[#Levels of Consistency in SQL#Implementation of Isolation Levels|Implementation of Isolation Levels]]
	- [[#Levels of Consistency in SQL#Locking|Locking]]
			- [[#Conflict serializable#2-Phase Locking|2-Phase Locking]]



### "We want performance"

#### Transaction Concept
- Unit of a program.
- Group of ops that are executed as a whole
- Maintain a consistent state in the system
- Inconsistent while inside de transaction

#### Main issues
- Concurrent execution of multiple transactions
- Failures of various kinds, such as hardware failures and system crashes

Focus on **READ** and **WRITE**
![[transac.png]]
If fails in the middle -> **ROLLBACK**

#### ACID
- ***Atomicity***. Either all operations of the transaction are properly reflected in the database or none are.  
- ***Consistency***. Execution of a transaction in isolation preserves the   consistency of the database.  
- ***Isolation***. Although multiple transactions may execute concurrently, each transaction must be unaware of other concurrently executing transactions. Intermediate transaction results must be hidden from other concurrently executed transactions. 
	- That is, for every pair of transactions Ti and Tj, it appears to Ti that either Tj, finished execution before Ti started, or Tj started execution after Ti finished.  
- ***Durability***. After a transaction completes successfully, the changes it has made to the database persist, even if there are system failures.

#### Transaction State
![[transac_state.png]]
- ***Active*** – the initial state; the transaction stays in this state while it is executing  
- ***Partially committed*** – after the final statement has been executed. (Higher concurrency causes more of this state)
- ***Failed*** – after the discovery that normal execution can no longer proceed.  
- ***Aborted*** – after the transaction has been rolled back and the database restored to its state prior to the start of the transaction.  
	Two options after it has been aborted:  
	- Restart the transaction  
		- Can be done only if no internal logical error  
	- Kill the transaction  
- ***Committed*** – after successful completion.



**Schedule**: a sequences of instructions that specify the  
chronological order in which instructions of concurrent  
transactions are executed

**Serial Mode**: One transaction at the time (1 after the other)
Schedule 1 is T1 and T2 in Serial Mode
This one is Schedule 3 
![[serail_sched.png]]
We can switch the order of blocks if they operate in diff objs

**Basic Assumption** – Each transaction preserves database  
consistency.
We focus on a particular form of schedule equivalence called  
***conflict serializability***


#### Conflicting Instructions
1. Ti : read(Q)    Tj : read(Q)     **No conflict**  
2. Ti : read(Q)    Tj : write(Q)    **Conflict**  
3. Ti : write(Q)    Tj : read(Q)    **Conflict**  
4. Ti : write(Q)    Tj : write(Q)    **Conflict**

Forces temporal order: usually the older transaction executes first

##### Conflict equivalent
If a schedule S can be transformed into a schedule S' by a series  
of swaps of non-conflicting instructions, we say that S and S' are  
**conflict equivalent**.  

##### Conflict serializable
We say that a schedule S is **conflict serializable** if it is conflict  
equivalent to a serial schedule.
![[conf_serl.png]]
![[non_conf_srl.png]]
(Does not follow the "Precedence Graph")
We are unable to swap instructions in the above schedule to obtain either  
the serial schedule < T3 , T4 >, or the serial schedule < T4 , T3 >.
![[serl_test.png]]


#### Recoverable Schedules
- If transaction Tj reads a data item previously written by a transaction Ti , then the commit of Tj must appear after the commit of Ti  
- The following schedule is not recoverable:
![[unrec.png]]
- If T8 rolls back, T9 has read an inconsistent database state.  
- Database must ensure that schedules are recoverable.
Can only commit T9 after T8

#### Cascading rollback
- A single transaction failure leads to a series of transaction rollbacks.
![[casc_sch.png]]
(the schedule is recoverable)

#### Cascadeless schedules
- cascading rollbacks cannot occur
- Every cascadeless schedule is also recoverable 
	- Because if the read of Tj appears after the commit of Ti, then the commit of Tj will also appear after the commit of Ti
- It is desirable to restrict the schedules to those that are  **cascadeless**

### Levels of Consistency in SQL
- **Serializable** — ensures serializable execution.  
- **Repeatable read** — only committed records to be read.  
	- Repeated reads of same record must return same value.  
	- However, a transaction may not be serializable; it may find some records inserted by a transaction but not find others.  
- **Read committed** — only committed records can be read.  
	- Successive reads of a record may return different (committed) values.  
- **Read uncommitted** — even uncommitted records may be read.


![[const.png]]
**Analysis Queries** can benefit for "Read Uncommited" as it is the fastest (and full parallel)

In SQL Server the default is READ COMMITTED (preferes a performance approach)

Some systems have additional isolation levels  
- Snapshot isolation (not part of the SQL standard) each transaction works on its own snapshot of the data. 
- When commiting a problem might arise as each transaction spanshot might be different
- It allows no conflicts while inside the transaction, but problems in commit


#### Implementation of Isolation Levels
(Locking, Timestamps, Multiple versions of each data item)

#### Locking
- Lock on entire database vs. lock on items  
- How long to hold lock?  
- Shared vs. exclusive locks

1. **Exclusive (X) mode**. Data item can be both read as well as  
written. X-lock is requested using lock-X instruction.  
2. **Shared (S) mode**. Data item can only be read. S-lock is  
requested using lock-S instruction.
![[lock_comp.png]]

Bad Lock example:
![[bad_lock.png]]
You should not release a lock inside a transaction

###### 2-Phase Locking
![[2P_lock.png]]
A protocol which ensures conflict-serializable schedules  
- Phase 1: Growing Phase  
	- Transaction may obtain locks  
	- Transaction may not release locks  
- Phase 2: Shrinking Phase  
	- Transaction may release locks  
	- Transaction may not obtain locks 
- The protocol assures serializability  
	- It can be proved that the transactions can be serialized in the order of  their lock points

Does not PREVENT **DEADLOCKS**
![[DEADL.png]]
- The problem is that the transactions are locking in reverse order (B, A and A, B)
- The potential for deadlock exists in most locking protocols.  
- **Starvation** is also possible if concurrency control manager is badly designed. For example:  
	- A transaction may be waiting for an X-lock on an item, while a sequence of other transactions request and are granted an S-lock on the same item.  
	- The same transaction is repeatedly rolled back due to deadlocks.  
- Concurrency control manager can be designed to prevent starvation.


