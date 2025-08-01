# Part 3: Install Hive and TEZ on the 3-NameNode Hadoop Cluster

<br/><br/>
<p align="center">
<picture>
  <img alt="docker" src="https://github.com/kavindatk/hive_tez_spakr_cluster_3NN/blob/main/images/hive.JPG" width="250" height="150">
</picture>
  
<picture>
  <img alt="docker" src="https://github.com/kavindatk/hive_tez_spakr_cluster_3NN/blob/main/images/tez.jpg" width="250" height="150">
</picture>
</p>

<br/><br/>

In the previous setup, we successfully installed and tested a <b>3-NameNode Hadoop environment</b>.
In this article, I’ll explain how to set up a <b>high availability (HA) Hive environment</b> with 3 active-standby HiveServer2 instances running on top of the Hadoop cluster.
For this setup, I’ll be using <b>Hive 4.0.1 and TEZ 0.10.4 </b>, which is the current stable version.
We’ll begin with the Hive installation and configuration, and once that’s complete, we’ll move on to setting up Tez in the next stages.

<br/><br/>

## Step 1: Download, Extract, and Rename

The following steps show how to download the current stable versions of HIVE and TEZ, extract the files, and rename the directories for easier use.

#### Hive

```bash
wget https://archive.apache.org/dist/hive/hive-4.0.1/apache-hive-4.0.1-bin.tar.gz
tar -xvzf apache-hive-4.0.1-bin.tar.gz
mv apache-hive-4.0.1-bin hive
```

#### TEZ

```bash
wget https://dlcdn.apache.org/tez/0.10.4/apache-tez-0.10.4-bin.tar.gz
tar -xvzf apache-tez-0.10.4-bin.tar.gz
mv apache-tez-0.10.4-bin tez
```

<br/>

# Hive
<br/>
## Step 2: Set Environment Variables and Configure Hive

We have already downloaded and renamed the Hive folder as needed. In this step, we’ll set the required environment variables and configure Hive.

The steps below show how to complete this process.

```xml
    Note: These steps should be performed on all 5 servers (3 NameNodes and 2 DataNodes).
    Although HiveServer2 will run on the master nodes, we want the ability to connect to Hive from any node,
    including the DataNodes.To make this possible, we need to copy the Hive setup to all servers
    and apply the same configurations everywhere.
```
<br/>

#### 2.1 Set Environment Variables

```bash
nano nano ~/.bashrc
```

```bash
#Hive Related Options
export HIVE_HOME=/opt/hive
export HIVE_CONF=$HIVE_HOME/conf
export PATH=$PATH:$HIVE_HOME/bin
export CLASSPATH=$CLASSPATH:$HADOOP_HOME/lib/*:.
export CLASSPATH=$CLASSPATH:$HIVE_HOME/lib/*:.
```

```bash
source ~/.bashrc
```

<br/>

#### 2.2 Configure Hive

##### hive-env.sh 

```bash
cd hive/conf/
cp hive-env.sh.template hive-env.sh
nano hive-env.sh
```

Then add or modify

```bash
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
export HADOOP_HOME=/opt/hadoop
export HIVE_CONF_DIR=/opt/hive/conf
```

##### hive-site.xml

```bash
cp hive-default.xml.template hive-site.xml
```

Then add or modify

```xml
<configuration>
	<property>
		<name>javax.jdo.option.ConnectionURL</name>
		<value>jdbc:mysql://bigdataproxy:3307/metastore?createDatabaseIfNotExist=true</value>
		<description>Metastore connection URL using HAProxy</description>
	</property>
	<property>
		<name>javax.jdo.option.ConnectionDriverName</name>
		<value>com.mysql.cj.jdbc.Driver</value>
		<!--value>org.mariadb.jdbc.Driver</value-->
	</property>
	<property>
		<name>javax.jdo.option.ConnectionUserName</name>
		<value>hiveuser</value>
	</property>
	<property>
		<name>javax.jdo.option.ConnectionPassword</name>
		<value>hiveuser</value>
	</property>
	<property>
		<name>hive.server2.thrift.bind.host</name>
		<value>mst01</value>
	</property>
	<property>
		<name>hive.metastore.warehouse.dir</name>
		<value>hdfs:///user/hive/warehouse</value>
	</property>
	<property>
		<name>hive.exec.scratchdir</name>
		<value>hdfs:///tmp/hive</value>
	</property>
	<property>
		<name>hive.repl.rootdir</name>
		<value>hdfs:///usr/hive/repl</value>
	</property>
	<property>
		<name>hive.exec.local.scratchdir</name>
		<value>/tmp/${user.name}</value>
	</property>
	<property>
		<name>hive.downloaded.resources.dir</name>
		<value>/tmp/${user.name}_resources</value>
	</property>
	<property>
		<name>hive.metastore.db.type</name>
		<value>mysql</value>
	</property>
	<property>
		<name>hive.server2.support.dynamic.service.discovery</name>
		<value>true</value>
	</property>
	<property>
		<name>hive.server2.zookeeper.namespace</name>
		<value>hiveserver2</value>
	</property>
	<property>
		<name>hive.server2.transport.mode</name>
		<value>binary</value>
	</property>
	<property>
		<name>hive.zookeeper.quorum</name>
		<value>mst01:2181,mst02:2181,mst03:2181</value>
	</property>
	<property>
		<name>hive.server2.enable.doAs</name>
		<value>true</value>
	</property>
	<property>
		<name>hive.execution.engine</name>
		<value>mr</value>
	</property>
	<property>
		<name>hive.vectorized.execution.enabled</name>
		<value>true</value>
	</property>
	<property>
		<name>hive.vectorized.execution.reduce.enabled</name>
		<value>true</value>
	</property>
	<property>
		<name>hive.tez.container.size</name>
		<value>8192</value>
	</property>
	<property>
		<name>hive.tez.java.opts</name>
		<value>-Xmx6144m</value>
	</property>
	<property>
		<name>hive.metastore.uris</name>
		<value>thrift://mst01:9083,thrift://mst02:9083,thrift://mst03:9083</value>
	</property>
	<property>
		<name>hive.server2.thrift.port</name>
		<value>10000</value>
	</property>
	<property>
		<name>hive.server2.zookeeper.publish.configs</name>
		<value>true</value>
	</property>
	  <property>
		<name>hive.server2.authentication</name>
		<value>NONE</value>
		<description>
		  Expects one of [nosasl, none, ldap, kerberos, pam, custom, saml, jwt].
		  Client authentication types.
			NONE: no authentication check
			LDAP: LDAP/AD based authentication
			KERBEROS: Kerberos/GSSAPI authentication
			CUSTOM: Custom authentication provider
					(Use with property hive.server2.custom.authentication.class)
			PAM: Pluggable authentication module
			NOSASL:  Raw transport
			SAML: SAML 2.0 compliant authentication. This is only supported in http transport mode.
			JWT: JWT based authentication. HS2 expects JWT contains the user name as subject and was signed by an
				 asymmetric key. This is only supported in http transport mode.
		</description>
	  </property>

<property>
  <name>hive.metastore.transactional.event.listeners</name>
  <value>org.apache.hive.hcatalog.listener.DbNotificationListener</value>
</property>

<property>
  <name>hive.metastore.dml.events</name>
  <value>true</value>
</property>
<property>
  <name>hive.create.as.external.legacy</name>
  <value>false</value>
</property>	     

<property>
  <name>hive.default.table.type</name>
  <value>managed</value>
</property>

<property>
  <name>hive.create.as.insert.only</name>
  <value>true</value>
</property>

<property>
  <name>hive.support.concurrency</name>
  <value>true</value>
  <description>Whether Hive supports concurrency or not. A ZooKeeper instance must be up and running for the default Hive lock manager to support read-write locks.</description>
</property>

<property>
  <name>hive.txn.manager</name>
  <value>org.apache.hadoop.hive.ql.lockmgr.DbTxnManager</value>
  <description>Set this to org.apache.hadoop.hive.ql.lockmgr.DbTxnManager to enable ACID transactions.</description>
</property>

<property>
  <name>hive.enforce.bucketing</name>
  <value>true</value>
  <description>Whether bucketing is enforced.</description>
</property>

<property>
  <name>hive.exec.dynamic.partition.mode</name>
  <value>nonstrict</value>
  <description>In strict mode, the user must specify at least one static partition in case the user accidentally overwrites all partitions. In nonstrict mode all partitions are allowed to be dynamic.</description>
</property>

<property>
  <name>hive.compactor.initiator.on</name>
  <value>true</value>
  <description>Whether to run the initiator and cleaner threads on this metastore instance. Set this to true on one instance of the Thrift metastore service as part of turning on Hive transactions.</description>
</property>
<property>
  <name>metastore.client.capability.check</name>
  <value>false</value> 
</property>

<property>
  <name>hive.compactor.worker.threads</name>
  <value>1</value> 
</property>


<!--
   TEZ configurations
-->     
	<property>
	  <name>hive.execution.engine</name>
	  <value>tez</value>
	</property>
	<property>
	  <name>tez.lib.uris</name>
	  <value>hdfs:///apps/tez/share/tez.tar.gz</value>
	</property>
	<property>
	  <name>hive.tez.container.size</name>
	  <value>4096</value>
	</property>
	<property>
	  <name>hive.tez.java.opts</name>
	  <value>-Xmx3072m</value>
	</property>
	<property>
	  <name>hive.tez.log.level</name>
	  <value>INFO</value>
	</property>
```

```bash
cd hive/lib/
rm -rf guava-22.0.jar
rm -rf log4j-slf4j-impl-2.18.0.jar
cp $HADOOP_HOME/share/hadoop/common/lib/guava-27.0-jre.jar /opt/hive/lib/
```

Build metastore

```bash
cd /opt/hive/bin/
schematool -dbType mysql -initSchema
```

Once it done, it will show following message 

```bash
Initialization script completed
```

Now we create required HDFS folders

```bash
hdfs dfs -ls /
hdfs dfs -rm -r /user
hdfs dfs -ls /
hdfs dfs -mkdir -p /user/hive/warehouse

hdfs dfs -chmod g+w /user
hdfs dfs -chmod g+wx /user

hdfs dfs -chmod g+w /tmp
hdfs dfs -chmod g+wx /tmp

hdfs dfs -chown -R hive:hive /user/hive/warehouse
hdfs dfs -chmod -R 770 /user/hive/warehouse
```

## Step 3: Start Hive Services (metastore and Hiveserver2)

For All 3 NN

```bash
hdfs dfs -chmod 777 /user/hive/warehouse
nohup hive --service metastore > hive-metastore.log 2>&1 &
nohup hive --service hiveserver2 > hive-server2.log 2>&1 &
```

## Step 4: Test and Verification

```bash
beeline -u "jdbc:hive2://mst01:2181,mst02:2181,mst03:2181/;serviceDiscoveryMode=zooKeeper;zooKeeperNamespace=hiveserver2"
```

<picture>
  <img alt="docker" src="https://github.com/kavindatk/hive_tez_spakr_cluster_3NN/blob/main/images/hive_log.JPG" width="800" height="400">
</picture>


```bash
#Zookeeper verification

zkCli.sh -server mst01:2181

#Then execute

[zk: mst01:2181(CONNECTED) 0] ls /
[hadoop-ha, hiveserver2, killQueries, yarn-leader-election, zookeeper]
[zk: mst01:2181(CONNECTED) 1]
[zk: mst01:2181(CONNECTED) 1] ls /hiveserver2
[serverUri=mst01:10000;version=4.0.0;sequence=0000000005, serverUri=mst02:10000;version=4.0.0;sequence=0000000006, serverUri=mst03:10000;version=4.0.0;sequence=0000000004]
[zk: mst01:2181(CONNECTED) 2]quit
```

## Useful Commands

```bash
# 1. Stop HiveServer2
pkill -f hiveserver2

# 2. Stop Metastore
pkill -f metastore
```

## Hive server web

http://name node ip:10002/



# TEZ
<br/>

## Step 2: Set Environment Variables and Configure TEZ


We have already downloaded and renamed the TEZ folder as needed. In this step, we’ll set the required environment variables and configure TEZ.

The steps below show how to complete this process.

<br/>

#### 2.1 Set Environment Variables

```bash
nano nano ~/.bashrc
```

```bash
#TEZ Related Options
export TEZ_HOME=/opt/tez
export TEZ_CONF_DIR=$TEZ_HOME/conf

export TEZ_JARS=$TEZ_HOME
# For enabling hive to use the Tez engine
if [ -z "$HIVE_AUX_JARS_PATH" ]; then
export HIVE_AUX_JARS_PATH="$TEZ_JARS"
else
export HIVE_AUX_JARS_PATH="$HIVE_AUX_JARS_PATH:$TEZ_JARS"
fi

export HADOOP_CLASSPATH=${TEZ_CONF_DIR}:${TEZ_JARS}/*:${TEZ_JARS}/lib/*
```

```bash
source ~/.bashrc
```

<br/>

#### 2.2 Configure TEZ

##### Create HDFS Folders

```bash
sudo cp $HIVE_HOME/lib/protobuf-java-3.24.4.jar /opt/tez/lib/
rm -rf /opt/tez/lib/slf4j-reload4j-1.7.36.jar
```

```bash
hdfs dfs -mkdir /apps
hdfs dfs -mkdir /apps/tez
hdfs dfs -chmod g+w /apps
hdfs dfs -chmod g+wx /apps
hdfs dfs -put /opt/tez/* /apps/tez/
hdfs dfs -put $HIVE_HOME/lib/hive-exec-4.0.0.jar /apps/tez
```

##### tez-site.xml

```bash
nano /opt/tez/conf/tez-site.xml
```

Then add or modify

```xml

<configuration>
  <property>
    <name>tez.lib.uris</name>
    <value>hdfs:///apps/tez/share/tez.tar.gz</value>
  </property>
  <property>
    <name>tez.use.cluster.hadoop-libs</name>
    <value>true</value>
  </property>
</configuration>
```

##### hive-site.xml

TEZ run on top of HIVE therefore we have to introduce TEZ configurations to HIVE , add or modify following in hive-site.xml (All 3NN)

```xml
<property>
  <name>hive.execution.engine</name>
  <value>tez</value>
</property>
<property>
  <name>tez.lib.uris</name>
  <value>hdfs:///apps/tez/share/tez.tar.gz</value>
</property>
<property>
  <name>hive.tez.container.size</name>
  <value>4096</value>
</property>
<property>
  <name>hive.tez.java.opts</name>
  <value>-Xmx3072m</value>
</property>
<property>
  <name>hive.tez.log.level</name>
  <value>INFO</value>
</property>
```

### Distribute Hive and Tez Configurations to All Nodes

Now that we’ve successfully configured Hive and Tez, the next step is to distribute these configurations across the remaining nodes in the cluster.
Before copying, make sure to check for any node-specific configurations, such as unique paths, hostnames, or roles (like active/standby). Avoid overwriting those parts during distribution.
Once verified, you can safely copy the Hive and Tez folders along with their configuration files to all other nodes.


## Step 3: Test and Verification

```bash
# Create Dummy Data Set
cd /home/hadoop
seq 1 10000000 > numbers.txt
hdfs dfs -mkdir /user/hive/warehouse/sample/
hdfs dfs -put numbers.txt /user/hive/warehouse/sample/
```

```bash
beeline -u "jdbc:hive2://bigdataproxy/;serviceDiscoveryMode=zooKeeper;zooKeeperNamespace=hiveserver2"
```


```bash
0: jdbc:hive2://bigdataproxy/> show databases;
INFO  : Compiling command(queryId=hadoop_20250724081919_b58a82cb-3908-45aa-9bf1-bd379d447b67): show databases
INFO  : Semantic Analysis Completed (retrial = false)
INFO  : Created Hive schema: Schema(fieldSchemas:[FieldSchema(name:database_name, type:string, comment:from deserializer)], properties:null)
INFO  : Completed compiling command(queryId=hadoop_20250724081919_b58a82cb-3908-45aa-9bf1-bd379d447b67); Time taken: 0.005 seconds
INFO  : Concurrency mode is disabled, not creating a lock manager
INFO  : Executing command(queryId=hadoop_20250724081919_b58a82cb-3908-45aa-9bf1-bd379d447b67): show databases
INFO  : Starting task [Stage-0:DDL] in serial mode
INFO  : Completed executing command(queryId=hadoop_20250724081919_b58a82cb-3908-45aa-9bf1-bd379d447b67); Time taken: 0.029 seconds
+----------------+
| database_name  |
+----------------+
| data_load      |
| default        |
+----------------+
2 rows selected (0.101 seconds)
0: jdbc:hive2://bigdataproxy/>
0: jdbc:hive2://bigdataproxy/>
0: jdbc:hive2://bigdataproxy/> use data_load;
INFO  : Compiling command(queryId=hadoop_20250724082529_8fbbd373-7be8-48f5-9f3e-905fe7555ed0): use data_load
INFO  : Semantic Analysis Completed (retrial = false)
INFO  : Created Hive schema: Schema(fieldSchemas:null, properties:null)
INFO  : Completed compiling command(queryId=hadoop_20250724082529_8fbbd373-7be8-48f5-9f3e-905fe7555ed0); Time taken: 0.012 seconds
INFO  : Concurrency mode is disabled, not creating a lock manager
INFO  : Executing command(queryId=hadoop_20250724082529_8fbbd373-7be8-48f5-9f3e-905fe7555ed0): use data_load
INFO  : Starting task [Stage-0:DDL] in serial mode
INFO  : Completed executing command(queryId=hadoop_20250724082529_8fbbd373-7be8-48f5-9f3e-905fe7555ed0); Time taken: 0.025 seconds
No rows affected (0.06 seconds)
0: jdbc:hive2://bigdataproxy/> CREATE  TABLE data_load.number_data (
. . . . . . . . . . . . . . .> val_col bigint
. . . . . . . . . . . . . . .> )row format delimited fields terminated by ','
. . . . . . . . . . . . . . .> location '/user/hive/warehouse/sample/';
INFO  : Compiling command(queryId=hadoop_20250724082550_5e8f74fa-2bf9-42e9-b89e-f89f3d9a42bf): CREATE  TABLE data_load.number_data (
val_col bigint
)row format delimited fields terminated by ','
location '/user/hive/warehouse/sample/'
INFO  : Semantic Analysis Completed (retrial = false)
INFO  : Created Hive schema: Schema(fieldSchemas:null, properties:null)
INFO  : Completed compiling command(queryId=hadoop_20250724082550_5e8f74fa-2bf9-42e9-b89e-f89f3d9a42bf); Time taken: 0.057 seconds
INFO  : Concurrency mode is disabled, not creating a lock manager
INFO  : Executing command(queryId=hadoop_20250724082550_5e8f74fa-2bf9-42e9-b89e-f89f3d9a42bf): CREATE  TABLE data_load.number_data (
val_col bigint
)row format delimited fields terminated by ','
location '/user/hive/warehouse/sample/'
INFO  : Starting task [Stage-0:DDL] in serial mode
INFO  : Completed executing command(queryId=hadoop_20250724082550_5e8f74fa-2bf9-42e9-b89e-f89f3d9a42bf); Time taken: 0.87 seconds
No rows affected (0.944 seconds)
0: jdbc:hive2://bigdataproxy/> show tables;
INFO  : Compiling command(queryId=hadoop_20250724082559_6d421984-4f2f-4258-a6db-2905e8598e16): show tables
INFO  : Semantic Analysis Completed (retrial = false)
INFO  : Created Hive schema: Schema(fieldSchemas:[FieldSchema(name:tab_name, type:string, comment:from deserializer)], properties:null)
INFO  : Completed compiling command(queryId=hadoop_20250724082559_6d421984-4f2f-4258-a6db-2905e8598e16); Time taken: 0.017 seconds
INFO  : Concurrency mode is disabled, not creating a lock manager
INFO  : Executing command(queryId=hadoop_20250724082559_6d421984-4f2f-4258-a6db-2905e8598e16): show tables
INFO  : Starting task [Stage-0:DDL] in serial mode
INFO  : Completed executing command(queryId=hadoop_20250724082559_6d421984-4f2f-4258-a6db-2905e8598e16); Time taken: 0.091 seconds
+--------------+
|   tab_name   |
+--------------+
| number_data  |
| test_01      |
+--------------+
2 rows selected (0.168 seconds)
0: jdbc:hive2://bigdataproxy/> desc number_data;
INFO  : Compiling command(queryId=hadoop_20250724082607_a2c743ae-695b-428d-9632-3e10dc3aaba5): desc number_data
INFO  : Semantic Analysis Completed (retrial = false)
INFO  : Created Hive schema: Schema(fieldSchemas:[FieldSchema(name:col_name, type:string, comment:from deserializer), FieldSchema(name:data_type, type:string, comment:from deserializer), FieldSchema(name:comment, type:string, comment:from deserializer)], properties:null)
INFO  : Completed compiling command(queryId=hadoop_20250724082607_a2c743ae-695b-428d-9632-3e10dc3aaba5); Time taken: 0.094 seconds
INFO  : Concurrency mode is disabled, not creating a lock manager
INFO  : Executing command(queryId=hadoop_20250724082607_a2c743ae-695b-428d-9632-3e10dc3aaba5): desc number_data
INFO  : Starting task [Stage-0:DDL] in serial mode
INFO  : Completed executing command(queryId=hadoop_20250724082607_a2c743ae-695b-428d-9632-3e10dc3aaba5); Time taken: 0.048 seconds
+-----------+------------+----------+
| col_name  | data_type  | comment  |
+-----------+------------+----------+
| val_col   | bigint     |          |
+-----------+------------+----------+
1 row selected (0.18 seconds)
hadoop@mst01:~/HIVE_LOG$ beeline -u "jdbc:hive2://bigdataproxy/;serviceDiscoveryMode=zooKeeper;zooKeeperNamespace=hiveserver2" -n hadoop
Connecting to jdbc:hive2://bigdataproxy/;serviceDiscoveryMode=zooKeeper;zooKeeperNamespace=hiveserver2
25/07/24 08:50:57 [main]: INFO jdbc.ZooKeeperHiveClientHelper: Discovered HiveServer2 hosts in ZooKeeper [/hiveserver2]: [serverUri=mst03:10000;version=4.0.0;sequence=0000000018, serverUri=mst02:10000;version=4.0.0;sequence=0000000020, serverUri=mst01:10000;version=4.0.0;sequence=0000000019]
25/07/24 08:50:57 [main]: INFO jdbc.HiveConnection: Connected to mst03:10000
Connected to: Apache Hive (version 4.0.0)
Driver: Hive JDBC (version 4.0.0)
Transaction isolation: TRANSACTION_REPEATABLE_READ
Beeline version 4.0.0 by Apache Hive
0: jdbc:hive2://bigdataproxy/> select sum(val_col) from data_load.number_data;
INFO  : Compiling command(queryId=hadoop_20250724085100_7c24996a-b32b-43d2-9118-9f248f284d71): select sum(val_col) from data_load.number_data
INFO  : No Stats for data_load@number_data, Columns: val_col
INFO  : Semantic Analysis Completed (retrial = false)
INFO  : Created Hive schema: Schema(fieldSchemas:[FieldSchema(name:_c0, type:bigint, comment:null)], properties:null)
INFO  : Completed compiling command(queryId=hadoop_20250724085100_7c24996a-b32b-43d2-9118-9f248f284d71); Time taken: 4.232 seconds
INFO  : Concurrency mode is disabled, not creating a lock manager
INFO  : Executing command(queryId=hadoop_20250724085100_7c24996a-b32b-43d2-9118-9f248f284d71): select sum(val_col) from data_load.number_data
INFO  : Query ID = hadoop_20250724085100_7c24996a-b32b-43d2-9118-9f248f284d71
INFO  : Total jobs = 1
INFO  : Launching Job 1 out of 1
INFO  : Starting task [Stage-1:MAPRED] in serial mode
INFO  : Subscribed to counters: [] for queryId: hadoop_20250724085100_7c24996a-b32b-43d2-9118-9f248f284d71
INFO  : Tez session hasn't been created yet. Opening session
----------------------------------------------------------------------------------------------
        VERTICES      MODE        STATUS  TOTAL  COMPLETED  RUNNING  PENDING  FAILED  KILLED
----------------------------------------------------------------------------------------------
Map 1            container       RUNNING      2          0        2        0       0       0
Reducer 2        container        INITED      1          0        0        1       0       0
----------------------------------------------------------------------------------------------
        VERTICES      MODE        STATUS  TOTAL  COMPLETED  RUNNING  PENDING  FAILED  KILLED
----------------------------------------------------------------------------------------------
INFO  : Completed executing command(queryId=hadoop_20250724085100_7c24996a-b32b-43d2-9118-9f248f284d71); Time taken: 135.676 seconds
Map 1 .......... container     SUCCEEDED      2          2        0        0       0       0
Reducer 2 ...... container     SUCCEEDED      1          1        0        0       0       0
----------------------------------------------------------------------------------------------
VERTICES: 02/02  [==========================>>] 100%  ELAPSED TIME: 16.49 s
----------------------------------------------------------------------------------------------
+-----------------+
|       _c0       |
+-----------------+
| 50000005000000  |
+-----------------+
1 row selected (140.468 seconds)
0: jdbc:hive2://bigdataproxy/>

```

<picture>
  <img alt="docker" src="https://github.com/kavindatk/hive_tez_cluster_3NN/blob/main/images/tez_log.JPG" width="800" height="400">
</picture>


## Useful Commands

```bash
#If you got any error on TEZ running (Version Mismatches)
cd /opt/hive/lib/
mv tez-api-0.10.3.jar tez-api-0.10.3.jar_backup
cp /opt/tez/tez-api-0.10.4.jar /opt/hive/lib/
```
