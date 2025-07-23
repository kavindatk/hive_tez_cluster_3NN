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
  <property>
    <name>javax.jdo.option.ConnectionURL</name>
    <value>jdbc:mysql://bigdataproxy:3306/metastore?createDatabaseIfNotExist=true</value>
    <description>
      JDBC connect string for a JDBC metastore.
      To use SSL to encrypt/authenticate the connection, provide database-specific SSL flag in the connection URL.
      For example, jdbc:postgresql://myhost/db?ssl=true for postgres database.
    </description>
  </property>

  <property>
    <name>javax.jdo.option.ConnectionDriverName</name>
    <value>com.mysql.jdbc.Driver</value>
    <description>Driver class name for a JDBC metastore</description>
  </property>
  <property>
    <name>javax.jdo.option.ConnectionUserName</name>
    <value>hiveuser</value>
    <description>Username to use against metastore database</description>
  </property>
  <property>
    <name>javax.jdo.option.ConnectionPassword</name>
    <value>hiveuser</value>
    <description>password to use against metastore database</description>
  </property>
  <property>
    <name>hive.metastore.uris</name>
    <value>thrift://bigdataproxy:9083</value>
    <description>Thrift URI for the remote metastore. Used by metastore client to connect to remote metastore.</description>
  </property>
  <property>
    <name>hive.server2.support.dynamic.service.discovery</name>
    <value>true</value>
    <description>Whether HiveServer2 supports dynamic service discovery for its clients. To support this, each instance of HiveServer2 currently uses ZooKeep>
  </property>
  <property>
    <name>hive.server2.zookeeper.namespace</name>
    <value>hiveserver2</value>
    <description>The parent node in ZooKeeper used by HiveServer2 when supporting dynamic service discovery.</description>
  </property>
  <property>
    <name>hive.zookeeper.quorum</name>
    <value>zk1:2181,zk2:2181,zk3:2181</value>
    <description>
      List of ZooKeeper servers to talk to. This is needed for:
      1. Read/write locks - when hive.lock.manager is set to
      org.apache.hadoop.hive.ql.lockmgr.zookeeper.ZooKeeperHiveLockManager,
      2. When HiveServer2 supports service discovery via Zookeeper.
      3. For delegation token storage if zookeeper store is used, if
      hive.cluster.delegation.token.store.zookeeper.connectString is not set
      4. LLAP daemon registry service
      5. Leader selection for privilege synchronizer
    </description>
  </property>

```

```bash
cd hive/lib/
rm -rf guava-22.0.jar
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


## Step 3: Start Hive Services (metastore and Hiveserver2)

For All 3 NN

```bash
nohup hive --service metastore > hive-metastore.log 2>&1 &
nohup hive --service hiveserver2 > hive-server2.log 2>&1 &
```
