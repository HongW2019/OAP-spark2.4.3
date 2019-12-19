# Getting Started with OAP
[![Build Status](https://travis-ci.org/Intel-bigdata/OAP.svg?branch=master)](https://travis-ci.org/Intel-bigdata/OAP)

If you want to get started with OAP quickly, we also provide pre-build [Spark-2.3.2]() and [OAP]() to you, then you can directly skip to [Configuration](#Configuration).
## Prerequisites
We suppose you have set Hadoop clusters which have Yarn, Hive and Spark, and this clusters have configured appropriately according to Apache Hadoop guidance [docs]( https://hadoop.apache.org/docs/stable/index.html).  We recommend you install [Spark-2.3.2]( https://github.com/apache/spark/tree/v2.3.2) and refer to [guidance](https://github.com/apache/spark) for building Spark details.
## Dependencies
You will need to install required packages on the build system:
*	autoconf
*	automake
*	gcc-c++
*	libnuma-devel
*	libtool
*	numactl-devel
*	numactl
*	memkind
## Building

```
git clone https://github.com/Intel-bigdata/OAP.git
cd OAP & git checkout -b branch-0.6-spark-2.3.2 origin/branch-0.6-spark-2.3.2
mvn clean -q -Ppersistent-memory -DskipTests package
```
Profile `persistent-memory` is Optional.
You can find the “oap-0.6-with-spark-2.3.2.jar”  in “./target/”.
When you compiled with “persistent-memory”, you need create the “persistent-memory.xml” file in conf directory by command
`cd conf & cp persistent-memory.xml.template  ./persistent-memory.xml`
Then you need set the “initialPath” of numa node in “persistent-memory.xml”.

## Configuration
When Spark runs on clusters, we list corresponding OAP configurations in “$SPARK_HOME/conf/spark-defaults.conf” to different deployment modes of Spark.
### Spark on Yarn with Client Mode
```
spark.master                     yarn
spark.deploy-mode                client
spark.sql.extensions             org.apache.spark.sql.OapExtensions
spark.files                     /<PATH_TO_OAP_JAR>/oap-0.6-with-spark-2.3.2.jar  # absolute path  
spark.executor.extraClassPath                ./oap-0.6-with-spark-2.3.2.jar      # relative path
spark.driver.extraClassPath     /<PATH_TO_OAP_JAR>/oap-0.6-with-spark-2.3.2.jar  # absolute path
```
### Spark on Yarn with Cluster Mode

```
spark.master                      yarn
spark.deploy-mode                 cluster
spark.sql.extensions              org.apache.spark.sql.OapExtensions
spark.files                       /<PATH_TO_OAP_JAR>/oap-0.6-with-spark-2.3.2.jar     # absolute path    
spark.executor.extraClassPath     ./ oap-0.6-with-spark-2.3.2.jar                     # relative path 
spark.driver.extraClassPath       ./oap-0.6-with-spark-2.3.2.jar                      # relative path

```
### Standalone and Spark on K8S Mode
```
spark.sql.extensions               org.apache.spark.sql.OapExtensions
spark.executor.extraClassPath      /<PATH_TO_OAP_JAR>/oap-0.6-with-spark-2.3.2.jar  # absolute path
spark.driver.extraClassPath        /<PATH_TO_OAP_JAR>/oap-0.6-with-spark-2.3.2.jar  # absolute path
```

**NOTE**:Note: For spark standalone mode, you have to put oap-0.6.0-with-2.3.2.jar to both driver and executors since spark.files is not working, also don't forget to update corresponding extraClassPath. 

In the following part, we will take Spark on Yarn with Client Mode for example to introduce you more configurations to correctly deploy Spark with OAP.


### For yarn
With Yarn, you need to set the following properties to ensure all the available resources (CPU cores, memory) can be fully utilized and not be exceeded by the Spark executors with OAP.
```
yarn.nodemanager.vmem-pmem-ratio
yarn.nodemanager.resource.memory-mb                   # total available memory size of each worker
yarn.scheduler.maximum-allocation-mb                  # no more than yarn.nodemanager.resource.memory-mb
yarn.scheduler.minimum-allocation-mb                  # less than yarn.scheduler.maximum-allocation-mb
yarn.nodemanager.resource.cpu-vcores                  # total available CPU cores of each worker
yarn.scheduler.maximum-allocation-vcores              # no more than yarn.nodemanager.resource.cpu-vcores
yarn.scheduler.minimum-allocation-vcores              # less than yarn.scheduler.maximum-allocation-vcore
```
You need to ensure that the above properties are consistent among the master and all the workers, so we recommend you copy hdfs-site.xml, mapred-site.xml, yarn-site.xml of master to your workers to keep consistent among nodes. If failed to launch Spark with OAP, you need to check the logs to find the reason.
### For spark
Next you also need add the following configurations to $SPARK_HOME/conf/spark-defaults.conf. 
```
spark.driver.memory
spark.executor.cores                              # less than yarn.scheduler.maximum-allocation-vcores
spark.executor.memory                             # less than yarn.scheduler.maximum-allocation-mb                              
spark.yarn.executor.memoryOverhead                # close to spark.memory.offHeap.size
spark.executor.instances                                     
spark.memory.offHeap.enabled                    
spark.memory.offHeap.size
```
Executor instances can be 1~2X of worker nodes. Considering the count of executor instances (N) on each node, executor memory can be around 1/N of each worker total available memory. Usually each worker has one or two executor instances. However, considering the cache utilization, one executor per work node is recommended. Always enable offHeap memory and set a reasonable (the larger the better) size, as long as OAP's fine-grained cache takes advantage of offHeap memory, otherwise user might encounter weird circumstances.
After deployment and configuration, you can run by` bin/spark-sql, bin/spark-shell, bin/spark-submit --deploy-mode client, sbin/start-thriftserver or bin/pyspark. `

## How to Use OAP
### How to Use Index with OAP on Spark
You can run Spark with the following example to try OAP index function.
```
. $SPARK_HOME/bin/spark-shell
> spark.sql(s"""CREATE TEMPORARY TABLE oap_test (a INT, b STRING)
      | USING oap)
      | OPTIONS (path 'hdfs:///<oap-data-dir>')""".stripMargin)
> val data = (1 to 300).map { i => (i, s"this is test $i") }.toDF().createOrReplaceTempView("t")
> spark.sql("insert overwrite table oap_test select * from t")
> spark.sql("create oindex index1 on oap_test (a)")
> spark.sql("show oindex from oap_test").show()
> spark.sql("SELECT * FROM oap_test WHERE a = 1").show()
> spark.sql("drop oindex index1 on oap_test")
```
For  more detailed examples on OAP performance comparation, you can refer to this page for further instructions.

### How to Use Cache with OAP on Spark
If you want to run OAP with cache function, firstly you should add some configurations into `$SPARK_HOME/conf/spark-defaults.conf`. OAP provides two medias to cache the hot data: DRAM and DCPMM.

#### DRAM Cache Configuration in ` $SPARK_HOME/conf/spark-defaults.conf `
```
spark.memory.offHeap.enabled                true
spark.memory.offHeap.size                  <set a suitable size>
spark.sql.oap.parquet.data.cache.enable     true     #for parquet fileformat
spark.sql.oap.orc.data.cache.enable         true     #for orc fileformat
```
You can run Spark with the follow example to try OAP cache function, then you can find the cache metric with OAP TAB in the spark history Web UI.
```
. $SPARK_HOME/bin/spark-shell
> spark.sql(s"""CREATE TEMPORARY TABLE oap_test (a INT, b STRING)
      | USING parquet)
      | OPTIONS (path 'hdfs:///<oap-data-dir>')""".stripMargin)
> val data = (1 to 3000).map { i => (i, s"this is test $i") }.toDF().createOrReplaceTempView("t")
> spark.sql("insert overwrite table oap_test select * from t")
> spark.sql("SELECT * FROM oap_test WHERE a = 1").show()
```
When you want to use DCPMM to cache hot data, you should change and add following configurations.
#### DCPMM Cache configuration in `$SPARK_HOME/conf/spark-defaults.conf`
```
spark.executor.instances                                  <2X of worker nodes>
spark.yarn.numa.enabled                                    true            # enable numa
spark.executorEnv.MEMKIND_ARENA_NUM_PER_KIND               1
spark.memory.offHeap.enabled                               false
spark.speculation                                          false
spark.sql.oap.fiberCache.memory.manager                    pm              # use DCPMM as cache media
spark.sql.oap.fiberCache.persistent.memory.initial.size                    # total available DCPMM  per executor
spark.sql.oap.fiberCache.persistent.memory.reserved.size                   # the left DCPMM per executor
spark.sql.oap.parquet.data.cache.enable                    true            # for parquet fileformat
spark.sql.oap.orc.data.cache.enable                        true            # for orc fileformat
```
