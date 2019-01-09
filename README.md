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

connections required
`backup` server can connect to `pg` server as **superuser** `postgres`
