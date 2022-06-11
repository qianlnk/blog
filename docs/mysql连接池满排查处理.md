# mysql连接池满排查处理

1. 查询连接情况

    ```
    root@localhost > show processlist;
    …...
    1001 rows in set (0.00 sec)
    root@localhost > show variables like '%proces%';
    Empty set (0.00 sec)
    ```

2. 检查参数

    ```
    root@localhost > show global status like 'Max_used_connections';
    +----------------------+-------+
    | Variable_name | Value |
    +----------------------+-------+
    | Max_used_connections | 1001 |
    +----------------------+-------+
    1 row in set (0.00 sec)
    ```

3. 通过命令生成杀进程脚本

    ```
    root@localhost > select concat('KILL ',id,';') from information_schema.processlist where user=’sam' into outfile '/tmp/a.txt
    ```
 
    脚本内容如下：
        
    ```
        +------------------------+
        | concat('KILL ',id,';') |
        +------------------------+
        | KILL 31964612; |
        | KILL 31964609; |
        | KILL 31964611; |
        …...
        | KILL 31966619; |
        | KILL 31966620; |
        +------------------------+
        991 rows in set (0.02 sec)
        root@localhost >
    ```

4. 执行上面生成的KILL脚本

    ```
    root@localhost > source /tmp/a.txt
    Query OK, 0 rows affected (0.00 sec)
    Query OK, 0 rows affected (0.00 sec)
    ……
    ```

5. 检查连接状况，恢复正常

    ```
    root@localhost > show processlist;
    ```

6. 修改Max_used_connections参数（注：记得要修改my.cnf文件，下次重启动后仍然有效）

    ```
    mysql> set GLOBAL max_connections=2000;
    Query OK, 0 rows affected (0.00 sec)
    
    mysql> show variables like '%max_connections%';
    +-----------------+-------+
    | Variable_name | Value |
    +-----------------+-------+
    | max_connections | 2000 |
    +-----------------+-------+
    1 row in set (0.00 sec)
    ```