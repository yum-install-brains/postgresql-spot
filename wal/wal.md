## WAL

WAL – it is a history log of all changes and actions in a database system so as to ensure that no data has been lost due to failures, such as a power failure, or some other server failure that causes the server crash. As the log contains sufficient information about each transaction executed already, the database server should be able to recover the database cluster by replaying changes and actions in the transaction log in case of the server crash. WAL files also used in Point-in-Time Recovery (PITR) and Streaming Replication (SR).

Without WAL all data in shared_buffers would be lost in case of server failure. PostgreSQL writes all modifications as history data into a persistent storage, to prepare for failures. In PostgreSQL, the history data are known as WAL data.

WAL records are written into the in-memory WAL buffer by change operations such as insertion, deletion, or commit action. They are immediately written into a WAL segment file on the storage when a transaction commits/aborts. LSN (Log Sequence Number) of WAL record represents the location where its record is written on the transaction log. LSN of record is used as the unique id of WAL record.

Query to find WAL file, which contains specified LSN:
```sql
SELECT pg_walfile_name('1/00002D3E');  # In version 10 or later, "SELECT pg_walfile_name('1/00002D3E');"
     pg_xlogfile_name     
--------------------------
 000000010000000100000000
(1 row)
```

### REDO
When we consider how database system recovers, there may be one question; what point does PostgreSQL start to recover from? The answer is REDO point; that is, the location to write the WAL record at the moment when the latest checkpoint is started. In fact, the database recovery processing is strongly linked to the checkpoint processing and both of these processing are inseparable.

### Full-Page Writes
Suppose that the TABLE_A's page-data on the storage is corrupted, because the operating system has failed while the background-writer process has been writing the dirty pages. As XLOG records cannot be replayed on the corrupted page, we would need an additional feature.

PostgreSQL supports a feature referred to as full-page writes to deal with such failures. If it is enabled, PostgreSQL writes a pair of the header-data and the entire page as XLOG record during the first change of each page after every checkpoint; default is enabled. In PostgreSQL, such a XLOG record containing the entire page is referred to as backup block (or full-page image).

### WAL segments
A WAL segment is a 16 MB file, by default, and it is internally divided into pages of 8192 bytes (8 KB). The WAL segment filename is in hexadecimal 24-digit number.

### WAL buffers flush triggers
Writing WAL from buffer to disk may be caused when any one of the following occurs:
- One running transaction has committed or has aborted.
- The WAL buffer has been filled up with many tuples have been written. (The WAL buffer size is set to the parameter wal_buffers.)
- A WAL writer process writes periodically.

If one of above occurs, all WAL records on the WAL buffer are written into a WAL segment file regardless of whether their transactions have been committed or not.

It is taken for granted that DML (Data Manipulation Language) operations write XLOG records, but so do non-DML operations. As described above, a commit action writes a XLOG record that contains the id of committed transaction. Another example may be a checkpoint action to write a XLOG record that contains general information of this checkpoint. Furthermore, SELECT statement creates XLOG records in special cases, though it does not usually create them. For example, if deletion of unnecessary tuples and defragmentation of the necessary tuples in pages occur by HOT(Heap Only Tuple) during a SELECT statement processing, the XLOG records of modified pages are written into the WAL buffer.

### WAL writer
WAL writer is a background process to check the WAL buffer periodically and write all unwritten XLOG records into the WAL segments. The purpose of this process is to avoid burst of writing of XLOG records. If this process has not been enabled, the writing of XLOG records might have been bottlenecked when a large amount of data committed at one time.

WAL writer is working by default and cannot be disabled. Check interval is set to the configuration parameter wal_writer_delay, default value is 200 milliseconds.

### Checkpoints  and database recovery
In PostgreSQL, the checkpointer (background) process performs checkpoints; its process starts when one of the following occurs:
- The interval time set for checkpoint_timeout from the previous checkpoint has been gone over (the default interval is 300 seconds (5 minutes)).
- In version 9.4 or earlier, the number of WAL segment files set for checkpoint_segments has been consumed since the previous checkpoint (the default number is 3).
- In version 9.5 or later, the total size of the WAL segment files in the pg_xlog (in version 10 or later, pg_wal) has exceeded the value of the parameter max_wal_size (the default value is 1GB (64 files)).
- PostgreSQL server stops in smart or fast mode.


Its process also does it when a superuser issues CHECKPOINT command manually.

Checkpoint not only flush shared buffers to disk, but also prepares database recovery (for more info read about pg_control file).

Checkpointing creates the checkpoint record which contains the REDO point, and stores the checkpoint location and more into the pg_control file. Therefore, PostgreSQL enables to recover itself by replaying WAL data from the REDO point (obtained from the checkpoint record) provided by the pg_control file. When PostgreSQL starts up, it reads the pg_control file at first.

### Database recovery
(1) PostgreSQL reads all items of the pg_control file when it starts. If the state item is in 'in production', PostgreSQL will go into recovery-mode because it means that the database was not stopped normally; if 'shut down', it will go into normal startup-mode.  
(2) PostgreSQL reads the latest checkpoint record, which location is written in the pg_control file, from the appropriate WAL segment file, and gets the REDO point from the record. If the latest checkpoint record is invalid, PostgreSQL reads the one prior to it. If both records are unreadable, it gives up recovering by itself. (Note that the prior checkpoint is not stored from PostgreSQL 11.)  
(3) Proper resource managers read and replay XLOG records in sequence from the REDO point until they come to the last point of the latest WAL segment. When a XLOG record is replayed and if it is a backup block, it will be overwritten on the corresponding table's page regardless of its LSN. Otherwise, a (non-backup block's) XLOG record will be replayed only if the LSN of this record is larger than the pd_lsn of a corresponding page.

### WAL files management
PostgreSQL writes XLOG records to one of the WAL segment files stored in the pg_xlog subdirectory (in version 10 or later, pg_wal subdirectory), and switches for a new one if the old one has been filled up. The number of the WAL files will vary depending on several configuration parameters, as well as server activity.

WAL segment switches occur when one of the following occurs:
- WAL segment has been filled up.
- The function pg_switch_xlog has been issued.
- archive_mode is enabled and the time set to archive_timeout has been exceeded.

Switched file is usually recycled (renamed and reused) for future use but it may be removed later if not necessary.

The number of WAL files adaptively changes depending on the server activity. If the amount of WAL data writing has constantly increased, the estimated number of the WAL segment files as well as the total size of WAL files also gradually increase. In the opposite case (i.e. the amount of WAL data writing has decreased), these decrease.

If the total size of the WAL files exceeds max_wal_size, a checkpoint will be started.

### WAL archiving
Continuous Archiving is a feature that copies WAL segment files to archival area at the time when WAL segment switches, and is performed by the archiver (background) process. The copied file is called an archive log. This feature is usually used for hot physical backup and PITR (Point-in-Time Recovery) described in Chapter 10.

The path of archival area is set to the configuration parameter archive_command. For example, using the following parameter, WAL segment files are copied to the directory '/home/postgres/archives/' every time when each segment switches:
```
archive_command = 'cp %p /home/postgres/archives/%f'
```
where, placeholder %p is copied WAL segment, and %f is archive log.

### Sources
- [interdb.jp blog](http://www.interdb.jp/pg/pgsql09.html)
- [DBA2](https://www.youtube.com/watch?v=GghMySWRH48&t=1s&index=8&list=PLaFqU3KCWw6JgufXBiW4dEB2-tDpmOXPH)
