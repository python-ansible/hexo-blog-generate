---
title: mysql-pt 套件使用 001
tags: [mysql]
toc: true
mathjax: true
date: 2019-02-26 11:13:07
categories: db
---
## percona-xtrabackup 
**不停机备份工具**  
`yum install -y percona-xtrabackup` 

连接mysql服务器  
`xtrabackup --user=DVADER --password=14MY0URF4TH3R --backup --target-dir=/data/backups/`  

创建备份  
`xtrabackup --backup --target-dir=/data/backups/`  

准备备份 -- 处理一致性  
`xtrabackup --prepare --target-dir=/data/backups/`  

恢复备份  
`rsync -avrP /data/backups/ /var/lib/mysql/`  
`rsync -avz /root/wzl/backups/ root@100.68.8.27:/var/lib/mysql/`  

修改权限  
`chown -R mysql:mysql /var/lib/mysql` 

跳过单个主备不一致
```
set global sql_slave_skip_counter=1;
start slave;  
```

查看每个表的数据大小  
**这个不准不要用！**  
`select table_name,table_rows,data_length/1024/1024 "data_length" from information_schema.tables where table_schema in ('neutron','keystone') order by table_rows desc;`

## percona-toolkit
依赖  
```
rpm -qa perl-DBI perl-DBD-MySQL perl-Time-HiRes perl-IO-Socket-SSL  
yum install -y perl-DBI perl-DBD-MySQL perl-Time-HiRes perl-IO-Socket-SSL perl-Digest perl-Digest-MD5 perl-TermReadKey  
```

安装  
```
https://www.percona.com/downloads/percona-toolkit/LATEST/  
rpm -ivh percona-toolkit  
rpm -ivh percona-toolkit-debuginfo  
```

确认  
```
pt-query-digest -help  
pt-table-checksum -help  
```

权限  
```
CREATE USER 'new_root'@'%' IDENTIFIED BY '***';    
GRANT ALL PRIVILEGES ON *.* TO 'new_root'@'%' IDENTIFIED BY '***' WITH GRANT OPTION;  
```

**--create-replicate-table** 只有第一次需要 

检查数据库的一致性  
`pt-table-checksum --nocheck-replication-filters --no-check-binlog-format --databases=neutron,keystone h=192.168.1.102,u=root,p=root,P=3306`  

检查表的一致性  
`pt-table-checksum --nocheck-replication-filters --no-check-binlog-format --replicate=neutron.checksums --create-replicate-table --databases=neutron --tables=haha h=192.168.1.101,u=root,p=123456,P=3306`  

只打印  
`pt-table-sync --replicate=neutron.checksums h=192.168.1.101,u=root,p=123456 h=192.168.1.102,u=root,p=123456 --print`  

执行修复  
`pt-table-sync --replicate=neutron.checksums h=192.168.1.101,u=root,p=123456 h=192.168.1.102,u=root,p=123456 --execute`  

## binlog
以二进制日志记录了所有的 DDL(数据定义) 语句 和 DML(数据操作) 语句。  
语句以事件保存，不包括查询语句，主要用于数据的恢复和复制。
```
show binary logs;
mysqlbinlog mysqld-bin.000001
-d -database
mysqlbinlog -d neutron mysqld-bin.000001 > neutron.txt
-D --disable-log-bin 禁用
mysqlbinlog -D mysqld-bin.000001
-s --short-form 只输出sql
--start-datetime
--stop-datetime
--start-position
--stop-position
```

**远程读取** 
```
-R --read-from-remote-server  
-h --host  
mysqlbinlog -R -h 192.168.0.1 -p root mysqld-bin.000001
```

**批量删除表**
```
Select CONCAT( 'drop table ', task_id, ';' )FROM task Where start_time = '1999-01-01 01' and end_time < '2019-01-01 01';
```

## event_scheduler
```
SET GLOBAL event_scheduler=1;
USE keystone;

DELIMITER $$
CREATE EVENT IF NOT EXISTS token_expire_remove ON SCHEDULE EVERY 1 DAY STARTS TIMESTAMP '2019-03-20 02:00:00'
DO
BEGIN
START TRANSACTION;
SET @timenow=now();
DELETE FROM token WHERE expires < @timenow;
COMMIT;
END $$
DELIMITER ;
-------------------------------
SET GLOBAL event_scheduler=1;
USE bonree;

DELIMITER $$
CREATE EVENT IF NOT EXISTS table_expire_remove ON SCHEDULE EVERY 1 DAY STARTS TIMESTAMP '2019-03-20 02:00:00'
DO
BEGIN
START TRANSACTION;
SET @timenow=now();
SET @enddate=date_add(@timenow,interval -1 quarter);
SET @exeSql="SELECT CONCAT('DROP TABLE ', task_id, ';')FROM task WHERE start_time='1999-01-01 01' AND end_time < @enddate;";
PREPARE stmt FROM @exeSql;
EXECUTE stmt;
DELETE FROM task WHERE start_time='1999-01-01 01' AND end_time < @enddate;
COMMIT;
END $$
DELIMITER ;
-------------------------------
查看:
show processlist;
持久化:
event_scheduler=ON
```

**压缩binlog**
`tar -zcvf mysql-bin.000001.tar.gz /var/lib/mysql/mysql-bin.000001;`
`find -name 'mysql-bin.0000*' ! -name '*.gz' -exec tar -zcvf {}.tar.gz {} \;`
`find -name 'mysql-bin.0000*' ! -name '*.gz' -exec echo {} \;`

## mysql 导入导出
导出表结构
mysqldump -uroot -proot -d neutron > neutron.sql
导出单张表
mysqldump -uroot -proot neutron pa_igws > nsp.sql
导入表结构
mysql -uroot -proot < /tmp/neutron.sql

## mysql 慢查询清理
show tables like '%slow%';
CREATE TABLE slow_log_bak LIKE slow_log;
RENAME TABLE slow_log to slow_log_old,slow_log_bak to slow_log;
drop table slow_log_old;

临时
set global log_queries_not_using_indexes=0;
show global variables like 'log%';
永久
slow_query_log=1
long_query_time=3
slow_query_log_file=/var/lib/mysql/slow_query.log
log-queries-not-using-indexes=0 默认就是不开启的
