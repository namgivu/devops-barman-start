ref. http://docs.pgbarman.org/release/2.5

# barman installation on ubuntu

# setup servers
require having the WAL archiving & backup methods to use [ref.](http://docs.pgbarman.org/release/2.5/#design-and-architecture)
here we choose [scenario 2 - Backup via rsync/SSH plus streaming](http://docs.pgbarman.org/release/2.5/#scenario-2-backup-via-rsyncssh)

naming note
`pg`       as server ID and host name where PostgreSQL is installed
`backup`   as host name where Barman is located
`barman`   as the user running Barman on the backup server (identified by the parameter barman_user in the configuration)
`postgres` as the user running PostgreSQL on the pg serve


ref. https://www.a2hosting.com/kb/developer-corner/postgresql/managing-postgresql-databases-and-users-from-the-command-line

# install barman
sudo apt install -y barman

# psql barman@backup --> postgres@pg
create user barman@pg
```bash
you@localhost $ ssh pg
you@pg $ sudo adduser barman # enter password when prompted
```

create barman as postgres superuser @pg
```bash
you@pg $ sudo su postgres
postgres@pg $ createuser -s -P barman # enter password when prompted
postgres@pg $ psql -c 'ALTER USER barman WITH superuser;'
```

in case you need to drop barman@pg:psql
```bash
postgres@pg $ psql -c 'drop owned by barman; drop user barman';
```

psql barman@backup --> postgres@pg
```bash
barman@backup $ psql -c 'SELECT version()' -U barman -h pg postgres
```

barman server config @ conninfo
```ini
[pg]
; ...
conninfo = host=pg user=barman dbname=postgres
```


# ssh postgres@pg --> barman@backup 
create ssh pub+pri key for postgres@pg
```bash
you@localhost $ ssh pg
you@pg $ sudo su postgres
postgres@pg $ mkdir -p $HOME/.ssh && ls $HOME/.ssh # ensure we don't have file :id_rsa here
postgres@pg $ ssh-keygen -t rsa

```

create ssh pub+pri key for barman@backup
```bash
you@localhost $ ssh backup
you@pg $ sudo su barman
barman@backup $ mkdir -p $HOME/.ssh && ls $HOME/.ssh # ensure we don't have file :id_rsa here
barman@backup $ ssh-keygen -t rsa
```

register pub key for the two
```bash
barman@backup $ printf '\nssh-rsa your-POSTGRES-SSH-PUB-key-here' >> $HOME/.ssh/authorized_keys
postgres@pg $ printf '\nssh-rsa your-BARMAN-SSH-PUB-key-here' >> $HOME/.ssh/authorized_keys
```

ssh postgres@pg --> barman@backup 
```bash
you@localhost $ ssh pg
you@pg $ sudo su postgres
postgres@pg $ ssh barman@backup -C true #TODO do we need -C here?
```

ssh barman@backup --> postgres@pg  
```bash
you@localhost $ ssh backup
you@backup $ sudo su barman
barman@backup $ ssh postgres@pg -C true #TODO do we need -C here?
```

# configuration
```bash
you@localhost $ ssh backup
you@localhost $ echo "
[pg]
description     = \"Our main PostgreSQL server\"
conninfo        = host=pg user=barman dbname=postgres
backup_method   = postgres
# backup_method = rsync
" > /etc/barman.d/pg.conf
```

