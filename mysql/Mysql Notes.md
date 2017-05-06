### MySQL修改数据库编码:

```

ALTER TABLE order_fsm_event_logs CONVERT TO CHARACTER SET utf8;

```



```

ALTER TABLE `order_fsm_event_logs` MODIFY `payload` Text CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NULL;

```



### MySQL局域网访问

1. 首先修改mysql绑定的host, comment掉my.conf[/etc/mysql/mysql.conf.d/mysqld.cnf]中的这一行:`bind-address = 127.0.0.1`.

2. 执行一下sql语句:

```



mysql>use mysql;

mysql>update user set host = '%' where user = 'root';

mysql>GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY '123456' WITH GRANT OPTION;

mysql>FLUSH PRIVILEGES;

```



### MySQL导出数据到文件

- 方案一

``` sql

select * from couriers INTO OUTFILE '/home/dev/out.csv' FIELDS TERMINATED BY ','ENCLOSED BY '"' LINES TERMINATED BY '\n';

```

这个方案有一个问题，需要MySQL当前登录的用户有访问文件的权限。否则会抛出错误：`Access denied for user 'usr'@'%'`



- 方案二

```

mysql -h rds-test-env.c8pr5ecdyixx.us-east-1.rds.amazonaws.com -udiamond_admin -p0UNv8XPHQkXsV2w6 diamond -B -e "select * from couriers;" | sed "s/'/\'/;s/\t/\",\"/g;s/^/\"/;s/$/\"/;s/\n//g" > vehicle_categories.csv

```



### MySQL 创建相同结构的表，并导入数据

``` sql

CREATE TABLE backup LIKE source;

INSERT INTO backup SELECT * from source;

```

