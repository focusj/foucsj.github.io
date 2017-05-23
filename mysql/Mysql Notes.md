### MySQL修改数据库编码:

```

ALTER TABLE order_fsm_event_logs CONVERT TO CHARACTER SET utf8;

```



```

ALTER TABLE `order_fsm_event_logs` MODIFY `payload` Text CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NULL;

```


### 添加Column
```
ALTER TABLE `entity_user` ADD COLUMN `user_review_score` int(11) NOT NULL DEFAULT '0',
                          ADD COLUMN `positive_ratings` varchar(10) COLLATE utf8mb4_unicode_ci NOT NULL DEFAULT '';
```


### 修改MySQL绑定host

首先修改mysql绑定的host, 改掉my.conf[/etc/mysql/mysql.conf.d/mysqld.cnf]中的这一行:`bind-address = 0.0.0.0`。重启服务，局域网可以访问MySQL。


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

### MySQL in查询并按照in中的数据排序

``` sql
select * from table where id in (1, 2, 3) order by field(id, 1, 2, 3);
```

### MySQL in 最多能包含多少数据？
The number of values in the IN list is only limited by the max_allowed_packet value.

- max_allowed_packet
max_allowed_packet default value: 4194304, min value: 1024, max value: 1073741824

The maximum size of one packet or any generated/intermediate string, or any parameter sent by the mysql_stmt_send_long_data() C API function. The default is 4MB.

The packet message buffer is initialized to net_buffer_length bytes, but can grow up to max_allowed_packet bytes when needed. This value by default is small, to catch large (possibly incorrect) packets.

You must increase this value if you are using large BLOB columns or long strings. It should be as big as the largest BLOB you want to use. The protocol limit for max_allowed_packet is 1GB. The value should be a multiple of 1024; nonmultiples are rounded down to the nearest multiple.

When you change the message buffer size by changing the value of the max_allowed_packet variable, you should also change the buffer size on the client side if your client program permits it. The default max_allowed_packet value built in to the client library is 1GB, but individual client programs might override this. For example, mysql and mysqldump have defaults of 16MB and 24MB, respectively. They also enable you to change the client-side value by setting max_allowed_packet on the command line or in an option file.

The session value of this variable is read only. The client can receive up to as many bytes as the session value. However, the server will not send to the client more bytes than the current global max_allowed_packet value. (The global value could be less than the session value if the global value is changed after the client connects.)