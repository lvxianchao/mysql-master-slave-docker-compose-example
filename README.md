# MySQL 8.0.34 搭建主从示例

> 此项目用于记录 Docker Compose 方式搭建 MySQL 8.0.34 的主从同步。

## 端口

主库：33061
从库：33062

## 运行

将项目代码克隆到本地以后，进入到工作目录中，执行：

```shell
docker compose up -d
```

## 主库操作

```shell
# 进入到主库容器内
docker exec -it mysql-master /bin/bash

# 连接 MySQL
mysql -u root -p

# 创建同步用户
CREATE USER 'replicator' @'%' IDENTIFIED BY '123123';

# 授权
GRANT REPLICATION SLAVE ON *.* TO 'replicator'@'%';

# 刷新
FLUSH PRIVILEGES;

# 查看主库状态
SHOW MASTER STATUS;
```

在执行了 `SHOW MASTER STATUS;` 以后，应该会得到如下类似的信息：

```mysql
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000003 |     1016 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```

需要记录下其中 `File` 和 `Position` 两个字段的值，稍后在从库配置上需要用到。

## 从库操作

### 配置主库信息

```mysql
CHANGE MASTER TO MASTER_HOST = 'master',
MASTER_USER = 'replicator',
MASTER_PASSWORD = '123123',
MASTER_LOG_FILE = 'mysql-bin.000003',
MASTER_LOG_POS = 1016;
```

需要将上面的 `MASTER_LOG_FILE` 和 `MASTER_LOG_POS` 两个字段的值修改为刚刚在主库中看到的值;

### 开启从库

```mysql
START SLAVE;
```

### 查看状态

```mysql
SHOW SLAVE STATUS;
```

在执行 `SHOW SLAVE STATUS;` 以后可以看到以下输出：

```mysql
mysql> show slave status \G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for source to send event
                  Master_Host: master
                  Master_User: replicator
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000003
          Read_Master_Log_Pos: 1016
               Relay_Log_File: b3ce74c776e1-relay-bin.000004
                Relay_Log_Pos: 326
        Relay_Master_Log_File: mysql-bin.000003
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 1016
              Relay_Log_Space: 1077
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 1
                  Master_UUID: 561e3ab5-3ffd-11ee-a59b-0242c0a89302
             Master_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Replica has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 
            Executed_Gtid_Set: 
                Auto_Position: 0
         Replicate_Rewrite_DB: 
                 Channel_Name: 
           Master_TLS_Version: 
       Master_public_key_path: 
        Get_master_public_key: 0
            Network_Namespace: 
1 row in set, 1 warning (0.00 sec)
```

主要观察以下三项的值都正确，则配置成功，可以去主库进行某些操作看看能否同步到从库：

1. `Slave_IO_State` 的值为 `Waiting for source to send event`。
2. `Slave_IO_Running` 的值为 `Yes`。
3. `Slave_SQL_Running` 的值为 `Yes`。

如果以上三项的值不符合预期，可以通过查看 `Last_Error`、`Last_IO_Error`、`Last_SQL_Error` 等字段查看错误。
