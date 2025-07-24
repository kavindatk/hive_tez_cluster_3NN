# Part 3: Install Hive, Tez, and Spark on the 3-NameNode Hadoop Cluster

<br/><br/>
<p align="center">
<picture>
  <img alt="docker" src="https://github.com/kavindatk/hive_tez_spakr_cluster_3NN/blob/main/images/hive.JPG" width="250" height="150">
</picture>
  
<picture>
  <img alt="docker" src="https://github.com/kavindatk/hive_tez_spakr_cluster_3NN/blob/main/images/tez.jpg" width="250" height="150">
</picture>


<picture>
  <img alt="docker" src="https://github.com/kavindatk/hive_tez_spakr_cluster_3NN/blob/main/images/spark.JPG" width="250" height="150">
</picture>
</p>

</p>

<br/><br/>

In the previous setup, we successfully installed and tested a <b>3-NameNode Hadoop environment</b>.
In this article, I’ll explain how to set up a <b>high availability (HA) Hive environment</b> with 3 active-standby HiveServer2 instances running on top of the Hadoop cluster.
For this setup, I’ll be using <b>Hive 4.0.1</b>, which is the current stable version.
We’ll begin with the Hive installation and configuration, and once that’s complete, we’ll move on to setting up Tez and Spark in the next stages.

<br/><br/>

## Step 1: Download, Extract, and Rename

The following steps show how to download the current stable versions of HIVE, TEZ and SPARK, extract the files, and rename the directories for easier use.

#### Hive

```bash
wget https://archive.apache.org/dist/hive/hive-4.0.1/apache-hive-4.0.1-bin.tar.gz
tar -xvzf apache-hive-4.0.1-bin.tar.gz
mv apache-hive-4.0.1-bin hive
```

<br/><

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
</configuration>
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


