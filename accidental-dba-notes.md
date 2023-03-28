# Accidental DBA Notes

---
## Common Backup Strategies
---

The three most common types:
1. Sever-level snapshot backups
2. Native full SQL Server backups
3. Transaction log backups

### Sever-level snapshot backups

The most common modern way to run backups.  It backups the server, files, and 
database by taking a snapshot.  Uses Volume Snapshot Service (VSS).
Example products: Veeam, SnapManager, NetBackup, BackupExec, and AppAssure.

Works really well for most backup needs by taking a snapshot of the sever's
contents at the time of excecution.

Potential Risks:

Taking a backup that is "dirty" or "crash-consistent".

If a snapshot of the server is taken in the middle of a SQL event, then
the restore of that backup would be the same as an unplanned server failure 
for the database.  SQL Server will attempt to recover from this restore point.
This might work "most" of the time but will result in data loss eventually.

Instead the snapshot process needs to be application-aware.  In that it will 
communicate to SQL Server that a snapshot is coming.  This will give SQL Server
the chance to pause and or complete writes to the database's before the 
snapshot is taken.

Potential Pain Points:

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

Potential Pain Points:

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

Potential Pain Points:

The restore process requires rolling forward through each of the backups in 
sequence. This takes more technical ability than either the VSS or full backup.

### Recovery Models

Full Recovery vs. Simple Recovery
*view by right clicking on the database*

| Full Recivery | Simple Recovery |
| ----------- | ----------- |
| Logs all transactions | Also Logs all transactions |
| ----------- | ----------- |
| Logs cleared when backuped up | Logs is cleared when transaction is complete |

Potential Risks:

Setting a database to *Full Recovery* but never making transactional log 
backups.  Here the server will never clear log files and will run into 
associated storage issues with an constantly growing set of logs.  Becomes 
even more crtical if the logs are kept on the same drive as the OS (C: drive 
for instance).