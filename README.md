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
```

```bash
source  ~/.bashrc #Update system
```

Now that we have successfully installed <b>Java 11 on all 5 servers</b>, I’ve also gone ahead and added the <b>basic Hadoop enviroment variables</b> to all the servers at the same time to make things easier.
In the next step, I’ll show you how to set up <b>passwordless SSH login</b> between all 5 servers so they can communicate with each other.
After that, we’ll install and configure <b>ZooKeeper on 3 of the nodes</b>. Once ZooKeeper is ready, we’ll move on to the <b>full Hadoop configuration and setup</b>.
