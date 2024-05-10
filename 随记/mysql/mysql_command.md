# Mysql常用命令

安装与启动

```
sudo yum install -y mariadb mariadb-server

mysql.server start 
sudo systemctl stop mariadb
```

授权与撤销

```
SHOW GRANTS FOR 'test'@'%'
grant all privileges on eyes_sea.* to 'sea'@'localhost' identified by '123' with grant option;
flush privileges;

select host, user, password from user;
REVOKE ALL PRIVILEGES ON *.* FROM 'sea'@'%';
delete from user where user='sea';
flush privileges;
```

连接占用

```
mysqladmin -utest -ptest status
show global variables like '% connect%'

set GLOBAL max_connections=300;
flush privileges;
```

建库

```
CREATE DATABASE IF NOT EXISTS aa;
```

>MySQL存储emoji表情需要在MySQL的服务端设置为utf8mb4，
show variables like '%char%'查看字符集，默认的是latin1，
需要在my.conf 中的相应模块中加入配置，然后重启 MySQL。
---

```
[client]
default-character-set=utf8mb4

[mysql]
default-character-set=utf8mb4

[mysqld]
character-set-server=utf8mb4
collation-server=utf8mb4_general_ci
```

>字符集更改成utf8mb4
才能插入emoji表情。
---

索引操作

```
show index from table_name
ALTER TABLE aa DROP INDEX uk_name_dm_id;
ALTER TABLE `aa` ADD UNIQUE ( `uuid` );
ALTER TABLE `aa` ADD KEY `idx_search_key` (`search_key`);
```

修改操作

```
alter table 表名 drop column 列名

alter table [count].[dbo].[inst] add id int identity(1,1) not null // 添加自增列
alter table aa modify icon bigint(20) unsigned NOT NULL COMMENT '图标';
alter table aa add `uuid` varchar(64) COMMENT 'UUID' after id;
```

聚合统计

```
SELECT SUM(record_count) from (SELECT distinct table_name, record_count FROM db_meta_info) AS a;
SELECT SUM(sum_record_count) from (SELECT SUM(record_count) AS sum_record_count FROM db_meta_info GROUP BY table_name) AS a;
```

备份迁移

```
create table aa_bak(select * from aa);
mysqldump -utest -ptest --databases dd >dd.sql
mysqldump -utest -ptest dd aa>aa.sql

insert into dd.aa select * from dd1.aa1;
mysql -usea -p'123' dd < dd.sql
source /home/magneto/aa.sql
select * from test_mysql_10000000_18_1551316435883 into outfile "test_mysql_10000000.txt";
LOAD DATA LOCAL INFILE '/home/magneto/aa.csv' INTO TABLE aa FIELDS TERMINATED BY ',' LINES TERMINATED BY '\r\n' IGNORE 1 LINES;
sudo find / -name test_mysql_10000000.txt // 查找文件
```