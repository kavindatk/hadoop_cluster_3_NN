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

<br/><br/>

### Technologies Used

For this setup, I’m using <b>Hadoop 3.4</b> along with <b>OpenJDK 11 (Java 11)</b>.
The steps below will guide you through <b>downloading, configuring, and launching the Hadoop cluster</b> in an Ubuntu environment.
