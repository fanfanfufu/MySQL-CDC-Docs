## 一 MySQL有二进制日志相关信息查看

### 1.1 在MySQL命令行界面中通过下面的命令进行查看

```mysql
show variables like '%log_bin%';
```

如果查询到底数据中log_bin对应的value为ON的话, 说明MySQL的二进制日志已经开启了(一般情况下, MySQL的二进制日志都是默认开启的)．

![image-20211010185815210](../MySQL-CDC/MySQL-CDC001/show-log_bin截图.png)

并且从上图中可以看出，当前MySQL的binlog日志文件的base名称为binlog, 其所在的文件夹为/var/lib/mysql 

### 1.2 其余查看binlog日志文件信息的命令

```mysql
-- 1. 查看MySQL服务器中所有的二进制日志文件
show master logs;或者 show binary logs;
```

![image-20211010230121506](../MySQL-CDC/MySQL-CDC001/show-binary-logs截图.png)

​		从上图可以看到该命令的结果可以得到二进制日志文件(binlog)名, 文件大小以及是否加密这三个信息

```mysql
-- 2. 查看当前的二进制日志的信息
show master status;
```

![image-20211010230328322](../MySQL-CDC/MySQL-CDC001/show-master-status截图.png)

​		该命令可以得到当前最新的二进制日志文件(binlog)名, 最新的文件偏移量以及写入binlog文件的库和被忽略的库等信息

```mysql
-- 3. 查看binlog相关配置信息
show variables like '%binlog%';
```

![image-20211010230643120](../MySQL-CDC/MySQL-CDC001/binlog信息截图.png)

​		该命令可以看到很多关于binlog的相关配置, 就不一一阐述了. 重点可以关注一下binlog_format这个信息, 这是binlog文件的输出格式, MySQL主从同步,以及CDC都会需要关注此变量的值.

```mysql
-- 4. 关闭当前binlog并打开一个新的binlog
flush logs;
```

![image-20211011215435367](../MySQL-CDC/MySQL-CDC001/flush-logs截图.png)

## 二 MySQL二进制日志内容查看

### 2.1 MySQL二进制日志格式

​		MySQL二进制日志的格式有三种: STATEMENT, ROW, MIXED.

​		(1) STATEMENT: 基于SQL语句的格式. binlog中存储的是执行的语句.

​		(2) ROW: 基于每行的记录格式. 有多少行数据被修改,就会有多少行相关的信息记录到binlog中. 是目前的MySQL默认的binlog格式.

​		(3) MIXED: 混合模式.MySQL根据情况记录SQL语句或者是每行数据信息到binlog中.

### 2.2 设置binlog_format

#### 2.2.1 SESSION级别设置

​		MySQL命令行中输入命令:

```mysql
SET binlog_format='STATEMENT';
-- 或
SET binlog_format='ROW';
-- 或
SET binlog_format='MIXED';
```

![image-20211012215722034](../MySQL-CDC/MySQL-CDC001/session级别设置binlog_formate.png)

​		在MySQL命令行界面中, 输入: 

```mysql
SET GLOBAL binlog_format = 'STATEMENT';
-- 或
SET GLOBAL binlog_format = 'ROW';
-- 或
SET GLOBAL binlog_format = 'MIXED';
```

​		全局设置需要退出当前的MySQL命令行会话, 才能使得设置生效.

![image-20211012220405165](../MySQL-CDC/MySQL-CDC001/全局级别设置binlog_format.png)

### 2.3 binlog日志内容查看

​		查看binlog的内容, 需要在终端命令行中使用mysqlbinlog工具来解析查看.

#### 2.3.1 以STATEMENT模式进行记录

![image-20211012221722598](../MySQL-CDC/MySQL-CDC001/statement模式记录binlog.png)

#### 2.3.2 参考statement模式记录的方式以ROW模式记录一次UPDATE操作

![image-20211012222221305](../MySQL-CDC/MySQL-CDC001/row模式记录binlog.png)

#### 2.3.3 使用MIXED模式记录一次update操作

![image-20211012222522521](../MySQL-CDC/MySQL-CDC001/mixed模式记录binlog.png)

#### 2.3.4 查看binlog内容

​		退出MySQL命令行, 来到终端命令行, 输入下面的命令:

```shell
mysqlbinlog (path of binlog file)
例如在本例中:
mysqlbinlog /var/lib/mysql/binlog.000007
```

​		就可以在控制台中看到其前面对salary表的update操作所产生的二进制日志了.

​		(1) statement模式的update日志

![image-20211012223639569](../MySQL-CDC/MySQL-CDC001/statement模式的update日志.png)

​		(2) row模式的update日志

![image-20211012223918153](../MySQL-CDC/MySQL-CDC001/row模式的update日志.png)

​		(3) mixed模式的update日志

![image-20211012224059370](../MySQL-CDC/MySQL-CDC001/mixed模式的update日志.png)

#### 2.3.5 binlog日志内容解析

​		前面三种模式下update的日志记录可以看到一些比较共同的地方, 这里对其中的一部分进一步说明:

```tex
# at 1201
#211012 14:24:30 server id 1  end_log_pos 1354 CRC32 0x23f5462f         Query   thread_id=20    exec_time=0     error_code=0
```

​		(1) '# at 1201': '# at'后面的数字表示的是binlog文件中事件记录的起始位置(文件偏移量)

​		(2) '#211012 14:24:30' : 时间开始的的时间戳

​		(3) 'server id': 表示产生该事件的MySQL服务器的server_id的值, server_id是在MySQL的配置文件(my.cnf或者my.ini)中配置的.

​		(4) 'end_log_pos' : 后面的数字以及看着像寄存器地址的十六进制字符串就是下一个事件记录的起始位置.

​		(5) 'Query': 事件类别的名称. 查看binlog文件中的事件需要在MySQL命令行中, 使用命令 show binlog event 进行查看.

```mysql
show binlog events in 'binlog.000007';
```

![image-20211012232015425](../MySQL-CDC/MySQL-CDC001/binlog日志文件事件查看.png)

​		(6) 'thread_id': 表示执行事件的线程的编号.

​		(7) 'exex_time': 在主节点上, 表示事件的执行耗时; 在从节点上表示从节点执行事件的最终时间与主节点上时间开始执行的时间之间的时间差, 该时间差可以作为衡量从节点相对于主节点滞后时长的指标.

​		(8) 'error_code': 时间的执行结果. 为0意味着执行成功.

#### 2.3.6 查看binlog指定可选项

​		在使用mysqlbinlog工具查看binlog内容时, 对于ROW模式所产生的日志内容, 其数据内容无法查看详情. 当想要查看详情时,就需要在命令中加上一些可选项了.

```shell
mysqlbinlog /var/lib/mysql/binlog.00007 --verbose
或者:
mysqlbinlog /var/lib/mysql/binlog.00007 --v
```

![image-20211012230127447](../MySQL-CDC/MySQL-CDC001/查看row模式的行数据记录.png)

如果不想看到被编码后的行数据记录, 可以再增加可选项 --base64-output="decode-rows"

```
mysqlbinlog /var/lib/mysql/binlog.000007 --verbose --base64-output="decode-rows"
```

![image-20211012230519094](../MySQL-CDC/MySQL-CDC001/base64-ouput.png)

​		一般情况下,一个binlog日志文件会有很多行, 当只想查看其中一部分的时候, 就可以根据时间或者文件位置进行过滤筛选, 相关的指令可选项有:

​		(1) 根据时间范围: --start-datetime="2021-10-12 14:18:00" 和 --stop-datetime="2021-10-12 14:22:00". 二者可以单独用,也可以同时使用

​		(2) 根据文件偏移量: --start-position=596 和 --stop-position=1021, 二者可以单独用,也可以同时使用.