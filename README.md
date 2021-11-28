# **Movie recommendation system**

 **Scala programming:**   
 Evgeniy Lutanin, *y.lutanin@innopolis.university*  
 
 **Hadoop clustering:**  
 Roman Nabiullin, *r.nabiullin@innopolis.university*  
 
 **Report composing:**  
 Karina Singatullina, *k.singatullina@innopolis.university*  
 
*Introduction to Big Data,  
BS19-DS-01, BS19-AAI-01,  
**Skappa team**,  
2021*


## Introduction
The goal of our work was to complete the implementation of a movie recommendation system.  
The system contains a model trained on a dataset of graded movies and predicts (recommends) new ones to the user.  
The report describes our solution, results, analysis, difficulties encountered and alternative ways to implement.


## Project hierarchy
```xml
spark-recommendation-system  
├── build.sbt  
├── user_rating.tsv  
├── user_ratings.tsv 
├── rating2.csv *
├── movie2.csv *
└── src  
    ├── Grader.scala  
    └── MovieLensALS.scala  
```

* `build.sbt` contains build configuration and dependencies.
* `user_ratings.tsv` this file is used for collecting user ratings.
* `user_rating.tsv` must be created for saving preferences and cannot be empty.
* `rating2.csv` and `movie2.csv` are datasets which are too large for storage in the repository. Please, dowload them from [here](https://www.dropbox.com/sh/y5uck2kbzizaes6/AACpapXw-JMmLJB_xCGOZOFCa?dl=0).
* `Grader.scala` for reading user preferences.
* `MovieLensALS.scala` contains main class for training recommendation system.


## Description of the work done



## Create new user

Create *hadoop* user with home directory and assign a password to them.

```
sudo useradd -m hadoop

sudo passwd hadoop
```

Switch to *hadoop* user.

```
sudo su hadoop
```

Switch to bash and restart terminal.

```
chsh -s /bin/bash
```

Download hadoop v3.3.1 and unarchive it in */home/hadoop/*, i. e. in *~/*

# ssh connection

**Ensure that all the machines are connected to the same network!**

## Hostnames

Own ip in the network can be found as

```
hostname -I
```

Add hostname entries to */etc/hosts* file. For example, 

```
192.168.88.182 roma
192.168.88.219 evgen
```

This step should be repeated after changing a network.

## Passwordless ssh

Generate keys and distributed them between **all** the machines in the cluster including the own one.

```
ssh-keygen -b 4096

ssh-copy-id -i $HOME/.ssh/id_rsa.pub  hadoop@roma
ssh-copy-id -i $HOME/.ssh/id_rsa.pub  hadoop@evgen
```

After that, it should be possible to establish ssh connection without password with

```
ssh hadoop@evgen
```

# Hadoop configs

To share hadoop configs between nodes, we can use

```
sudo scp -r hadoop@roma:~/hadoop/etc/hadoop/* hadoop@evgen:~/hadoop/etc/hadoop
```

## workers

```
hadoop@evgen
hadoop@roma
```

## core-site.xml

Specify the namenode address

```xml
<property>
	<name>fs.defaultFS</name>
	<value>hdfs://roma:9000</value>
</property>
```

## hdfs-site.xml

Folder for temporary files. This is necessary to ensure data persistence across datanodes.

```xml
<property>
    <name>hadoop.tmp.dir</name>
    <value>/home/hadoop/hadoop_tmp</value>
</property>
```

The number of replication in the file system.

```xml
<property>	
    <name>dfs.replication</name>
    <value>1</value>
</property>
```


## yarn-site.xml

Node on which the resource manager runs

```xml
<property>
    <name>yarn.resourcemanager.hostname</name>
    <value>roma</value>
</property>
```

Specify ports for YARN services.

```xml
<property>
    <name>yarn.resourcemanager.scheduler.address</name>
    <value>roma:8030</value>
</property>
```

```xml
<property>
    <name>yarn.resourcemanager.resource-tracker.address</name>
    <value>roma:8031</value>
</property>
```

```xml
<property>
    <name>yarn.resourcemanager.address</name>
    <value>roma:8032</value>
</property>
```

```xml
<property>
    <name>yarn.resourcemanager.admin.address</name>
    <value>roma:8033</value>
</property>
```

Some strange but important config fields.

```xml
<property>
    <name>yarn.resourcemanager.webapp.address</name>
    <value>roma:8088</value>
</property>
```

```xml
<property>
	<name>yarn.nodemanager.aux-services</name>
	<value>mapreduce_shuffle</value>
</property>
```

```xml
<property>
	<name>yarn.nodemanager.aux-services.mapreduce_shuffle.class</name>
	<value>org.apache.hadoop.mapred.ShuffleHandler</value>
</property>
```

```xml
<property>
	<name>yarn.nodemanager.vmem-check-enabled</name>
	<value>false</value>
	<description>Whether virtual memory limits will be enforced for containers</description>
</property>
```

## mapred-site.xml

Tell that we want to use YARN

```xml
<property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
</property>
```

Specify MapReduce folders.

```xml
<property>
  <name>yarn.app.mapreduce.am.env</name>
  <value>HADOOP_MAPRED_HOME=/home/hadoop/hadoop</value>
</property>
```

```xml
<property>
  <name>mapreduce.map.env</name>
  <value>HADOOP_MAPRED_HOME=/home/hadoop/hadoop</value>
</property>
```

```xml
<property>
  <name>mapreduce.reduce.env</name>
  <value>HADOOP_MAPRED_HOME=/home/hadoop/hadoop</value>
</property>
```

## Commands

### Basic

Format hadoop file system

```
hdfs namenode -format
```

Start hadoop file system.

```
start-dfs.sh
```

Information about datanodes

```
hdfs dfsadmin -report
```

Stop hadoop file system.

```
stop-dfs.sh
```

Start yarn daemon.

```
start-yarn.sh
```

Stop yarn daemon.

```
stop-yarn.sh
```

Stop all hadoop daemons.

```
stop-all.sh
```

List all nodes in YARN in all states.

```
yarn node -list -all
```

### Order

After connecting the hosts to the cluster

```
hdfs namenode -format (if we want to erase file system)

start-dfs.sh

start-yarn.sh
```


## Tasks

Estimating Pi value using Monte Carlo algorithm based on MapReduce.

```
hadoop jar ~/hadoop/share/hadoop/mapreduce/hadoop-mapreduce-examples-3.3.1.jar pi 100 40
```

Spark job for the assignment.

```
spark-submit --master yarn spark-recommendation_2.12-3.2.0_1.0.jar hdfs://roma:9000/user/hadoop/files -user true
```
