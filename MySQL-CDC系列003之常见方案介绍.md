常见的CDC开源方案包括: 

​		DataX, Debezium, Canal, Maxwell, Flink-CDC, Sqoop, kettle, Oracle Goldengate, 相关的对比分析如下:

|           |             Flink-CDC              | Canal  | Maxwell |                      Debezium                      |  DataX   |  Sqoop   |        Kettle        | Oracle Goldengate |
| :-------: | :--------------------------------: | :----: | :-----: | :------------------------------------------------: | :------: | :------: | :------------------: | :---------------: |
|  CDC机制  |                日志                |  日志  |  日志   |                        日志                        |   查询   |   查询   |         查询         |       日志        |
| 增量同步  |                支持                |  支持  |  支持   |                        支持                        |  不支持  |   支持   |        不支持        |       支持        |
| 全量同步  |                支持                | 不支持 | 不支持  |                        支持                        |   支持   |   支持   |         支持         |       支持        |
| 全量+增量 |                支持                | 不支持 | 不支持  |                        支持                        |  不支持  |  不支持  |        不支持        |       支持        |
| 断点续传  |                支持                |  支持  |  支持   |                        支持                        |  不支持  |  不支持  |        不支持        |       支持        |
|  分布式   |                支持                |  支持  | 不支持  |                        支持                        |  不支持  |   支持   |         支持         |       支持        |
|  数据库   | MySQL<br />PostgreSQL<br />MongoDB | MySQL  |  MySQL  | MySQL<br />PostgreSQL<br />MongoDB<br />SQL Server | 种类齐全 | 种类齐全 |       种类齐全       |      Oracle       |
| 生态情况  |                活跃                |  活跃  |  活跃   |                        活跃                        |   活跃   | 目前只读 | pentaho-kettle，活跃 |     找不到了      |

- 对比增量同步能力：
  - 基于日志的方式，可以很好的做到增量同步；
  - 而基于查询的方式是很难做到增量同步的。
- 对比全量同步能力，基于查询或者日志的 CDC 方案基本都支持，除了 Canal和Maxwell。
- 而对比全量 + 增量同步的能力，只有 Flink CDC、Debezium、Oracle Goldengate 支持较好。
- 默认是否支持分布式方面，Maxwell和DataX是不支持的。
- 在数据转换 / 数据清洗能力上，当数据进入到 CDC 工具的时候是否能较方便的对数据做一些过滤或者清洗，甚至聚合？
  - 在 Flink CDC 上操作相当简单，可以通过 Flink SQL 去操作这些数据；
  - 但是像 DataX、Debezium 等则需要通过脚本或者模板去做，所以用户的使用门槛会比较高。