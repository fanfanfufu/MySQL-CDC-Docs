# 构建 MySQL 和 Postgres 上的 Streaming ETL

通过在Windows上借助WSL来复刻这个教程, 来了解和学习Flink CDC, 初步入门. 整个工程的架构图如下

![image-20211205230853304](.\MySQL-CDC009\image-000.png)

## 一 准备阶段

### 1.1 flink运行相关准备

#### 1.1.1 flink下载

​		在[官方的下载页面](https://flink.apache.org/zh/downloads.html)下载flink的版本,我选择的是下载1.13.2的二进制压缩包

![image-20211204201943869](.\MySQL-CDC009\image-001.png)

​		在WSL中对其进行解压, 并进入到解压出来的目录中

```
tar -xzf flink-1.13.2-bin-scala_2.11.tgz
cd flink-1.13.2/
```

​		便得到了带运行的flnk环境了

![image-20211205230451559](.\MySQL-CDC009\image-002.png)

#### 1.1.2 flink相关connector下载

​		下载好elasticsearch、MySQL、postgresql对应的connector

- ​		[flink-sql-connector-elasticsearch7_2.11-1.13.2.jar](https://repo.maven.apache.org/maven2/org/apache/flink/flink-sql-connector-elasticsearch7_2.11/1.13.2/flink-sql-connector-elasticsearch7_2.11-1.13.2.jar)

- ​		[flink-sql-connector-mysql-cdc-2.2-SNAPSHOT.jar](https://repo1.maven.org/maven2/com/ververica/flink-sql-connector-mysql-cdc/2.2-SNAPSHOT/flink-sql-connector-mysql-cdc-2.2-SNAPSHOT.jar)

- ​		[flink-sql-connector-postgres-cdc-2.2-SNAPSHOT.jar](https://repo1.maven.org/maven2/com/ververica/flink-sql-connector-postgres-cdc/2.2-SNAPSHOT/flink-sql-connector-postgres-cdc-2.2-SNAPSHOT.jar)

  下载好后，将这三个connector放入解压出来的flink-1.13.2/lib 目录中即可完成connector的相关准备

![image-20211205234338365](.\MySQL-CDC009\image-003.png)

### 1.2 docker相关准备

#### 1.2.1 编写docker-compose.yml文件

​		文件内容如下：

```yaml
version: '2.1'
services:
  postgres:
    image: debezium/example-postgres:1.1
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_PASSWORD=1234
      - POSTGRES_DB=postgres
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
  mysql:
    image: debezium/example-mysql:1.1
    ports:
      - "3306:3306"
    environment:
      - MYSQL_ROOT_PASSWORD=123456
      - MYSQL_USER=mysqluser
      - MYSQL_PASSWORD=mysqlpw
  elasticsearch:
    image: elastic/elasticsearch:7.6.0
    environment:
      - cluster.name=docker-cluster
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - discovery.type=single-node
    ports:
      - "9200:9200"
      - "9300:9300"
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
  kibana:
    image: elastic/kibana:7.6.0
    ports:
      - "5601:5601"
```



#### 1.2.2 所需镜像拉取到本地

​		docker pull 拉取相关镜像到本地，提前准备好

```sh
docker pull debezium/example-postgres:1.1
docker pull debezium/example-mysql:1.1
docker pull elastic/elasticsearch:7.6.0
docker pull elastic/kibana:7.6.0
```

![image-20211205192923621](.\MySQL-CDC009\image-004.png)

### 1.3 数据相关准备

#### 1.3.1 运行起docker-compose

​		在docker-compose.yml文件所在的目录下执行命令：

```sh
docker-compose build
```

![image-20211205193119573](.\MySQL-CDC009\image-005.png)

```sh
docker-compose up -d
```

![image-20211205193155433](.\MySQL-CDC009\image-006.png)

通过 docker ps 可以查看容器是否启动成功

![image-20211205193227804](.\MySQL-CDC009\image-007.png)

还可以通过访问 http://localhost:5601，通过是否进入kibana初始化页面判断是否启动成功

![image-20211205193324491](.\MySQL-CDC009\image-008.png)

​		可以看出，相关数据库的容器都已成功启动并正常运行，接下来就可以准备数据了。

#### 1.3.2 初始化部分数据

​		可以通过两种方式写入数据到数据库中：

​		（1）在docker-compose文件所在目录中使用命令进入到对应的数据库的命令行中，使用postgresql的容器进行演示。

​		（2）在Windows中通过docker desktop客户端进入容器，再通过相关的命令进入到数据库命令行中，使用MySQL的容器进行演示。

##### 1.3.2.1 MySQL中数据准备

​		通过docker destop进入到MySQL的容器中，再输入下面的命令进入MySQL命令行：

```
mysql -u root -p (密码：123456)
```

​		建库建表

```mysql
-- 创建库
CREATE DATABASE mydb;

USE mydb;
-- 创建products表
CREATE TABLE products (
  id INTEGER NOT NULL AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  description VARCHAR(512)
);
ALTER TABLE products AUTO_INCREMENT = 101;
-- 创建orders表
CREATE TABLE orders (
  order_id INTEGER NOT NULL AUTO_INCREMENT PRIMARY KEY,
  order_date DATETIME NOT NULL,
  customer_name VARCHAR(255) NOT NULL,
  price DECIMAL(10, 5) NOT NULL,
  product_id INTEGER NOT NULL,
    -- Whether order has been placed
  order_status BOOLEAN NOT NULL 
) AUTO_INCREMENT = 10001;
```

![image-20211205194104246](.\MySQL-CDC009\image-009.png)

​		写入数据

```mysql
INSERT INTO products
VALUES (default,"scooter","Small 2-wheel scooter"),
       (default,"car battery","12V car battery"),
       (default,"12-pack drill bits","12-pack of drill bits with sizes ranging from #40 to #3"),
       (default,"hammer","12oz carpenter's hammer"),
       (default,"hammer","14oz carpenter's hammer"),
       (default,"hammer","16oz carpenter's hammer"),
       (default,"rocks","box of assorted rocks"),
       (default,"jacket","water resistent black wind breaker"),
       (default,"spare tire","24 inch spare tire");
       
INSERT INTO orders
VALUES (default, '2020-07-30 10:08:22', 'Jark', 50.50, 102, false),
       (default, '2020-07-30 10:11:09', 'Sally', 15.00, 105, false),
       (default, '2020-07-30 12:00:30', 'Edward', 25.25, 106, false);
```

![image-20211205194210789](.\MySQL-CDC009\image-010.png)

##### 1.3.2.2 postgresql中数据准备

​		在docker-compose.yml所在的目录下，使用如下命令进入postgresql命令行：

```sh
docker-compose exec postgres psql -h localhost -U postgres 
```

​		建表并写入数据

```sql
-- 创建表
CREATE TABLE shipments (
   shipment_id SERIAL NOT NULL PRIMARY KEY,
   order_id SERIAL NOT NULL,
   origin VARCHAR(255) NOT NULL,
   destination VARCHAR(255) NOT NULL,
   is_arrived BOOLEAN NOT NULL
 );
 ALTER SEQUENCE public.shipments_shipment_id_seq RESTART WITH 1001;
 ALTER TABLE public.shipments REPLICA IDENTITY FULL;
 -- 写入数据
 INSERT INTO shipments
 VALUES (default,10001,'Beijing','Shanghai',false),
        (default,10002,'Hangzhou','Shanghai',false),
        (default,10003,'Shanghai','Hangzhou',false);
```

![image-20211205194857733](.\MySQL-CDC009\image-011.png)

## 二 启动Flink以及Flink中的数据准备

### 2.1 启动Flink

​		在解压出来的flink-1.13.2目录下，输入如下命令即可启动flink：

```bash
./bin/start-cluster.sh 
```

​		执行效果如下，说明flink已经成功启动并正常运行了：

![image-20211204203107209](.\MySQL-CDC009\image-012.png)

​		也可以访问 http://localhost:8081/ ，看到Flink的Web UI页面，也说明Flink启动成功了。

![image-20211205234745050](.\MySQL-CDC009\image-013.png)

### 2.2 启动Flink-SQL客户端

​		在flink的主目录下输入命令：

```bash
./bin/sql-client.sh
```

![image-20211205195423042](.\MySQL-CDC009\image-014.png)

### 2.3 创建相关数据表

​		首先，开启 checkpoint，每隔3秒做一次 checkpoint

```sql
SET execution.checkpointing.interval = 3s;
```

![image-20211205200643063](.\MySQL-CDC009\image-015.png)

​		然后输入Flink-SQL创建相关数据表

​		（1）创建相关的source表，分别对应MySQL中的products、orders表，postgresql中的shipments表

```sql
-- Flink SQL
CREATE TABLE products (
    id INT,
    name STRING,
    description STRING,
    PRIMARY KEY (id) NOT ENFORCED
  ) WITH (
    'connector' = 'mysql-cdc',
    'hostname' = 'localhost',
    'port' = '3306',
    'username' = 'root',
    'password' = '123456',
    'database-name' = 'mydb',
    'table-name' = 'products'
  );

CREATE TABLE orders (
   order_id INT,
   order_date TIMESTAMP(0),
   customer_name STRING,
   price DECIMAL(10, 5),
   product_id INT,
   order_status BOOLEAN,
   PRIMARY KEY (order_id) NOT ENFORCED
 ) WITH (
   'connector' = 'mysql-cdc',
   'hostname' = 'localhost',
   'port' = '3306',
   'username' = 'root',
   'password' = '123456',
   'database-name' = 'mydb',
   'table-name' = 'orders'
 );

CREATE TABLE shipments (
   shipment_id INT,
   order_id INT,
   origin STRING,
   destination STRING,
   is_arrived BOOLEAN,
   PRIMARY KEY (shipment_id) NOT ENFORCED
 ) WITH (
   'connector' = 'postgres-cdc',
   'hostname' = 'localhost',
   'port' = '5432',
   'username' = 'postgres',
   'password' = 'postgres',
   'database-name' = 'postgres',
   'schema-name' = 'public',
   'table-name' = 'shipments'
 );
```

![image-20211205211842395](.\MySQL-CDC009\image-016.png)

​		（2）创建sink表，是对应写入到es中的数据结构

```sql
-- Flink SQL
CREATE TABLE enriched_orders (
   order_id INT,
   order_date TIMESTAMP(0),
   customer_name STRING,
   price DECIMAL(10, 5),
   product_id INT,
   order_status BOOLEAN,
   product_name STRING,
   product_description STRING,
   shipment_id INT,
   origin STRING,
   destination STRING,
   is_arrived BOOLEAN,
   PRIMARY KEY (order_id) NOT ENFORCED
 ) WITH (
     'connector' = 'elasticsearch-7',
     'hosts' = 'http://localhost:9200',
     'index' = 'enriched_orders'
 );
```

![image-20211205211926641](.\MySQL-CDC009\image-017.png)

## 三 创建ETL任务将数据实时写入ElasticSearch

​		上一阶段已经在Flink中创建了相关的数据表，接下来就是创建实时任务，并进行实时的CDC-ETL测试了。

### 3.1 创建基于Flink-SQL的实时ETL任务

​		使用 Flink SQL 将订单表 `order` 与 商品表 `products`，物流信息表 `shipments` 关联，并将关联后的订单信息写入 Elasticsearch 中。

```sql
-- Flink SQL
INSERT INTO enriched_orders
SELECT o.*, p.name, p.description, s.shipment_id, s.origin, s.destination, s.is_arrived
FROM orders AS o
LEFT JOIN products AS p ON o.product_id = p.id
LEFT JOIN shipments AS s ON o.order_id = s.order_id;
```

​		Flink会将该SQL作为一个Job来调度执行，并为其生成了Job ID，如图所示：

![image-20211205212041222](.\MySQL-CDC009\image-018.png)

​		也可以通过 [Web UI](http://localhost:80801/running) 查看该任务的一些详细信息：

![image-20211205212142116](.\MySQL-CDC009\image-019.png)

### 3.2 在elasticsearch中创建相关索引

​		首先访问 http://localhost:5601/app/kibana#/management/kibana/index_pattern 创建 index pattern `enriched_orders`.

![image-20211205212355384](.\MySQL-CDC009\image-020.png)

​		然后就可以在 http://localhost:5601/app/kibana#/discover 看到写入的数据了.

![image-20211206230647362](.\MySQL-CDC009\image-021.png)

### 3.3 测试实时任务的效果

​		接下来就可以通过在MySQL、PostgreSQL中操作数据，每操作一次，然后再kibana中刷新一次，就能看到数据被实时写入到elasticsearch中的效果了。

​		（1）首先在MySQL中插入一条数据

```sql
INSERT INTO orders VALUES (default, '2020-07-30 15:22:00', 'Jark', 29.71, 104, false);
```

![image-20211205215053283](.\MySQL-CDC009\image-022.png)

​		（2）在PostgreSQL中插入一条数据

```sql
INSERT INTO shipments VALUES (default,10004,'Shanghai','Beijing',false);
```

![image-20211205215237070](.\MySQL-CDC009\image-023.png)

![image-20211205215335280](.\MySQL-CDC009\image-024.png)

​		（3）在MySQL中更新一条数据

```sql
UPDATE orders SET order_status = true WHERE order_id = 10004;
```

![image-20211205215508427](.\MySQL-CDC009\image-025.png)

​		（4）在PostgreSQL中更新一条数据

```sql
UPDATE shipments SET is_arrived = true WHERE shipment_id = 1004;
```

![image-20211205215636233](.\MySQL-CDC009\image-026.png)

​		（5）先删除PostgreSQL中的一条数据

```sql
DELETE FROM shipments WHERE order_id = 10004;
```

![image-20211206231427332](.\MySQL-CDC009\image-027.png)

![image-20211206231455832](.\MySQL-CDC009\image-028.png)

​		在MySQL中没有删除相关数据，而postgreSQL中删了相关数据后，ES中的数据就会缺少shipments表的字段的内容。

​		（6）在MySQL中删除一条数据

```sql
DELETE FROM orders WHERE order_id = 10004;
```

![image-20211205215809719](.\MySQL-CDC009\image-029.png)

​		（7）PostgreSQL早于MySQL新增一条数据

```sql
INSERT INTO shipments VALUES (default,10005,'重庆','成都',false);
```

![image-20211206231643815](.\MySQL-CDC009\image-030.png)

​		刷新kibana中的页面，并没有相关的数据出现，再往MySQL中新增一条数据

```sql
INSERT INTO orders VALUES (default, '2021-12-05 22:22:22', 'fanfanfufu', 74, 225, false);
```

![image-20211206232055293](.\MySQL-CDC009\image-031.png)

​		可以看到，在MySQL中新增了一条order_id=10005的数据后，才将postgreSQL中新增的与order_id=10005这条数据相关的shipment_id=10005的数据关联出来，可以看到刷新kibana后，新增加的数据，同时包含了来自MySQL-order表和PostgreSQL-shipment表中新增的数据的字段信息。

## 四 结束运行

4.1 结束并清除数据库容器

​		在docker-compose.yml的目中下，使用命令便可结束相关的数据库容器，并进行删除

```bash
docker-comopse down
```

![image-20211205222329954](.\MySQL-CDC009\image-032.png)

4.2 停掉Flink

```bash
./bin/stop-cluster.sh
```

![image-20211205222446561](.\MySQL-CDC009\image-033.png)

## 五 总结

​		该例子简单的演示了通过flink-cdc，监听数据的数据变化，然后进行实时的ETL的过程。通过实际对该过程的上手操作，可以简单的学习flink-sql的相关知识。但是由于对于flink-cdc的原理以及工作机制并未涉及，通过该案例还无法对flink-cdc有较为深刻的学习。

## 六 参考文档

[Flink CDC 系列 - 构建 MySQL 和 Postgres 上的 Streaming ETL](https://flink-learning.org.cn/article/detail/ae625d6b8dcb26be91bd076d2d091d8c)

