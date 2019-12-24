# Getting Started with OAP
[![Build Status](https://travis-ci.org/Intel-bigdata/OAP.svg?branch=master)](https://travis-ci.org/Intel-bigdata/OAP)

* [Prerequisites](#Prerequisites)
* [Configuration](#Configuration)
* [How to Use OAP](#How_to_Use_OAP)

## Prerequisites
Before getting started with OAP on Spark, we recommand you have set up a Hadoop cluster with YARN which runs well. Running Spark on YARN requires a binary distribution of Spark which is built with YARN support. We provide you with the the pre-built [Spark]() and [OAP](), so you can download both of them to your master machine.
## Configuration
There are two deploy modes that can be used to launch Spark applications on YARN. A common deployment strategy is to submit your application from a gateway machine that is physically co-located with your worker machines. In this setup, `client` mode is appropriate. In `client` mode, the driver is launched directly within the `spark-submit `process which acts as a client to the cluster. The input and output of the application is attached to the console. Thus, this mode is especially suitable for applications that involve the REPL (e.g. Spark shell).

To make Spark with OAP run well in `client` mode , We list required configurations in `$SPARK_HOME/conf/spark-defaults.conf`
```
spark.master                      yarn
spark.deploy-mode                 client
spark.sql.extensions              org.apache.spark.sql.OapExtensions
spark.files                       /<PATH_TO_OAP_JAR>/oap-0.6-with-spark-2.3.2.jar     # absolute path  
spark.executor.extraClassPath     ./oap-0.6-with-spark-2.3.2.jar                      # relative path
spark.driver.extraClassPath       /<PATH_TO_OAP_JAR>/oap-0.6-with-spark-2.3.2.jar     # absolute path
```
Alternatively, if your application is submitted from a machine far from the worker machines (e.g. locally on your laptop), it is common to use `cluster` mode to minimize network latency between the drivers and the executors. 
```
spark.master                      yarn
spark.deploy-mode                 cluster
spark.sql.extensions              org.apache.spark.sql.OapExtensions
spark.files                       /<PATH_TO_OAP_JAR>/oap-0.6-with-spark-2.3.2.jar     # absolute path    
spark.executor.extraClassPath     ./oap-0.6-with-spark-2.3.2.jar                      # relative path 
spark.driver.extraClassPath       ./oap-0.6-with-spark-2.3.2.jar                      # relative path
```
In addition to running on the YARN cluster managers, Spark also provides a simple standalone deploy mode. If install `Spark Standalone mode`, you simply place a compiled version of Spark and OAP on each node on the cluster.
```
spark.sql.extensions               org.apache.spark.sql.OapExtensions
spark.executor.extraClassPath      /<PATH_TO_OAP_JAR>/oap-0.6-with-spark-2.3.2.jar    # absolute path
spark.driver.extraClassPath        /<PATH_TO_OAP_JAR>/oap-0.6-with-spark-2.3.2.jar    # absolute path
```

In the following part, we will take `Spark on Yarn with Client Mode` for example to introduce you more configuration details to deploy Spark with OAP correctly.

```
spark.driver.memory
spark.executor.cores                              # less than yarn.scheduler.maximum-allocation-vcores
spark.executor.memory                             # less than yarn.scheduler.maximum-allocation-mb                              
spark.yarn.executor.memoryOverhead                # close to spark.memory.offHeap.size
spark.executor.instances                          # 1~2X of worker nodes         
spark.memory.offHeap.enabled                      
spark.memory.offHeap.size                         
```
Executor instances can be 1~2X of worker nodes. Considering the count of executor instances (N) on each node, executor memory can be around 1/N of each worker total available memory. Usually each worker has one or two executor instances. However, considering the cache utilization, one executor per worker node is recommended. Always enable offHeap memory and set a reasonable (the larger the better) size, as long as OAP's fine-grained cache takes advantage of offHeap memory, otherwise user might encounter weird circumstances.

After deployment and configuration, you can run by` bin/spark-sql, bin/spark-shell, bin/spark-submit, sbin/start-thriftserver or bin/pyspark. `
If failed to launch Spark with OAP, you need to check the logs to find the reason.
## How to Use OAP
### Use Index with OAP on Spark
After you have start Hadoop and YRAN, you can run Spark with the following example to try OAP index function with Spark shell.
```
. $SPARK_HOME/bin/spark-shell
```
Then create a table on corresponding HDFS data path, here take our data path ```hdfs:///user/oap/```for example.
```
> spark.sql(s"""CREATE TABLE oap_test (a INT, b STRING)
       USING parquet
       OPTIONS (path 'hdfs:///user/oap/')""".stripMargin)
```

```
> val data = (1 to 30000).map { i => (i, s"this is test $i") }.toDF().createOrReplaceTempView("t")
> spark.sql("insert overwrite table oap_test select * from t")
```
Create index with OAP Index
```
> spark.sql("create oindex index1 on oap_test (a)")
> spark.sql("show oindex from oap_test").show()
```
Use OAP Index
```
> spark.sql("SELECT * FROM oap_test WHERE a = 1").show()
```
Drop index
```
> spark.sql("drop oindex index1 on oap_test")
```
For  more detailed examples on OAP performance comparation, you can refer to this [page](https://github.com/Intel-bigdata/OAP/wiki/OAP-examples) for further instructions.

### Use DRAM to Cache with OAP
If you want to run OAP with cache function, firstly you should change some configurations into `$SPARK_HOME/conf/spark-defaults.conf`. OAP provides two media types to cache hot data: DRAM and DCPMM.

#### DRAM Cache Configuration in ` $SPARK_HOME/conf/spark-defaults.conf `
```
spark.memory.offHeap.enabled                true
spark.memory.offHeap.size                  <set a suitable size>
spark.sql.oap.parquet.data.cache.enable     true     #for parquet fileformat
spark.sql.oap.orc.data.cache.enable         true     #for orc fileformat
```
You can run Spark with the following example to try OAP cache function with DRAM. We recommand you use Thrift server
The Thrift JDBC/ODBC server implemented here corresponds to the HiveServer2 in Hive 1.2.1. You can test the JDBC server with the beeline script that comes with Spark.
In the [Index](#Use Index with OAP on Spark) part, we have create a table oap_test, next we will try OAP Cache.
When we use ```spark-shell``` to create table oap_test, ```metastore_db``` will be created in the current directory "$SPARK_HOME/bin/" , so firstly we need to run Thrift JDBC server in the same directory "$SPARK_HOME/bin/"
```
. $SPARK_HOME/sbin/start-thriftserver.sh
```
Now you can use beeline to test the Thrift JDBC/ODBC server, vsr211 is hostname, so you need change to your hostname.
```
./beeline -u jdbc:hive2://vsr211:10000       
```
When ```0: jdbc:hive2://vsr211:10000> ``` shows up, then you can directly use table oap_test, which is stored in the `default` database.
```
> show databases;
> use default;
> show tables;
> use oap_test;
```
next you can run query like
```
> SELECT * FROM oap_test WHERE a = 1;
> SELECT * FROM oap_test WHERE a = 2;
> SELECT * FROM oap_test WHERE a = 3;
```
Then you can find the cache metric with OAP TAB in the spark history Web UI.

![webUI](https://github.com/HongW2019/OAP-spark2.4.3/blob/master/webUI.png)
### Use DCPMM to Cache with OAP 
When you want to use DCPMM to cache hot data, firstly you need have DCPMM formatted and mounted on your clusters, and have installed the following requied packages like `numactl numactl-devel memkind `
Then you need create a file named “persistent-memory.xml” under "$SPARK_HOME/conf/" and set the “initialPath” of numa node in “persistent-memory.xml”. You can directly copy the following part only changing `/mnt/pmem0` `/mnt/pmem1` to your path to DCPMM.
```
<persistentMemoryPool>
  <!--The numa id-->
  <numanode id="0">
    <!--The initial path for Intel Optane DC persistent memory-->
    <initialPath>/mnt/pmem0</initialPath>
  </numanode>
  <numanode id="1">
    <initialPath>/mnt/pmem1</initialPath>
  </numanode>
</persistentMemoryPool>
```
#### DCPMM Cache configuration in `$SPARK_HOME/conf/spark-defaults.conf`
```
spark.executor.instances                                  <2X of worker nodes>
spark.yarn.numa.enabled                                    true            # enable numa
spark.executorEnv.MEMKIND_ARENA_NUM_PER_KIND               1
spark.memory.offHeap.enabled                               false
spark.speculation                                          false
spark.sql.oap.fiberCache.memory.manager                    pm              # use DCPMM as cache media
spark.sql.oap.fiberCache.persistent.memory.initial.size                    # ~total available DCPMM per executor
spark.sql.oap.fiberCache.persistent.memory.reserved.size                   # the left DCPMM per executor
spark.sql.oap.parquet.data.cache.enable                    true            # for parquet fileformat
spark.sql.oap.orc.data.cache.enable                        true            # for orc fileformat
```
![Spark configuration with DCPMM cache](https://github.com/HongW2019/OAP-spark2.4.3/blob/master/spark-conf.png)
You can also run Spark with the same following example to try OAP cache function with DCPMM, then you can find the cache metric with OAP TAB in the spark history Web UI.
```
. $SPARK_HOME/bin/spark-shell
> spark.sql(s"""CREATE TEMPORARY TABLE oap_test (a INT, b STRING)
      | USING parquet)
      | OPTIONS (path 'hdfs:///<oap-data-dir>')""".stripMargin)
> val data = (1 to 30000).map { i => (i, s"this is test $i") }.toDF().createOrReplaceTempView("t")
> spark.sql("insert overwrite table oap_test select * from t")
> spark.sql("SELECT * FROM oap_test WHERE a = 1").show()
```
