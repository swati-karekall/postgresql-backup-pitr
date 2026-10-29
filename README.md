# PostgreSQL Backup & Point-in-Time Recovery (PITR)

## Objective

Simulate an accidental `DROP TABLE` on a production-style database and recover all data to the moment just before the mistake, with zero data loss, using WAL archiving and a physical base backup.

## Environment

- PostgreSQL 17.3
- Linux 9.5 (VirtualBox VM)
- Tools: MobaXterm / PuTTY for SSH
- Data directory: `/u01/project`, WAL archive: `/u01/archives`, port 6666

## Scenario

The `shop` database has an `orders` table with 100,000 rows. After a base backup, 500 more orders arrive. Then someone drops the table by mistake. A nightly logical backup alone would lose the 500 new orders, so I used continuous WAL archiving and PITR to get everything back.

## Steps

1. Created the `shop` database and the `orders` table (100,000 rows).
2. Took a logical backup with `pg_dump` and restored it to test it.
3. Enabled WAL archiving (`archive_mode = on`, `archive_command` copying WAL files to `/u01/archives`) and verified it with `pg_stat_archiver`.
4. Took a physical base backup with `pg_basebackup` and verified it with `pg_verifybackup`.
5. Inserted 500 new orders (count = 100,500).
6. Created a named restore point: `SELECT pg_create_restore_point('before_drop');`
7. Forced a WAL switch so the restore point was archived.
8. Ran `DROP TABLE orders;` and switched the WAL again so the drop was archived.
9. Stopped the server and moved the broken data directory aside.
10. Restored the base backup as the new data directory.
11. Added recovery settings in `postgresql.auto.conf`: `restore_command`, `recovery_target_name = 'before_drop'`, `recovery_target_action = 'promote'`, and created `recovery.signal`.
12. Started the server and checked the log.
13. Verified the data, then removed the recovery settings and took a fresh base backup (the server moved to timeline 2 after recovery).

## Result

- The log shows `recovery stopping at restore point "before_drop"` and the server promoted to timeline 2.
- `SELECT count(*) FROM orders;` returned **100500**, so all orders were recovered, including the 500 added after the base backup.
- `pg_is_in_recovery()` returned `f`, so the server was open for normal use.

## Logical vs physical backup

| | Logical (`pg_dump`) | Physical (`pg_basebackup`) |
|---|---|---|
| What it copies | SQL objects and rows | Data files as they are on disk |
| Point-in-time recovery | No | Yes, with WAL archiving |
| Cross-version restore | Yes | No (same major version) |
| Restore a single table | Yes | No (whole cluster) |

## Issues faced and fixes

- **Non-default data directory and port:** my data directory was `/u01/project` and the port was 6666, so the default paths and service did not apply. I used `SHOW data_directory;` to find the path and `pg_ctl -D /u01/project` to stop and start the server.
- **Order of steps:** I tried to write the recovery settings before restoring the base backup, so the target folder did not exist yet. I restored the backup first, then added the settings and `recovery.signal`.
- **`cp: cannot stat ... .history` lines in the log:** these are harmless. PostgreSQL checks the archive for timeline history files and for the next WAL file, which do not exist yet.

## What I learned

- Why WAL archiving is needed for point-in-time recovery, and what `recovery.signal` and `restore_command` do
- Why a named restore point is safer than a timestamp
- Why the server moves to a new timeline after recovery
