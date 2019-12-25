# OAP Usage Guide
[![Build Status](https://travis-ci.org/Intel-bigdata/OAP.svg?branch=master)](https://travis-ci.org/Intel-bigdata/OAP)

* [OAP Architecture Overview](#OAP_Architecture_Overview)
* [Prerequisites](#Prerequisites)
* [Getting Started with OAP](#Getting_Started_with_OAP)
* [Cluster and Standalone Mode](#How_to_Use_OAP)
* [Working with OAP Index](#Working_with_OAP_Index)
* [Working with OAP Cache](#Working_with_OAP_Cache)
* [OAP Developer Guide](#OAP_Developer_Guide)

## Prerequisites
Before getting started with OAP on Spark, you should have set up a working Hadoop cluster with YARN and Spark. Running Spark on YARN requires a binary distribution of Spark which is built with YARN support. If you don't want to build Spark by yourself, we have a pre-built Spark-2.3.2, you can download Spark-2.3.2 from this [page]() to your master machine and unzip it to a given path. 
## Getting Started with OAP
### Building OAP
We have a pre-built OAP, you can download it from this [page]() to your master machine and unzip it to a given path such as “/opt/oap/jars/”. If you’d like to build OAP from source code, more details can be found at [OAP developer guide]().
### Configurations on Spark with OAP
A common deploy mode that can be used to launch Spark applications on YRAN is `client` mode. The `client` mode is especially suitable for applications such as Spark shell. Before we run ` . $SPARK_HOME/bin/spark-shell ` to launch Spark, firstly you should add OAP configurations in the file of `$SPARK_HOME/conf/spark-defaults.conf`

```
spark.master                      yarn
spark.deploy-mode                 client
spark.sql.extensions              org.apache.spark.sql.OapExtensions
spark.files                       /opt/oap/jars/oap-0.6-with-spark-2.3.2.jar          # absolute path of OAP jar  
spark.executor.extraClassPath     ./oap-0.6-with-spark-2.3.2.jar                      # relative path of OAP jar
spark.driver.extraClassPath       /opt/oap/jars/oap-0.6-with-spark-2.3.2.jar          # absolute path of OAP jar
```
### Run Spark with OAP 
After deployment and configuration, you can follow the steps to run Spark shell and check if OAP configurations work. 

```
. $SPARK_HOME/bin/spark-shell
> spark.sql(s"""CREATE TABLE oap_test (a INT, b STRING)
       USING parquet
       OPTIONS (path 'hdfs:///user/oap/')""".stripMargin)
> val data = (1 to 30000).map { i => (i, s"this is test $i") }.toDF().createOrReplaceTempView("t")
> spark.sql("insert overwrite table oap_test select * from t")
> spark.sql("create oindex index1 on oap_test (a)")
> spark.sql("show oindex from oap_test").show()
```
when your Spark shell shows the same as below picture, it means you have start Spark with OAP successfully.
![Spark_shell_running_results](https://github.com/HongW2019/OAP-spark2.4.3/blob/master/docs/image/spark_shell_oap.png)

## YARN Cluster and Spark Standalone Mode
### YARN Cluster mode
There are two deploy modes that can be used to launch Spark applications on YARN, `client` and `cluster` mode. if your application is submitted from a machine far from the worker machines (e.g. locally on your laptop), it is common to use `cluster` mode to minimize network latency between the drivers and the executors. Launching Applications with spark-submit can support different deploy modes that Spark supports, so you can run spark-submit to use YARN cluster mode.
#### Configurations on Spark with OAP on YARN Cluster Mode
Before run spark-submit, you should add OAP configurations in the file of `$SPARK_HOME/conf/spark-defaults.conf`
```
spark.master                      yarn
spark.deploy-mode                 cluster
spark.sql.extensions              org.apache.spark.sql.OapExtensions
spark.files                       /opt/oap/jars/oap-0.6-with-spark-2.3.2.jar          # absolute path    
spark.executor.extraClassPath     ./oap-0.6-with-spark-2.3.2.jar                      # relative path 
spark.driver.extraClassPath       ./oap-0.6-with-spark-2.3.2.jar                      # relative path
```
then you can run spark-submit with YARN cluster mode
```
.$SPARK_HOME/bin/spark-submit \
  --master yarn \
  --deploy-mode cluster \
  ...
```
### Spark Standalone mode
In addition to running on the YARN cluster managers, Spark also provides a simple standalone deploy mode. If install `Spark Standalone mode`, you simply place a compiled version of Spark and OAP on each node on the cluster.
```
spark.sql.extensions               org.apache.spark.sql.OapExtensions
spark.executor.extraClassPath      /opt/oap/jars/oap-0.6-with-spark-2.3.2.jar    # absolute path
spark.driver.extraClassPath        /opt/oap/jars/oap-0.6-with-spark-2.3.2.jar    # absolute path
```

## Working with OAP Index
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

## Working with OAP Cache

If you want to run OAP with cache function, there are two media types in OAP to cache hot data: DRAM and DCPMM. To better
firstly you should change some configurations into `$SPARK_HOME/conf/spark-defaults.conf`. 
### Use DRAM to Cache with OAP


#### DRAM Cache Configuration in ` $SPARK_HOME/conf/spark-defaults.conf `
```
spark.memory.offHeap.enabled                true
spark.memory.offHeap.size                  <set a suitable size>
spark.sql.oap.parquet.data.cache.enable     true     #for parquet fileformat
spark.sql.oap.orc.data.cache.enable         true     #for orc fileformat
```
You can run Spark with the following example to try OAP cache function with DRAM. We recommand you use Thrift server
The Thrift JDBC/ODBC server implemented here corresponds to the HiveServer2 in Hive 1.2.1. You can test the JDBC server with the beeline script that comes with Spark.
In the Working with OAP Index, we have create a table oap_test, next we will try OAP Cache.
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
> SHOW databases;
> USE default;
> SHOW tables;
> USE oap_test;
```
next you can run query like
```
> SELECT * FROM oap_test WHERE a = 1;
> SELECT * FROM oap_test WHERE a = 2;
> SELECT * FROM oap_test WHERE a = 3;
...
```
Then you can find the cache metric with OAP TAB in the spark history Web UI.

![webUI](https://github.com/HongW2019/OAP-spark2.4.3/blob/master/webUI.png)



### Use DCPMM to Cache with OAP 
When you want to use DCPMM to cache hot data, you should follow the below steps.
Step 1. You need have DCPMM formatted and mounted on your clusters.
Step 2. Make libmemkind.so.0, libnuma.so.1 be accessed in each executor node. (Centos: /lib64/)
Step 3. Install numactl by `yum install numactl -y `
Step 4. Create a file named “persistent-memory.xml” under "$SPARK_HOME/conf/" and set the “initialPath” of numa node in “persistent-memory.xml”. You can directly copy the following part only changing `/mnt/pmem0` `/mnt/pmem1` to your path to DCPMM.
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
spark.executor.instances                                   4               # 2x of number of your worker nodes
spark.yarn.numa.enabled                                    true            # enable numa
spark.executorEnv.MEMKIND_ARENA_NUM_PER_KIND               1
spark.memory.offHeap.enabled                               false
spark.speculation                                          false
spark.sql.oap.fiberCache.memory.manager                    pm              # use DCPMM as cache media
spark.sql.oap.fiberCache.persistent.memory.initial.size    450g            # ~90% of total available DCPMM per executor
spark.sql.oap.fiberCache.persistent.memory.reserved.size   30g             # the left DCPMM per executor
spark.sql.oap.parquet.data.cache.enable                    true            # for parquet fileformat
spark.sql.oap.orc.data.cache.enable                        true            # for orc fileformat
```
Here we privide you with an example, this cluster consists of 2 worker nodes, per node has 2 pieces of 488GB DCPMM ; 96 Vcores, 360GB Memory.  
![Spark configuration with DCPMM cache](https://github.com/HongW2019/OAP-spark2.4.3/blob/master/spark-conf.png)

You can also run Spark with the same following example to try OAP cache function with DCPMM, then you can find the cache metric with OAP TAB in the spark history Web UI.
```
cd $SPARK_HOME/bin/
. $SPARK_HOME/sbin/start-thriftserver.sh
./beeline -u jdbc:hive2://vsr211:10000
> SHOW databases;
> USE default;
> SHOW tables;
> USE oap_test;
> SELECT * FROM oap_test WHERE a = 1;
> SELECT * FROM oap_test WHERE a = 2;
> SELECT * FROM oap_test WHERE a = 3;
...
```
