# Beeling client accessing Thrift Server in Docker

Apache Spark is an open-source, distributed processing system used for big data workloads.

In this demo, a Spark container uses a Hadoop YARN cluster as a resource management and job scheduling technology to perform distributed data processing.

This Docker image contains Spark binaries prebuilt and uploaded in Docker Hub.

Spark client access a remote Hive metastore created in a MySQL Server for Spark catalog.


## Start Swarm cluster

1. start swarm mode in node1
```shell
$ docker swarm init --advertise-addr <IP node1>
$ docker swarm join-token worker  # issue a token to add a node as worker to swarm
```

2. add 3 more workers in swarm cluster (node2, node3, node4)
```shell
$ docker swarm join --token <token> <IP nodeN>:2377
```

3. label each node to anchor each container in swarm cluster
```shell
docker node update --label-add hostlabel=hdpmst node1
docker node update --label-add hostlabel=hdp1 node2
docker node update --label-add hostlabel=hdp2 node3
docker node update --label-add hostlabel=hdp3 node4
```

4. create an external "overlay" network in swarm to link the 2 stacks (hdp and spk)
```shell
docker network create --driver overlay mynet
```

5. start spark master, client and mysql server
```shell
$ docker stack deploy -c docker-compose.yml spk
$ docker service ls
ID             NAME          MODE         REPLICAS   IMAGE                              PORTS
iedh5i4psnxj   spk_mysql     replicated   1/1        mysql:5.7                          
ij64wq5yg7yq   spk_spk_cli   replicated   1/1        mkenjis/ubspkcli_yarn_img:latest   
yoxz4c6818w2   spk_spk_mst   replicated   1/1        mkenjis/ubspkcli_yarn_img:latest
```

## Set up MySQL server

In mysql 5.7
1. download hive binaries and unpack it
```shell
wget https://archive.apache.org/dist/hive/hive-1.2.1/apache-hive-1.2.1-bin.tar.gz
tar -xzf apache-hive-1.2.1-bin.tar.gz
```

2. access mysql server node and run hive script to create metastore tables
```shell
/root/staging/apache-hive-1.2.1-bin/scripts/metastore/upgrade/mysql
mysql -uroot -p metastore < hive-schema-1.2.0.mysql.sql
Enter password:
```

In mysql 8.0
1. download hive binaries and unpack it
```shell
https://archive.apache.org/dist/hive/hive-2.1.0/apache-hive-2.1.0-bin.tar.gz
tar -xzf apache-hive-2.1.0-bin.tar.gz
```

2. access mysql server node and run hive script to create metastore tables
```shell
/root/staging/apache-hive-2.1.0-bin/scripts/metastore/upgrade/mysql
mysql -uroot -p metastore < hive-schema-2.1.0.mysql.sql
Enter password:
```

## Set up Spark master

1. access spark master node
```shell
$ docker container exec -it <spk_cli ID> bash
```

2. copy following file into $SPARK_HOME/conf
```shell
$ cd $SPARK_HOME/conf
$ # create hive-site.xml with settings provided
```

3. start spark-shell installing mysql jar files
```shell
$ spark-shell --packages mysql:mysql-connector-java:5.1.49
2021-12-05 11:09:14 WARN  NativeCodeLoader:62 - Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
Setting default log level to "WARN".
To adjust logging level use sc.setLogLevel(newLevel). For SparkR, use setLogLevel(newLevel).
2021-12-05 11:09:40 WARN  Client:66 - Neither spark.yarn.jars nor spark.yarn.archive is set, falling back to uploading libraries under SPARK_HOME.
Spark context Web UI available at http://802636b4d2b4:4040
Spark context available as 'sc' (master = yarn, app id = application_1638723680963_0001).
Spark session available as 'spark'.
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /___/ .__/\_,_/_/ /_/\_\   version 2.3.2
      /_/
         
Using Scala version 2.11.8 (Java HotSpot(TM) 64-Bit Server VM, Java 1.8.0_181)
Type in expressions to have them evaluated.
Type :help for more information.

scala> 
```

4. create a persistent table in metastore
```shell
scala> val df = spark.read.option("inferSchema","true").csv("housing.data").toDF("CRIM","ZN","INDUS","CHAS","NOX","RM","AGE","DIS","RAD","TAX","PTRATIO","B","LSTAT","MEDV")
scala> df.show
scala> df.write.saveAsTable("housing")
scala> 
```

5. exit the shell and start thriftserver
```shell
start-thriftserver.sh --packages mysql:mysql-connector-java:5.1.49
```

## Set up Spark client

1. access spark client node
```shell
$ docker container exec -it <spk_cli ID> bash
```

2. copy following file into $SPARK_HOME/conf
```shell
$ cd $SPARK_HOME/conf
$ # create hive-site.xml with settings provided
```

3. start beeline
```shell
$ beeline
Beeline version 1.2.1.spark2 by Apache Hive

beeline> !connect jdbc:hive2://97dbb9943552:10000
Connecting to jdbc:hive2://97dbb9943552:10000
Enter username for jdbc:hive2://97dbb9943552:10000: hive
Enter password for jdbc:hive2://97dbb9943552:10000: 
2023-07-17 15:34:45 INFO  Utils:310 - Supplied authorities: 97dbb9943552:10000
2023-07-17 15:34:45 INFO  Utils:397 - Resolved authority: 97dbb9943552:10000
2023-07-17 15:34:45 INFO  HiveConnection:203 - Will try to open client transport with JDBC Uri: jdbc:hive2://97dbb9943552:10000
Connected to: Spark SQL (version 2.3.2)
Driver: Hive JDBC (version 1.2.1.spark2)
Transaction isolation: TRANSACTION_REPEATABLE_READ
0: jdbc:hive2://97dbb9943552:10000> show tables;
+-----------+------------+--------------+--+
| database  | tableName  | isTemporary  |
+-----------+------------+--------------+--+
| default   | housing    | false        |
+-----------+------------+--------------+--+
1 row selected (1.163 seconds)
0: jdbc:hive2://97dbb9943552:10000> select count(*) from housing;
+-----------+--+
| count(1)  |
+-----------+--+
| 506       |
+-----------+--+
1 row selected (4.179 seconds)
0: jdbc:hive2://97dbb9943552:10000> !quit
```

