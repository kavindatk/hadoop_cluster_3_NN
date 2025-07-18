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


