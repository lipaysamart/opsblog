## Concept
## Dependents
- 服务器: 2c4g
- 系统: ubuntu22

## Todo
- 环境搭建
- 配置用户凭证
- 启动组复制
- 引导组复制
- 添加组成员

#### 环境搭建

- 创建用户和组

```sh
groupadd mysql
useradd -r -g mysql -s /bin/false mysql
```

- 创建目录

```sh
export MYSQL_HOME=/opt/mgr-test
mkdir -p $MYSQL_HOME/{data,mysql-8.0}
mkdir $MYSQL_HOME/data/s{1..3}
```

- 下载二进制包

```sh
export MYSQL_BINARY=https://dev.mysql.com/get/Downloads/MySQL-8.0/mysql-8.0.40-linux-glibc2.28-x86_64.tar.xz
wget $MYSQL_BINARY 
tar -xf mysql-8.0.40-linux-glibc2.28-x86_64.tar.xz -C $MYSQL_HOME/mysql-8.0
```

- 设置软链

```sh
ln -s $MYSQL_BINARY/mysql-8.0/bin/{mysql,mysqld} /usr/bin
```

- 初始化数据目录

```sh
mysqld --initialize-insecure --basedir=$MYSQL_HOME/mysql-8.0 --datadir=$MYSQL_HOME/data/s1
mysqld --initialize-insecure --basedir=$MYSQL_HOME/mysql-8.0 --datadir=$MYSQL_HOME/data/s2
mysqld --initialize-insecure --basedir=$MYSQL_HOME/mysql-8.0 --datadir=$MYSQL_HOME/data/s3
```

- 文件授权

```sh
chown -R mysql:mysql $MYSQL_HOME/data
```

- 复制组成员配置

`vim $MYSQL_HOME/data/s1/my.cnf`

```ini
[mysqld]

# server configuration
datadir=/opt/mgr-test/data/s1
basedir=/opt/mgr-test/mysql-8.0/

# 这些设置将服务器配置为使用唯一标识符号 1，以启用 “使用全局事务标识符进行复制”，并仅允许执行可以使用 GTID 安全记录的语句。
server_id=1
gtid_mode=ON
enforce_gtid_consistency=ON
port=24801
socket=/opt/mgr-test/data/s1/s1.sock
# 对于组复制，数据必须存储在InnoDB事务存储引擎中,使用其他存储引擎（包括临时 MEMORY 存储引擎）可能会导致组复制出错
disabled_storage_engines="MyISAM,BLACKHOLE,FEDERATED,ARCHIVE,MEMORY"
# 将 Group Replication 插件添加到服务器启动时加载的插件列表中
plugin_load_add='group_replication.so'
# 将告诉插件它正在加入或创建的组名为 “aaaaaaaa-aaaa-aaaa-aaaa-aaaa” (必须是有效的 UUID。您可以使用 SELECT UUID（） 生成一个)。
group_replication_group_name="aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa"
# 变量配置为 off 会指示插件在服务器启动时不自动启动操作
group_replication_start_on_boot=off
# 引导时自动启动组复制
group_replication_bootstrap_group=off
# 组通信时的<地址:端口>
group_replication_local_address="127.0.0.1:24901"
# 组成员的<地址:端口>
group_replication_group_seeds="127.0.0.1:24901,127.0.0.1:24902,127.0.0.1:24903"
group_replication_recovery_get_public_key=ON
```

> 复制到其它数据目录下时需要注意替换 `s1` 为其它数据目录名称, `port` 与 `server_id` 保持唯一性.

- 配置 systemd

`vim /usr/lib/systemd/system/mysqld@replica01.service`

```ini
[Unit]
Description=MySQL Server
Documentation=man:mysqld(8)
Documentation=http://dev.mysql.com/doc/refman/en/using-systemd.html
After=network-online.target
Wants=network-online.target
After=syslog.target

[Install]
WantedBy=multi-user.target

[Service]
User=mysql
Group=mysql

Type=notify

# Disable service start and stop timeout logic of systemd for mysqld service.
TimeoutSec=0

# Execute pre and post scripts as root
PermissionsStartOnly=true

# Needed to create system tables
#ExecStartPre=/usr/bin/mysqld_pre_systemd %I

# Start main service
ExecStart=/usr/bin/mysqld --defaults-group-suffix=@%I --defaults-file=/opt/mgr-test/data/s1/my.cnf --mysqlx=off $MYSQLD_OPTS

# Use this to switch malloc implementation
EnvironmentFile=-/etc/sysconfig/mysql

# Sets open_files_limit
LimitNOFILE = 10000

Restart=on-failure

RestartPreventExitStatus=1

# Set enviroment variable MYSQLD_PARENT_PID. This is required for restart.
Environment=MYSQLD_PARENT_PID=1

PrivateTmp=false
```

> 再分别创建 `02` `03` 的服务, 注意替换 `s1` 为其它数据目录名称

#### 配置用户凭证

- 启动第一台 mysql-server

```sh
systemctl start mysqld@replica01.service
```

- 连接 mysql-server

```sh
mysql -uroot -S ./s1.sock
```

- 创建用户

> 禁用二进制日志记录以便在每个实例上单独创建复制用户

```sql
SET SQL_LOG_BIN=0;
CREATE USER rpl_user@'%' IDENTIFIED BY 'rZaToqYbiPJB473h';
GRANT REPLICATION SLAVE ON *.* TO rpl_user@'%';
GRANT CONNECTION_ADMIN ON *.* TO rpl_user@'%';
GRANT BACKUP_ADMIN ON *.* TO rpl_user@'%';
GRANT GROUP_REPLICATION_STREAM ON *.* TO rpl_user@'%';
FLUSH PRIVILEGES;
SET SQL_LOG_BIN=1;
```

- 使用凭证

```sql
/* in MySQL 8.0.32 before */
CHANGE MASTER TO MASTER_USER='rpl_user', MASTER_PASSWORD='rZaToqYbiPJB473h' FOR CHANNEL 'group_replication_recovery';
/* in MySQL 8.0.23 later */
CHANGE REPLICATION SOURCE TO SOURCE_USER='rpl_user', SOURCE_PASSWORD='rZaToqYbiPJB473h' FOR CHANNEL 'group_replication_recovery';
```

#### 启动组复制

- 查看组插件是否启用

```sql
show plugins;
+----------------------------------+----------+--------------------+----------------------+---------+
| Name                             | Status   | Type               | Library              | License |
+----------------------------------+----------+--------------------+----------------------+---------+
| binlog                           | ACTIVE   | STORAGE ENGINE     | NULL                 | GPL     |
(...) 
| group_replication                | ACTIVE   | GROUP REPLICATION  | group_replication.so | GPL     |
+----------------------------------+----------+--------------------+----------------------+---------+
```

#### 引导组复制

> 引导只能由单个服务器完成，该服务器启动组，并且只能执行一次。

```sql
SET GLOBAL group_replication_bootstrap_group=ON;
START GROUP_REPLICATION USER='rpl_user', PASSWORD='rZaToqYbiPJB473h';
SET GLOBAL group_replication_bootstrap_group=OFF;
```

- 查看组是否启动

```sql
SELECT * FROM performance_schema.replication_group_members;
+---------------------------+--------------------------------------+-------------------------+-------------+--------------+-------------+----------------+----------------------------+
| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST             | MEMBER_PORT | MEMBER_STATE | MEMBER_ROLE | MEMBER_VERSION | MEMBER_COMMUNICATION_STACK |
+---------------------------+--------------------------------------+-------------------------+-------------+--------------+-------------+----------------+----------------------------+
| group_replication_applier | 75732740-8f4b-11ef-8309-00163e0c6e68 | iZj6c36h1t72eafffdsvzyZ |       24801 | ONLINE       | PRIMARY     | 8.0.40         | XCom                       |
+---------------------------+--------------------------------------+-------------------------+-------------+--------------+-------------+----------------+----------------------------+
```

- 证明组成员能处理负载

```sql
CREATE DATABASE test;
USE test;
CREATE TABLE t1 (c1 INT PRIMARY KEY, c2 TEXT NOT NULL);
INSERT INTO t1 VALUES (1, 'Luis');
```

- 查看表 t1 的内容和二进制日志

```sql
mysql> SELECT * FROM t1;
+----+------+
| c1 | c2   |
+----+------+
|  1 | Luis |
+----+------+

mysql> SHOW BINLOG EVENTS;
+---------------+-----+----------------+-----------+-------------+--------------------------------------------------------------------+
| Log_name      | Pos | Event_type     | Server_id | End_log_pos | Info                                                               |
+---------------+-----+----------------+-----------+-------------+--------------------------------------------------------------------+
| binlog.000001 |   4 | Format_desc    |         1 |         123 | Server ver: 8.0.40-log, Binlog ver: 4                              |
| binlog.000001 | 123 | Previous_gtids |         1 |         150 |                                                                    |
| binlog.000001 | 150 | Gtid           |         1 |         211 | SET @@SESSION.GTID_NEXT= 'aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa:1'  |
| binlog.000001 | 211 | Query          |         1 |         270 | BEGIN                                                              |
| binlog.000001 | 270 | View_change    |         1 |         369 | view_id=14724817264259180:1                                        |
| binlog.000001 | 369 | Query          |         1 |         434 | COMMIT                                                             |
| binlog.000001 | 434 | Gtid           |         1 |         495 | SET @@SESSION.GTID_NEXT= 'aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa:2'  |
| binlog.000001 | 495 | Query          |         1 |         585 | CREATE DATABASE test                                               |
| binlog.000001 | 585 | Gtid           |         1 |         646 | SET @@SESSION.GTID_NEXT= 'aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa:3'  |
| binlog.000001 | 646 | Query          |         1 |         770 | use `test`; CREATE TABLE t1 (c1 INT PRIMARY KEY, c2 TEXT NOT NULL) |
| binlog.000001 | 770 | Gtid           |         1 |         831 | SET @@SESSION.GTID_NEXT= 'aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa:4'  |
| binlog.000001 | 831 | Query          |         1 |         899 | BEGIN                                                              |
| binlog.000001 | 899 | Table_map      |         1 |         942 | table_id: 108 (test.t1)                                            |
| binlog.000001 | 942 | Write_rows     |         1 |         984 | table_id: 108 flags: STMT_END_F                                    |
| binlog.000001 | 984 | Xid            |         1 |        1011 | COMMIT /* xid=38 */                                                |
+---------------+-----+----------------+-----------+-------------+--------------------------------------------------------------------+
```

#### 添加组成员

- 启动第二个 mysql-server

```sh
systemctl start mysqld@replica02.service
```

- 创建用户

```sql
SET SQL_LOG_BIN=0;
CREATE USER rpl_user@'%' IDENTIFIED BY 'rZaToqYbiPJB473h';
GRANT REPLICATION SLAVE ON *.* TO rpl_user@'%';
GRANT CONNECTION_ADMIN ON *.* TO rpl_user@'%';
GRANT BACKUP_ADMIN ON *.* TO rpl_user@'%';
GRANT GROUP_REPLICATION_STREAM ON *.* TO rpl_user@'%';
FLUSH PRIVILEGES;
SET SQL_LOG_BIN=1;
```

- 使用凭证

```sql
/* in MySQL 8.0.32 before */
CHANGE MASTER TO MASTER_USER='rpl_user', MASTER_PASSWORD='rZaToqYbiPJB473h' FOR CHANNEL 'group_replication_recovery';
/* in MySQL 8.0.23 later */
CHANGE REPLICATION SOURCE TO SOURCE_USER='rpl_user', SOURCE_PASSWORD='rZaToqYbiPJB473h' FOR CHANNEL 'group_replication_recovery';
```

- 启动组复制

```sql
START GROUP_REPLICATION USER='rpl_user', PASSWORD='rZaToqYbiPJB473h';
```

- 查询组成员

```sql
SELECT * FROM performance_schema.replication_group_members;
+---------------------------+--------------------------------------+-------------------------+-------------+--------------+-------------+----------------+----------------------------+
| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST             | MEMBER_PORT | MEMBER_STATE | MEMBER_ROLE | MEMBER_VERSION | MEMBER_COMMUNICATION_STACK |
+---------------------------+--------------------------------------+-------------------------+-------------+--------------+-------------+----------------+----------------------------+
| group_replication_applier | 75732740-8f4b-11ef-8309-00163e0c6e68 | iZj6c36h1t72eafffdsvzyZ |       24801 | ONLINE       | PRIMARY     | 8.0.40         | XCom                       |
| group_replication_applier | 7b41df78-8f4b-11ef-85cf-00163e0c6e68 | iZj6c36h1t72eafffdsvzyZ |       24802 | ONLINE       | SECONDARY   | 8.0.40         | XCom                       |
+---------------------------+--------------------------------------+-------------------------+-------------+--------------+-------------+----------------+----------------------------+
```

- 验证 s2 是否已与 s1 同步

```sql
mysql> SHOW DATABASES LIKE 'test';
+-----------------+
| Database (test) |
+-----------------+
| test            |
+-----------------+

mysql> SELECT * FROM test.t1;
+----+------+
| c1 | c2   |
+----+------+
|  1 | Luis |
+----+------+

mysql> SHOW BINLOG EVENTS;
+---------------+------+----------------+-----------+-------------+--------------------------------------------------------------------+
| Log_name      | Pos  | Event_type     | Server_id | End_log_pos | Info                                                               |
+---------------+------+----------------+-----------+-------------+--------------------------------------------------------------------+
| binlog.000001 |    4 | Format_desc    |         2 |         123 | Server ver: 8.0.40-log, Binlog ver: 4                              |
| binlog.000001 |  123 | Previous_gtids |         2 |         150 |                                                                    |
| binlog.000001 |  150 | Gtid           |         1 |         211 | SET @@SESSION.GTID_NEXT= 'aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa:1'  |
| binlog.000001 |  211 | Query          |         1 |         270 | BEGIN                                                              |
| binlog.000001 |  270 | View_change    |         1 |         369 | view_id=14724832985483517:1                                        |
| binlog.000001 |  369 | Query          |         1 |         434 | COMMIT                                                             |
| binlog.000001 |  434 | Gtid           |         1 |         495 | SET @@SESSION.GTID_NEXT= 'aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa:2'  |
| binlog.000001 |  495 | Query          |         1 |         585 | CREATE DATABASE test                                               |
| binlog.000001 |  585 | Gtid           |         1 |         646 | SET @@SESSION.GTID_NEXT= 'aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa:3'  |
| binlog.000001 |  646 | Query          |         1 |         770 | use `test`; CREATE TABLE t1 (c1 INT PRIMARY KEY, c2 TEXT NOT NULL) |
| binlog.000001 |  770 | Gtid           |         1 |         831 | SET @@SESSION.GTID_NEXT= 'aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa:4'  |
| binlog.000001 |  831 | Query          |         1 |         890 | BEGIN                                                              |
| binlog.000001 |  890 | Table_map      |         1 |         933 | table_id: 108 (test.t1)                                            |
| binlog.000001 |  933 | Write_rows     |         1 |         975 | table_id: 108 flags: STMT_END_F                                    |
| binlog.000001 |  975 | Xid            |         1 |        1002 | COMMIT /* xid=30 */                                                |
| binlog.000001 | 1002 | Gtid           |         1 |        1063 | SET @@SESSION.GTID_NEXT= 'aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa:5'  |
| binlog.000001 | 1063 | Query          |         1 |        1122 | BEGIN                                                              |
| binlog.000001 | 1122 | View_change    |         1 |        1261 | view_id=14724832985483517:2                                        |
| binlog.000001 | 1261 | Query          |         1 |        1326 | COMMIT                                                             |
+---------------+------+----------------+-----------+-------------+--------------------------------------------------------------------+
```

s3 的加入与 s2 步骤一致

## TroubleShooting

## Reference
- https://dev.mysql.com/doc/refman/8.0/en/binary-installation.html
- https://dev.mysql.com/doc/refman/8.0/en/group-replication-getting-started.html
