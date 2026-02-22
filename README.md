# Setup a Cluster of mysql instances with Podman 

## Create a Podman network
```
podman network create mysql-net
```

## Create directories for each mysql pod
```
mkdir -p /tmp/mysqlcluster/{mysql1,mysql2,mysql3}
```

## Generate uuid in linux terminal
```
uuidgen
1d613c70-31cc-4520-932c-29b670a3823e
```

## Create a config file named /tmp/mysqlcluster/mysql1/my.cnf
```
[mysqld]
server-id=1
log-bin=mysql-bin
binlog_format=ROW

gtid_mode=ON
enforce_gtid_consistency=ON

master_info_repository=TABLE
relay_log_info_repository=TABLE
transaction_write_set_extraction=XXHASH64

plugin_load_add='group_replication.so'
group_replication_group_name="1d613c70-31cc-4520-932c-29b670a3823e"
group_replication_start_on_boot=off
group_replication_local_address="mysql1:33061"
group_replication_group_seeds="mysql1:33061,mysql2:33061,mysql3:33061"
group_replication_bootstrap_group=off
```

## Create a mysql2 config file named /tmp/mysqlcluster/mysql2/my.cnf
```
[mysqld]
server-id=2
log-bin=mysql-bin
binlog_format=ROW

gtid_mode=ON
enforce_gtid_consistency=ON

master_info_repository=TABLE
relay_log_info_repository=TABLE
transaction_write_set_extraction=XXHASH64

plugin_load_add='group_replication.so'
group_replication_group_name="1d613c70-31cc-4520-932c-29b670a3823e"
group_replication_start_on_boot=off
group_replication_local_address="mysql2:33061"
group_replication_group_seeds="mysql1:33061,mysql2:33061,mysql3:33061"
group_replication_bootstrap_group=off
```

## Create a mysql3 config file named /tmp/mysqlcluster/mysql3/my.cnf
```
[mysqld]
server-id=3
log-bin=mysql-bin
binlog_format=ROW

gtid_mode=ON
enforce_gtid_consistency=ON

master_info_repository=TABLE
relay_log_info_repository=TABLE
transaction_write_set_extraction=XXHASH64

plugin_load_add='group_replication.so'
group_replication_group_name="1d613c70-31cc-4520-932c-29b670a3823e"
group_replication_start_on_boot=off
group_replication_local_address="mysql3:33061"
group_replication_group_seeds="mysql1:33061,mysql2:33061,mysql3:33061"
group_replication_bootstrap_group=off
```

## Create the primary mysql master
```
podman run -d \
  --name mysql1 \
  --network mysql-net \
  -e MYSQL_ROOT_PASSWORD=root@123 \
  -v ~/mysqlcluster/mysql1:/var/lib/mysql \
  -v ~/mysqlcluster/mysql1/my.cnf:/etc/mysql/conf.d/my.cnf:Z \
  mysql:8.0
```

## Create the mysql replica 1 
```
podman run -d \
  --name mysql2 \
  --network mysql-net \
  -e MYSQL_ROOT_PASSWORD=root@123 \
  -v ~/mysqlcluster/mysql2:/var/lib/mysql \
  -v ~/mysqlcluster/mysql2/my.cnf:/etc/mysql/conf.d/my.cnf:Z \
  mysql:8.0
```

## Create the mysql replica 2
```
podman run -d \
  --name mysql3 \
  --network mysql-net \
  -e MYSQL_ROOT_PASSWORD=root@123 \
  -v ~/mysqlcluster/mysql3:/var/lib/mysql \
  -v ~/mysqlcluster/mysql3/my.cnf:/etc/mysql/conf.d/my.cnf:Z \
  mysql:8.0
```

## Connect to primary mysql master ie. mysql1
```
podman exec -it mysql1 mysql -uroot -p

# In the mysql prompt
CREATE USER 'repl'@'%' IDENTIFIED BY 'repl@123';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
FLUSH PRIVILEGES;
```

## Configure group replication
```
CHANGE MASTER TO 
  MASTER_USER='repl',
  MASTER_PASSWORD='repl@123'
  FOR CHANNEL 'group_replication_recovery';

SET GLOBAL group_replication_bootstrap_group=ON;
START GROUP_REPLICATION;
SET GLOBAL group_replication_bootstrap_group=OFF;
```

## Start replicas on mysql2 prompt
```
CHANGE MASTER TO 
  MASTER_USER='repl',
  MASTER_PASSWORD='repl@123'
  FOR CHANNEL 'group_replication_recovery';

START GROUP_REPLICATION;
```

## Start replicas on mysql3 prompt
```
CHANGE MASTER TO 
  MASTER_USER='repl',
  MASTER_PASSWORD='repl@123'
  FOR CHANNEL 'group_replication_recovery';

START GROUP_REPLICATION;
```

## Check the cluster status
```
SELECT * FROM performance_schema.replication_group_members;
```
