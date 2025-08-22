# 02. Apache Hadoop Cluster - 3 NameNode Setup

<br/><br/>
<p align="center">
<picture>
  <img alt="docker" src="https://github.com/kavindatk/hadoop_cluster_3_NN/blob/main/images/Hadoop_logo_new.jpg" width="400" height="200">
</picture>
  
<picture>
  <img alt="docker" src="https://github.com/kavindatk/hadoop_cluster_3_NN/blob/main/images/zoekeeper.jpg" width="400" height="200">
</picture>
</p>

<br/><br/>


## Introduction to Hadoop Big Data Cluster Setup

In the previous article, we set up a <b>3-node MariaDB Galera cluster</b>. Now, we will use the same cluster as part of a <b>Hadoop Big Data setup</b> with <b>three NameNodes</b> for high availability (HA).

In my setup, I have <b>five servers</b>:

1. <b>Three servers</b> are used as <b>master nodes</b> to enable HA.
2. <b>Two servers</b> are used as <b>DataNodes (workers).</b>

To maintain high availability, we need <b>ZooKeeper</b>, which plays a key role in managing the cluster. ZooKeeper decides which NameNode is the <b>leader</b> and which ones are <b>followers</b>, based on availability.

ZooKeeper works best with an <b>odd number</b> of nodes (such as 3, 5, 7, or 9). That’s why I chose a <b>3-NameNode setup</b> instead of just 2 — to ensure reliable <b>quorum-based decision making.</b>

<br/>

### Technologies Used

For this setup, I’m using <b>Hadoop 3.4 and Zookeeper 3.8.4</b> along with <b>OpenJDK 11 (Java 11)</b>.
The steps below will guide you through <b>downloading, configuring, and launching the Hadoop cluster</b> in an Ubuntu environment.

<br/>

## Step 1: Pre-Installation Setup

In this step, we will prepare all 5 servers for the Hadoop cluster.

1. First, we’ll download the <b>Hadoop and ZooKeeper tar files</b>.

2. Then, we’ll install <b>Java (OpenJDK 11) on all 5 nodes</b>.

3. We’ll also set up <b>Java and Hadoop environment variables</b> to be used in the next steps.

4. Additionally, we’ll enable <b>passwordless SSH login between all servers</b>, which is required for cluster communication.

5. Note: <b>ZooKeeper</b> will only be configured on <b>3 of the 5 nodes</b>, as it works best with an odd number of nodes.

The following steps will walk you through this setup with commands.


## Step 2: Download, Extract, and Rename

The following steps show how to download the current stable versions of Hadoop and ZooKeeper, extract the files, and rename the directories for easier use.

#### ZOOKEEPER

```bash
wget https://dlcdn.apache.org/zookeeper/zookeeper-3.8.4/apache-zookeeper-3.8.4-bin.tar.gz

tar -xvzf apache-zookeeper-3.8.4-bin.tar.gz

mv apache-zookeeper-3.8.4-bin zookeepe

```


#### HADOOP

```bash
wget https://dlcdn.apache.org/hadoop/common/hadoop-3.4.0/hadoop-3.4.0.tar.gz

tar -xvzf hadoop-3.4.0.tar.gz

mv hadoop-3.4.0 hadoop
```

## Step 3: Install Java 11 and Set Environment Variables

In this step, we will install Java 11 (OpenJDK 11) on all 5 servers.
Java is a core requirement for setting up components in the Apache Bigtop ecosystem.

After installing Java, we’ll also configure the necessary environment variables so the system can recognize and use Java properly in the upcoming steps. 

```bash
sudo apt-get install openjdk-11-jdk
```

```bash
sudo update-alternatives --config java  #to get the path
```

<picture>
  <img alt="docker" src="https://github.com/kavindatk/hadoop_cluster_3_NN/blob/main/images/java_path.JPG" width="600" height="200">
</picture>


```bash
sudo nano ~/.bashrc
```
I have added Hadoop configurations as well

```bash
#Hadoop Related Options
export HADOOP_HOME=/opt/hadoop
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
export HADOOP_INSTALL=$HADOOP_HOME
export HADOOP_MAPRED_HOME=$HADOOP_HOME
export HADOOP_COMMON_HOME=$HADOOP_HOME
export HADOOP_HDFS_HOME=$HADOOP_HOME
export HADOOP_YARN_HOME=$HADOOP_HOME
export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
export PATH=$PATH:$HADOOP_HOME/sbin:$HADOOP_HOME/bin:$JAVA_HOME/bin
export HADOOP_OPTS="-Djava.library.path=$HADOOP_HOME/lib/native"
export HADOOP_CONF_DIR=/opt/hadoop/etc/hadoop


```

```bash
source  ~/.bashrc #Update system
```

Now that we have successfully installed <b>Java 11 on all 5 servers</b>, I’ve also gone ahead and added the <b>basic Hadoop environment variables</b> to all the servers at the same time to make things easier.
In the next step, I’ll show you how to set up <b>passwordless SSH login</b> between all 5 servers so they can communicate with each other.
After that, we’ll install and configure <b>ZooKeeper on 3 of the nodes</b>. Once ZooKeeper is ready, we’ll move on to the <b>full Hadoop configuration and setup</b>.

## Step 4: Set Up Passwordless SSH Login

In this step, I will show you how to <b>set up passwordless SSH login</b> between all servers.

This step is <b>mandatory</b>, as passwordless login is required for Hadoop nodes to communicate with each other during cluster operations.

```bash
# Configure SSH daemon:
ssh-keygen -A  
sudo sed -i 's/#PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config 
sudo sed -i 's/#PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config
```

```bash
# Add 'hadoop' to the 'wheel' group to allow sudo access 
sudo su 
useradd -m -s /bin/bash hadoop 
echo "hadoop:hadoop" | chpasswd 
usermod -aG sudo hadoop
```

```bash
# Generate SSH keys for 'hadoop' user
mkdir -p /home/hadoop/.ssh 
ssh-keygen -t rsa -b 4096 -f /home/hadoop/.ssh/id_rsa -N "" 
cp /home/hadoop/.ssh/id_rsa.pub /home/hadoop/.ssh/authorized_keys 
chmod 700 /home/hadoop/.ssh 
chmod 600 /home/hadoop/.ssh/authorized_keys 
chown -R hadoop:hadoop /home/hadoop/.ssh
```

```bash
# Copy public key to others
ssh-copy-id hadoop@<other-server-ip>
```

## Step 5: Install and Configure ZooKeeper

In this step, I will show you how to install ZooKeeper on 3 of the 5 nodes.
As part of the setup, we will also add environment variables related to ZooKeeper, so the system can access it easily.
The following steps will guide you through the installation and configuration of ZooKeeper on the selected nodes.

<br/>

### Set ZooKeeper Environment Variables

Before we proceed with the full configuration, we need to <b>set some required environment variables for ZooKeeper</b>.

These variables will help the system locate and run ZooKeeper properly.
I’ll add the following environment variables on <b>all 3 master nodes</b>, since we are using a 3-node ZooKeeper setup.

```bash
sudo nano ~/.bashrc
```

```bash
#Zookeeper Related Options
export ZOOKEEPER_HOME=/opt/zookeeper
export PATH=$PATH:$ZOOKEEPER_HOME/bin
```

```bash
source  ~/.bashrc #Update system
```
<br/>

### Set ZooKeeper Environment Variables

In this stage, we will add the necessary configuration files for ZooKeeper and then start the ZooKeeper service on all 3 nodes.

The following steps show how to complete the configuration and start the service.

```bash
# Create data directory
sudo mkdir -p /opt/zookeeper/data
sudo mkdir -p /opt/zookeeper/logs
sudo chown -R hadoop:hadoop /opt/zookeeper/data
sudo chown -R hadoop:hadoop /opt/zookeeper/logs
```

```bash
cd /opt/zookeeper/conf/
cp zoo_sample.cfg zoo.cfg
nano zoo.cfg
````

```bash
#Add or modifiy following
dataDir=/opt/zookeeper/data
dataLogDir=/opt/zookeeper/logs


# Cluster configuration
server.1=mst01:2888:3888
server.2=mst02:2888:3888
server.3=mst03:2888:3888

# Optional settings
maxClientCnxns=60
autopurge.snapRetainCount=3
autopurge.purgeInterval=24
```

#### Create Server ID Files

###### On Server 1 (zk1):

```bash
echo "1" | sudo tee /opt/zookeeper/data/myid
sudo chown hadoop:hadoop /opt/zookeeper/data/myid
```

###### On Server 2 (zk2):

```bash
echo "2" | sudo tee /opt/zookeeper/data/myid
sudo chown hadoop:hadoop /opt/zookeeper/data/myid
```

###### On Server 3 (zk3):

```bash
echo "3" | sudo tee /opt/zookeeper/data/myid
sudo chown hadoop:hadoop /opt/zookeeper/data/myid
```


###### Start ZOOKEEPER service and verify (All Servers)

```bash
zkServer.sh start
zkServer.sh status
```


<picture>
  <img alt="docker" src="https://github.com/kavindatk/hadoop_cluster_3_NN/blob/main/images/zookeeper.JPG" width="800" height="400">
</picture>


If the setup is successful, you will see that <b>one ZooKeeper node is marked as the Leader, while the other two nodes act as Followers.</b>

This confirms that the <b>ZooKeeper quorum is working correctly</b>.


#### HADOOP

So far, we have successfully set up <b>ZooKeeper on the 3 master nodes</b>.
Now, we’ll begin the <b>Hadoop installation on all 5 servers</b>.

To keep things simple, <b>I will configure Hadoop on one node only</b>. Once it’s set up, <b>I’ll copy the configuration and Hadoop files to the other four servers</b>.

In most Hadoop setups, you don’t need separate configurations for each server — <b>all nodes share the same configuration files</b>.

The following steps show the detailed Hadoop configuration, including the necessary files and settings required to run the cluster smoothly.


```bash
sudo mv hadoop /opt/
sudo chown -R hadoop:hadoop hadoop
cd /opt/hadoop/etc/hadoop/
```

#### Configuration Files

##### 1. Core Configuration (core-site.xml)

```xml
<property>
	<name>fs.defaultFS</name>
	<value>hdfs://hadoop-cluster</value>
</property>
<property>
	<name>hadoop.tmp.dir</name>
	<value>/opt/hadoop/tmp/hadoop-${user.name}</value>
</property>
<property>
	<name>hadoop.proxyuser.hue.hosts</name>
	<value>*</value> <!-- For futre HUE setup -->
</property>
<property>
	<name>hadoop.proxyuser.hue.groups</name>
	<value>*</value> <!-- For futre HUE setup -->
</property>
<property>
	<name>hadoop.proxyuser.hadoop.hosts</name>
	<value>*</value>
</property>
<property>
	<name>hadoop.proxyuser.hadoop.groups</name>
	<value>*</value>
</property>
<property>
	<name>hadoop.proxyuser.hive.groups</name>
	<value>*</value>  <!-- For futre Hive setup -->
</property>
<property>
	<name>hadoop.proxyuser.hive.hosts</name>
	<value>*</value> <!-- For futre Hive setup -->
</property>
<property>
	<name>hadoop.proxyuser.httpfs.groups</name>
	<value>*</value>
</property>

<property>
	<name>hadoop.proxyuser.httpfs.hosts</name>
	<value>*</value>
</property>
<property>
	<name>ha.zookeeper.quorum</name>
	<value>mst01:2181,mst02:2181,mst03:2181</value>
</property>
```


##### 2. HDFS Configuration (hdfs-site.xml)

```xml
<property>
	<name>dfs.nameservices</name>
	<value>hadoop-cluster</value>
</property>
<property>
	<name>dfs.ha.namenodes.hadoop-cluster</name>
	<value>nn1,nn2,nn3</value>
</property>
<property>
	<name>dfs.namenode.rpc-address.hadoop-cluster.nn1</name>
	<value>mst01:8020</value>
</property>
<property>
	<name>dfs.namenode.rpc-address.hadoop-cluster.nn2</name>
	<value>mst02:8020</value>
</property>
<property>
	<name>dfs.namenode.rpc-address.hadoop-cluster.nn3</name>
	<value>mst03:8020</value>
</property>
<property>
	<name>dfs.namenode.http-address</name>
	<value>0.0.0.0:9870</value>
</property>
<property>
	<name>dfs.client.failover.proxy.provider.hadoop-cluster</name>
	<value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
</property>
<property>
	<name>dfs.namenode.shared.edits.dir</name>
	<value>qjournal://mst01:8485;mst02:8485;mst03:8485/hadoop-cluster</value>
</property>
<property>
	<name>dfs.ha.fencing.methods</name>
	<value>sshfence</value>
</property>
<property>
	<name>dfs.ha.fencing.ssh.private-key-files</name>
	<value>/home/hadoop/.ssh/id_rsa</value>
</property>
<property>
	<name>dfs.ha.automatic-failover.enabled</name>
	<value>true</value>
</property>
<property>
	<name>dfs.ha.nn.not-become-active-in-safemode</name>
	<value>true</value>
</property>
<property>
	<name>dfs.journalnode.edits.dir</name>
	<value>/opt/hadoop/hdfs/journal</value>
</property>
<property>
	<name>dfs.datanode.address</name>
	<value>0.0.0.0:50010</value>
</property>
<property>
	<name>dfs.datanode.http.address</name>
	<value>0.0.0.0:50075</value>
</property>
<!-- property>
	<name>dfs.hosts.exclude</name>
	<value>/opt/hadoop/etc/hadoop/dfs.exclude</value>
</property -->
<property>
	<name>dfs.namenode.name.dir</name>
	<value>/opt/hadoop/hdfs/namenode</value>
</property>
<property>
	<name>dfs.namenode.checkpoint.dir</name>
	<value>/opt/hadoop/hdfs/checkpoint</value>
</property>
<property>
	<name>dfs.datanode.data.dir</name>
	<value>/opt/hadoop/hdfs/datanode</value>
</property>
<property>
	<name>dfs.replication</name>
	<value>1</value>
</property>
<property>
	<name>dfs.blocksize</name>
	<value>268435456</value> <!-- 256 MB -->
</property>
<property>
	<name>dfs.heartbeat.interval</name>
	<value>3</value>
</property>
<property>
	<name>dfs.namenode.checkpoint.period</name>
	<value>3600</value> <!-- Every hour -->
</property>
<property>
	<name>dfs.permissions.enabled</name>
	<value>true</value>
</property>
<property>
	<name>dfs.namenode.edits.journal-plugin.hdfs</name>
	<value>org.apache.hadoop.hdfs.qjournal.client.QuorumJournalManager</value>
</property>
<property>
	<name>dfs.namenode.edits.dir</name>
	<value>/opt/hadoop/hdfs/edits</value>
</property>
<property>
	<name>dfs.datanode.ipc.address</name>
	<value>0.0.0.0:8010</value>
</property>
<property>
	<name>dfs.user.home.dir.prefix</name>
	<value>/user</value>
</property>
<property>
	<name>dfs.namenode.acls.enabled</name>
	<value>true</value>
</property>

```

##### 3. MapReduce Configuration (mapred-site)

```xml
<property>
	<name>mapreduce.framework.name</name>
	<value>yarn</value>
</property>
<property>
	<name>mapreduce.jobtracker.address</name>
	<value>localhost:8032</value>
</property>
<property>
	 <name>mapreduce.jobhistory.done-dir</name>
	 <value>/opt/hadoop/hdfs/jobhistory/done</value>
</property>
<!-- Shuffle Service Configuration -->
<property>
	<name>mapreduce.shuffle.port</name>
	<value>13562</value>
</property>

<!-- Default Memory Configurations -->
<property>
	<name>mapreduce.map.memory.mb</name>
	<value>1024</value> <!-- 1 GB -->
</property>
<property>
	<name>mapreduce.reduce.memory.mb</name>
	<value>1024</value> <!-- 1 GB -->
</property>
<property>
	<name>mapreduce.map.java.opts</name>
	<value>-Xmx1536m</value>
</property>
<property>
	<name>mapreduce.reduce.java.opts</name>
	<value>-Xmx3072m</value>
</property>
<property>
	<name>mapreduce.jobhistory.intermediate-done-dir</name>
	<value>/opt/hadoop/hdfs/jobhistory/intermediate-done</value>
</property>
<property>
	<name>mapreduce.jobhistory.address</name>
	<value>0.0.0.0:10020</value>
</property>
<property>
	<name>mapreduce.jobhistory.webapp.address</name>
	<value>0.0.0.0:19888</value>
</property>
<property>
	<name>yarn.app.mapreduce.am.env</name>
	<value>HADOOP_MAPRED_HOME=/opt/hadoop</value>
</property>
<property>
	<name>mapreduce.map.env</name>
	<value>HADOOP_MAPRED_HOME=/opt/hadoop</value>
</property>
<property>
	<name>mapreduce.reduce.env</name>
	<value>HADOOP_MAPRED_HOME=/opt/hadoop</value>
</property>
```

##### 4. YARN Configuration (yarn-site.xml)

```xml
<property>
	<name>yarn.resourcemanager.ha.enabled</name>
	<value>true</value>
</property>
<property>
	<name>yarn.resourcemanager.cluster-id</name>
	<value>hadoop-cluster</value>
</property>
<property>
	<name>yarn.resourcemanager.ha.rm-ids</name>
	<value>rm1,rm2,rm3</value>
</property>
<property>
	<name>yarn.resourcemanager.hostname.rm1</name>
	<value>mst01</value>
</property>
<property>
	<name>yarn.resourcemanager.hostname.rm2</name>
	<value>mst02</value>
</property>
<property>
	<name>yarn.resourcemanager.hostname.rm3</name>
	<value>mst03</value>
</property>
<property>
	<name>yarn.resourcemanager.address.rm1</name>
	<value>mst01:8032</value>
</property>
<property>
	<name>yarn.resourcemanager.address.rm2</name>
	<value>mst02:8032</value>
</property>
<property>
	<name>yarn.resourcemanager.address.rm3</name>
	<value>mst03:8032</value>
</property>
<property>
	<name>yarn.nodemanager.aux-services</name>
	<value>mapreduce_shuffle</value>
</property>
<property>
	<name>yarn.resourcemanager.webapp.address.rm1</name>
	<value>mst01:8088</value>
</property>
<property>
	<name>yarn.resourcemanager.webapp.address.rm2</name>
	<value>mst02:8088</value>
</property>
<property>
	<name>yarn.resourcemanager.webapp.address.rm3</name>
	<value>mst03:8088</value>
</property>
<property>
	<name>yarn.log-aggregation-enable</name>
	<value>true</value>
</property>
<property>
	<name>yarn.nodemanager.resource.memory-mb</name>
	<value>6144</value>
</property>
<property>
	<name>yarn.scheduler.minimum-allocation-mb</name>
	<value>1024</value>
</property>
<property>
	<name>yarn.scheduler.maximum-allocation-mb</name>
	<value>6144</value>
</property>
<property>
	<name>yarn.resourcemanager.ha.automatic-failover.enabled</name>
	<value>true</value>
</property>
<property>
	<name>yarn.client.failover-proxy-provider</name>
	<value>org.apache.hadoop.yarn.client.ConfiguredRMFailoverProxyProvider</value>
</property>

<!-- Zookeeper Service -->
<property>
	<name>yarn.resourcemanager.zk-address</name>
	<value>mst01:2181,mst02:2181,mst03:2181</value>
</property>

```

##### 5. Workers Configuration (workers)

```xml
slv01
slv02
```

##### 6. Masters Configuration (masters)

```xml
mst01
mst02
mst03
```

##### 7.Hadoop Environment (hadoop-env.sh)

```bash
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk
export HADOOP_CONF_DIR=/opt/hadoop/etc/hadoop
export HADOOP_LOG_DIR=/opt/hadoop/logs
```

##### 8. Directory Creation
```bash
sudo mkdir -p /opt/hadoop/tmp
sudo mkdir -p /opt/hadoop/hdfs/namenode
sudo mkdir -p /opt/hadoop/hdfs/datanode
sudo mkdir -p /opt/hadoop/hdfs/checkpoint
sudo mkdir -p /opt/hadoop/hdfs/edits
sudo mkdir -p /opt/hadoop/hdfs/jobhistory
sudo mkdir -p /opt/hadoop/hdfs/journal
sudo mkdir -p /opt/hadoop/logs
sudo chown -R hadoop:hadoop /opt/hadoop/
```

<br/>

## Step 6: Format the Cluster and Start Hadoop Services

So far, we’ve completed the necessary XML configuration for a <b>3-master Hadoop cluster</b>.
You may have noticed that in some of the configuration files, I’ve used the hostname ``` bigdataproxy ```.

This is because I’m planning to use <b>HAProxy</b> for load balancing the <b>Timeline Service and Job History Server</b>.
Hadoop does not have a built-in load balancer for these components, so HAProxy will act as a workaround.
(I will explain how to set up HAProxy in a later step.)

For now, I’ve included ``` bigdataproxy ``` in the configuration, since it’s required for proper service routing.

The steps below will show you how to:

    1. Distribute files across the Hadoop cluster
    2. Format the Hadoop cluster
    3. Start the required services
    4. Verify the cluster status

##### 1. Distribute files across the Hadoop cluster

From mst01, copy configuration to all other nodes:

```bash
for node in mst02 mst03 slv01 slv02; do
    scp -r /opt/hadoop hadoop@$node:/opt/
done
```

##### 2. Format the Hadoop cluster

On mst01

```bash
#hdfs namenode -initializeSharedEdits
hdfs zkfc -formatZK
hadoop-daemon.sh start journalnode
```

On mst02 and mst03

```bash
hadoop-daemon.sh start journalnode
```
After that , again in mst01

On mst01

```bash
hdfs namenode -format 
hdfs --daemon start namenode
```

On mst02 and mst03

```bash
hdfs namenode -bootstrapStandby 
hdfs --daemon start namenode
```

##### 3. Start the required services

From mst01 , start the hadoop 

```bash
start-all.sh

stop-all.sh # For refresh (Optional)

start-all.sh
```

From All master start JobHistoryServer & ApplicationHistoryServer (Timeline Service)

```bash
yarn --daemon start timelineserver 
mapred --daemon start historyserver  
```

##### 4. Verify the cluster status


###### 1. Check HDFS Status
```bash
jps

hdfs dfsadmin -report
hdfs haadmin -getServiceState nn1
hdfs haadmin -getServiceState nn2
hdfs haadmin -getServiceState nn3
```

###### 2. Check YARN Status
```bash
yarn node -list
yarn application -list
```

###### 3. Web Interfaces
- NameNode (Active): http://< master node >:9870
- NameNode (Standby): http://< master node >:9870
- Resourc eManager: http://< master node >:8088
- Node Manager UI: http://< slave node >:8042
- DataNode UI: http://< slave node >:50075

###### 4. Test HDFS Operations
```bash
hdfs dfs -mkdir /test
hdfs dfs -put /etc/passwd /test/
hdfs dfs -ls /test
hdfs dfs -cat /test/passwd
```

###### 5. Test MapReduce
```bash
yarn jar $HADOOP_HOME/share/hadoop/mapreduce/hadoop-mapreduce-examples-3.4.0.jar pi 2 10
```

<picture>
  <img alt="docker" src="https://github.com/kavindatk/hadoop_cluster_3_NN/blob/main/images/verify_hadoop.JPG" width="800" height="400">
</picture>



## Step 7 : Enable Hadoop HTTPFS for Hue Integration

Now that we have successfully configured the <b>3-NameNode Hadoop cluster</b>, the next step is to <b>enable Hadoop HTTPFS (HTTP File System) mode</b>.

This feature will be useful in future steps, especially for integrating with Hue, which uses HTTPFS to browse and manage HDFS files through the web interface.

The following steps show how to <b>configure and enable HTTPFS</b> on the cluster.

```bash
cd /opt/hadoop/etc/hadoop/
nano httpfs-env.sh
```

Then add or modify

```xml
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
```

```bash
nano httpfs-site.xml
```

Then add or modify

```xml
	<property>
		  <name>httpfs.proxyuser.hue.hosts</name>
		  <value>*</value>
	</property>
	<property>
		  <name>httpfs.proxyuser.hue.groups</name>
		  <value>*</value>
	</property>
```


```bash
nano core-site.xml
```

Then add or modify

```xml
	<property>
		  <name>hadoop.proxyuser.httpfs.groups</name>
		  <value>*</value>
	</property>
	<property>
		  <name>hadoop.proxyuser.httpfs.hosts</name>
		  <value>*</value>
	</property> 
```

##### Start and Verify the httpfs service

```bash
hdfs --daemon start httpfs

# Verify the service

jps
ss -tulpn | grep 14000
curl -sS 'http://localhost:14000/webhdfs/v1?op=gethomedirectory&user.name=hdfs'
curl -sS 'http://hadoop-cluster:14000/webhdfs/v1?op=gethomedirectory&user.name=hdfs'
curl -i -X GET 'http://localhost:14000/webhdfs/v1?op=gethomedirectory&user.name=hdfs'
curl -i -X GET 'http://hadoop-cluster:14000/webhdfs/v1?op=gethomedirectory&user.name=hdfs'
```

<br/>

<picture>
  <img alt="docker" src="https://github.com/kavindatk/hadoop_cluster_3_NN/blob/main/images/httpfs_service.JPG" width="800" height="400">
</picture>
