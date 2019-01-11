Sadequl Hussain on digitalocean ref. https://www.digitalocean.com/community/tutorials/how-to-back-up-restore-and-migrate-postgresql-databases-with-barman-on-centos-7

naming note
`pg`       as server ID and host name where PostgreSQL is installed
`backup`   as host name where Barman is located
`barman`   as the user running Barman on the backup server (identified by the parameter barman_user in the configuration)
`postgres` as the user running PostgreSQL on the pg serve

:pg_dump will backup only the :db at the time we run :pg_dump
the gap between the two backup times will be lost

the right solution to do backup is this
> back up the contents of the PostgreSQL *data directory* 
  and the *WAL files*

WAL files contain lists of CRUD transactions that happen to a db
the actual data contained in db files located in the *data directory*


when restoring, PostgreSQL `restores the contents of the data directory first`, 
and `then plays the transactions on top of it from the WAL files`

normally DBAs would write own backup scripts and scheduled cron jobs to implement backup & restore
*Barman does this in a standardized way*

server 01 aka. `pg`      - main postgres instance
server 02 aka. `standby` - standby postgres instance
server 03 aka. `backup`  - barman backup space

need :postgres, :barman, and :sudoer user to proceed

Barman will be installed to **/var/lib/barman/**
config at file **pg_hba.conf** eg. /etc/postgresql/$version/main/pg_hba.conf
this make postgres on `backup` to accept any connection coming from the Barman server
> host    all     all     barman-backup-server-ip/32        trust

required connections
* psql - barman @ backup  -->  postgres @ pg+standby
* ssh  - barman @ backup  <->  postgres @ pg+standby

test connections
```bash
postgres@pg      $ ssh barman@backup
postgres@standby $ ssh barman@backup
barman@backup    $ ssh postgres@pg
barman@backup    $ ssh postgres@standby
```

Barman's main config file at **/etc/barman.conf**
global config + per-postgres-instance config

`configuration_files_directory` default as **/etc/barman.d** - Barman will use the *.conf files here
split config in /etc/barman.conf into parts eg. /etc/barman.d/part1.conf, /etc/barman.d/part2.conf, etc. as much as you want
here we put all in /etc/barman.conf

# Barman global config
```bash
barman@backup $ sudo nano /etc/barman.conf
```
where we config as below
```conf
[barman]
compression  = gzip
reuse_backup = link         # when creating full backups of the PostgreSQL server, Barman will try to save space by using link/symlink
immediate_checkpoint = true # ensure when Barman starts a full backup, it will request PostgreSQL to perform a CHECKPOINT
                            ie. checkpoints ensures any modified data in postgres's cache are written to data files

basebackup_retry_times = 3  # retry 03 times if backup fails
basebackup_retry_sleep = 30 # each retry be delayed in 30 sec

last_backup_maximum_age = 1 DAYS             # base backup file's age     - the last full backup NOT older than 03 days
retention_policy = RECOVERY WINDOW OF 7 days # base backup + wal file age - be able to recover our database to any time of the last seven days
```

full sample global config
```conf
[barman]
barman_home = /var/lib/barman
...
barman_user = barman
log_file = /var/log/barman/barman.log

compression = gzip
reuse_backup = link
...
immediate_checkpoint = true
...
basebackup_retry_times = 3
basebackup_retry_sleep = 30

last_backup_maximum_age = 1 DAYS
```

# per-instance config
note we mark per-instance section by `[pg]` 
we can make as much block like this as possible for each postgres instance ie. `[standby]`

we config connection info of the instance, and some backup settings specific for that instance

```conf
[pg]
description = "main postgres instance"

ssh_command = ssh postgres@pg
conninfo    = host=pg user=postgres

retention_policy      = RECOVERY WINDOW OF 7 days
wal_retention_policy  = main
retention_policy_mode = auto
```
          
# last config - switch on backup for the postgres instance
locate backup folder
```bash
barman@pg $ barman show-server "$instance_name" | grep incoming_wals_directory
```
sample output
> incoming_wals_directory: /var/lib/barman/$instance_name/incoming

```bash
postgres@pg $ nano "$PGDATA/postgresql.conf" # eg. PGDATA=/var/lib/postgresql/data
```
we config postgres to turn on backup/archive mode as below
```conf
wal_level    = archive # choices eg. minimal, archive, hot_standby, or logical #TODO what these values mean
archive_mode = on

archive_command = 'rsync -a  %p                      barman@backup:/var/lib/barman/$instance_name/incoming/%f' # command to use to archive a logfile segment
                             ;from local wal files   ;copied to remote path      
```

restart postgres instance
eg.
```bash
sudo systemctl restart postgresql-9.4.service
```

# aftermath test for Barman setup
```bash
barman@backup $ barman list-server # get the server's name as $instance_name
barman@backup $ barman check "$instance_name"
```
you may have "backup maximum age FAILED state" when having blank backup files
some often-seen issue
* barman@backup not able to log in :pg instance
* postgres instance :pg not done-configured for WAL archiving
* ssh not working between servers
* etc.

TODO continue from https://www.digitalocean.com/community/tutorials/how-to-back-up-restore-and-migrate-postgresql-databases-with-barman-on-centos-7#step-8-—-creating-the-first-backup
sample for create backup & do restore
