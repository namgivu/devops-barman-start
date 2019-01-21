# aftermath protocol 
all check must pass following the order listed below


```bash
pg='pg'
echo_pass_fail="&& echo 'PASS' || echo 'FAILED'"

# ssh connections check - all below must succeed ie. see $HOME folder listing 
barman@backup $ ssh postgres@pg   'ls $HOME' ${echo_pass_fail}
postgres@pg   $ ssh barman@backup 'ls $HOME' ${echo_pass_fail}


# psql connection check
barman@backup $ psql -t -c 'SELECT version();' --pset pager=off -U barman -h "$pg" postgres ${echo_pass_fail}


# barman health check
barman@backup $ barman list-server # get the server's name as $pg
barman@backup $ barman check "$pg"


# barman incoming_wals_directory check
barman@backup $ barman show-server "$pg" | grep incoming_wals_directory
# > incoming_wals_directory: /var/lib/barman/pg/incoming
postgres@pg $ cat /etc/postgresql/10/main/postgresql.conf | grep archive_command
# > archive_command = 'rsync -a  %p  barman@staging:/var/lib/barman/pg/incoming/%f' 
echo 'the two paths above MUST MATCH'

# barman WAL-archive test #TODO do we need this
barman@backup $ barman switch-xlog --force --archive "$pg"
barman@backup $ barman receive-wal "$pg"


#TODO barman backup test
#TODO barman restore test

```


# often-seen issues

## issues listing
01 ssh not working between servers
02 barman@backup not able to psql-log-in :pg server
03 postgres instance :pg incomplete configure for WAL archiving

## troubleshooter for 03 - error "WAL archive: FAILED (please make sure WAL shipping is setup)"
you may have "backup maximum age FAILED state" when having blank backup files
try this command to force a backup ref. https://stackoverflow.com/a/42616594/248616
```bash
barman@backup $ barman switch-xlog --force --archive "$pg"
```
as u may still getting the error, please check
eck Barman's :incoming_wals_directory vs postgresql.conf's :archive_command
in postgresql.conf we have :archive_command
> archive_command = 'rsync -a %p barman@backup:INCOMING_WALS_DIRECTORY/%f' 

we must have :INCOMING_WALS_DIRECTORY **same value** with
             :incoming_wals_directory returned by command `barman show-server pg`

bash util to check  
```bash
barman@backup $ barman show-server pg | grep incoming_wals_directory
# output1
# > incoming_wals_directory: /var/lib/barman/pg/incoming

postgres@pg $ cat /etc/postgresql/10/main/postgresql.conf | grep archive_command
# output2
# > archive_command = 'rsync -a  %p  barman@staging:/var/lib/barman/pg/incoming/%f' 
```

we **must have same path** in :output1 and :output2
if they not matched, please make them matched and don't forget to **restart postgres** afterward.


# some other useful troubleshooters
if barman need password to authenticate with postgres
```bash
sudoer@pg $ echo "$pg:5432:*:barman:password" >> ~/.pgpass
```

force wal-streaming fetch
```bash
barman receive-wal "$pg"
```
