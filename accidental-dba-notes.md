# Accidental DBA Notes


## Common Backup Strategies


The three most common types:
1. Sever-level snapshot backups
2. Native full SQL Server backups
3. Transaction log backups

### Sever-level snapshot backups

The most common modern way to run backups.  It backups the server, files, and 
database by taking a snapshot.  Uses Volume Snapshot Service (VSS).
Example products: Veeam, SnapManager, NetBackup, BackupExec, and AppAssure.

Works really well for most backup needs by taking a snapshot of the server's
contents at the time of execution.

**Potential Risks:**

Taking a backup that is "dirty" or "crash-consistent".

If a snapshot of the server is taken in the middle of a SQL event, then
the restore of that backup would be the same as an unplanned server failure 
for the database.  SQL Server will attempt to recover from this restore point.
This might work "most" of the time but will result in data loss eventually.

Instead the snapshot process needs to be application-aware.  In that it will 
communicate to SQL Server that a snapshot is coming.  This will give SQL Server
the chance to pause and or complete writes to the database's before the 
snapshot is taken.

**Potential Pain Points:**

The request to freeze the I/O for all databases before a backup can be painful
for the user base.  The larger the number of databases, the larger the pause
points become for the users.

Potential security concerns if you need to make the backup portable to hand
off to another team.  The backup could contain sensitive date that's on the
server.


### Native full SQL Server backups

This is the old fashion way to do backups.  Using the SQL Server backup
commands. It copy's all of the data pages to a destination file and includes 
some of the transaction logs.  It runs at the database level and not the 
server level.  

Much easier to make portable than VSS.  Only the needed data can be issolated
and backed up to give to another team.  Where as VSS backsup the entire server.

**Potential Pain Points:**

Backups are per database and not the server as a whole.  Backup plans need to 
be updated to include any newly added databases to the server.

More monitoring needed to know that the backup is running correctly.

Backups are larger and can be taxing on storage.  Need a process to move and 
or backup the backup files being created.

Restores are per database which can be labor intensive to roll back to live at
scale.

### Transaction log backups

SQL keeps a log of the transactions it's going to make before it makes them. 
A transaction log backup makes use of these logs to reduce the size of the data
that actually needs to be saved in a backup file.

An example of this used in the backup process is to take a single full backup 
overnight and save incremental backups of just the transaction log every hour 
after that.

**Potential Pain Points:**

The restore process requires rolling forward through each of the backups in 
sequence. This takes more technical ability than either the VSS or full backup.

### Recovery Models

Full Recovery vs. Simple Recovery
*view by right clicking on the database*

| Full Recovery | Simple Recovery |
| ----------- | ----------- |
| Logs all transactions | Also Logs all transactions |
| Logs cleared only when backup runs | Logs is cleared when transaction is complete |

**Potential Risks:**

Setting a database to *Full Recovery* but never making transactional log 
backups.  Here the server will never clear log files and will run into 
associated storage issues with an constantly growing set of logs.  Becomes 
even more critical if the logs are kept on the same drive as the OS (C: drive 
for instance).

## Restores

Workflows for a one time restores:
- The VSS process is simple.  Pick a snapshot and restore.
- SQL native backups have a few more steps.
    1. Pick a restore time to shoot for.
    2. Restore the closest full backup before that point.
    3. Restore the *last* differential backup before that point.
    4. Restore all of the logs up to that point.

Use the microsoft documentation when walking through a backup and restore.
*"You want to make sure you get it right... read the documentation."* 
*-Brent Ozar*

[MS Learn - SQL Server](https://learn.microsoft.com/en-us/sql/sql-server/?view=sql-server-ver16)

[MS Learn - Restore Database](https://learn.microsoft.com/en-us/sql/relational-databases/backup-restore/restore-a-differential-database-backup-sql-server?view=sql-server-ver16)

**Potential Risks:**

*Restoring over an existing database when only a subset of objects is needed.*
Restore to a new temp database instead and not over the original database.  Let 
the power users pull out the objects they are looking for from the new temp 
database.  This will avoid the restore fixing one users specific issue but 
potentially causing other data integrity problems.  

*When restoring an entire database*  Still restore to a new database name and
keep the original database intact.  Ask users to review and approve that the
restored database contains the data they need.  Rename the old database to
dbname_to_delete and save for a set amount of time before removing from the
server.

Look for any secondary processes that could have been broken by the restore.
This would be any kind of "high availability" or "disaster recovery" setup for 
the original database?  If it does, these need to be reviewed in the  
documentation to see if and how they need to be reenabled.

Examples of this include:
* Failover cluster
* Always On Availability Groups
* Database mirroring
* Replication
* Log shipping
* Backups of the log file tail


