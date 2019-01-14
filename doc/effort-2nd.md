Sadequl Hussain on digitalocean ref. https://www.digitalocean.com/community/tutorials/how-to-back-up-restore-and-migrate-postgresql-databases-with-barman-on-centos-7

naming note
`pg`       as server ID and host name where PostgreSQL is installed
`backup`   as host name where Barman is located
`barman`   as the user running Barman on the backup server (identified by the parameter barman_user in the configuration)
`postgres` as the user running PostgreSQL on the pg serve
$xxx       is value placeholder requiring us to fill in concrete values

:pg_dump will backup only the :db at the time we run :pg_dump
the gap between the two backup times will be lost

the right solution to do backup is this
> back up the contents of the PostgreSQL *data directory* 
  and the *WAL files*

WAL files contain lists of CRUD transactions that happen to a db
the actual data contained in db files located in the **data directory** ie. $PGDATA
eg. PGDATA=/var/lib/postgresql/data

when restoring, PostgreSQL `restores the contents of the data directory first`, 
and `then plays the transactions on top of it from the WAL files`

normally DBAs would write own backup scripts and scheduled cron jobs to implement backup & restore
**Barman helps to do this in a standardized way**

install Barman
```bash
sudoer@backup $ sudo apt install -y barman
```
Barman's home will be at **/var/lib/barman/**

postgres config file at **pg_hba.conf** eg. /etc/postgresql/$version/main/pg_hba.conf
make postgres on `pg` accept any connection coming from Barman server `backup`
> host    all             all             $backup_ip/32    trust

bash util
```bash
echo "

# my barman config
host    all             all             $backup_ip/32    trust
" | tee -a /etc/postgresql/$version/main/pg_hba.conf

```

related servers in the flow
server 01 aka. `pg`      - main postgres instance
server 02 aka. `standby` - standby postgres instance used for failed-over for `pg`
server 03 aka. `backup`  - barman backup space

related user in the flow
:postgres@pg, :barman@backup, :sudoer on all servers to switch among the roles

required connections
* psql - barman@backup  -->  postgres@pg+standby 
* ssh  - barman@backup  <->  postgres@pg+standby # setup detail in doc/effort-0th.md

test psql connection
```bash
barman@backup $ psql -c 'SELECT version()' -U barman -h $pg postgres
```

test ssh connections - all below must go thru 
barman@backup <-> postgres@pg
```bash
barman@backup $ ssh postgres@pg
postgres@pg   $ ssh barman@backup
```
same for barman@backup <-> postgres@standby

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
TODO reuse_backup not found
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

bash util
```bash
sudoer@backup $ echo "

;region my barman config
compression  = gzip
reuse_backup = link         
immediate_checkpoint = true 

basebackup_retry_times = 3  
basebackup_retry_sleep = 30 

last_backup_maximum_age = 1 DAYS             
retention_policy = RECOVERY WINDOW OF 7 days


[pg]
description = main postgres instance

ssh_command = ssh postgres@pg
conninfo    = host=pg user=postgres

retention_policy      = RECOVERY WINDOW OF 7 days
wal_retention_policy  = main
retention_policy_mode = auto
 
archiver = on
;endregion my barman config
" | sudo tee -a /etc/barman.conf
```
          
# last config - switch on backup for the postgres instance
locate backup folder
```bash
barman@pg $ barman show-server "$instance_name" | grep incoming_wals_directory
```
sample output
> incoming_wals_directory: /var/lib/barman/$instance_name/incoming

open :postgresql.conf to edit
```bash
postgres@pg $ psql -U postgres -c 'show config_file' # we got postgresql.conf location eg. /etc/postgresql/10/main/postgresql.conf
postgres@pg $ nano /etc/postgresql/10/main/postgresql.conf 
```
in this file, we config postgres to turn on backup/archive mode as below
```conf
wal_level = archive # choices eg. minimal, archive, hot_standby, or logical #TODO what these values mean

archive_mode = on
archive_command = 'rsync -a  %p                      barman@backup:/var/lib/barman/$instance_name/incoming/%f' # command to use to archive a logfile segment
                             ;from local wal files   ;copied to remote path      
```

bash util
```bash
postgres@pg $ echo "

#region my barman config
wal_level = archive 
archive_mode = on
archive_command = 'rsync -a  %p  barman@backup:/var/lib/barman/$instance_name/incoming/%f' 
#endregion
" | tee -a /etc/postgresql/10/main/postgresql.conf
```

restart postgres instance
```bash
sudoer@pg $ sudo /etc/init.d/postgresql restart
```

# aftermath test for Barman setup
```bash
barman@backup $ barman list-server # get the server's name as $instance_name
barman@backup $ barman check "$instance_name"
```
you may have "backup maximum age FAILED state" when having blank backup files
some often-seen issue
* ssh not working between servers
* barman@backup not able to psql-log-in :pg server
* postgres instance :pg incomplete configure for WAL archiving
* etc.


## check Barman's :incoming_wals_directory vs postgresql.conf's :archive_command
in postgresql.conf we have :archive_command
> archive_command = 'rsync -a %p barman@backup:INCOMING_WALS_DIRECTORY/%f' 

we must have :INCOMING_WALS_DIRECTORY same value with
             :incoming_wals_directory returned in `barman show-server pg`

bash util to check  
```bash
barman@backup $ barman show-server pg | grep incoming_wals_directory
# output1
# > incoming_wals_directory: /var/lib/barman/pg/incoming

postgres@pg $ cat /etc/postgresql/10/main/postgresql.conf | grep archive_command
# output2
# > archive_command = 'rsync -a  %p  barman@staging:/var/lib/barman/staging/incoming/%f' 
```

we must have same path in :output1 and :output2


# sample create backup 
```bash
barman@backup $ barman backup "$instance_name"
```

backup file will be created at **/var/lib/barman**
```bash
barman@backup $ ls /var/lib/barman
# /var/lib/barman   /base       # base backup file stored here
# /var/lib/barman   /incoming   # completed wal files sent here
# /var/lib/barman   /wals       # files here copied from :incoming 
```

> During a restoration, Barman will recover contents in **base** into the target server's **data directory** 
  Then files in **wals** folder are used to apply transaction changes and bring the target server to a consistent state.

view backup info
```bash
barman@backup $ barman list-backup "$instance_name"              # list backup ids
barman@backup $ barman show-backup "$instance_name" "$backup_id" # view backup detail by id
barman@backup $ barman list-files  "$instance_name" "$backup_id" # list base+wal files of :backup_id
```
sample output
>    :backup_id        :created_at                :base_size       :wal_size
> pg 20151111T051954 - Wed Nov 11 05:19:46 2015 - Size: 26.9 MiB - WAL Size: 0 B

# run backup as scheduled
backups should happen automatically on a schedule
open crontab file
```bash
barman@backup $ crontab -e
```
add below file
```crontab
30 23 * * * /usr/bin/barman backup $instance_name_here  # run backup every night at 1130PM
*  *  * * * /usr/bin/barman cron                        # very minute, perform maintenance on base and wal files
```

# sample restore
stop postgres
```bash
postgres@pg $ sudo /etc/init.d/postgresql stop
```

do restore
```bash
# list latest barman's backup
barman@backup $ barman show-backup "$instance_name" latest # we have :backup_id here

# restore from 
barman@backup $ barman recover --target-time "Begin time" \                          # use the begin time from the show-backup command
                               --remote-ssh-command "ssh postgres@$instance_name" \   
                               "$instance_name" "$backup-id" \   
                               /var/lib/pgsql/9.4/data  # path where you want the backup to be restored; retrieved by `ssh $instance_name 'echo $PGDATA' `
#TODO find a less-verbose command to replace the above
```

start postgres back
```bash
postgres@pg $ sudo /etc/init.d/postgresql start
```
