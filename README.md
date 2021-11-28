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


## Prerequirements
* Ubuntu OS
* Hadoop 3
* Spark 3
* Scala 2
* Java 8


## Project hierarchy
```xml
spark-recommendation-system  
├── configs.zip *  
├── build.sbt  
├── user_rating.tsv  
├── user_ratings.tsv 
├── rating2.csv *
├── movie2.csv *
└── src  
    ├── Grader.scala  
    └── MovieLensALS.scala  
```

* `configs.zip` contains configs for clustering.  
* `build.sbt` contains build configuration and dependencies.
* `user_ratings.tsv` this file is used for collecting user ratings.
* `user_rating.tsv` must be created for saving preferences and cannot be empty.
* `rating2.csv` and `movie2.csv` are datasets which are too large for storage in the repository. Please, dowload them from [here](https://www.dropbox.com/sh/y5uck2kbzizaes6/AACpapXw-JMmLJB_xCGOZOFCa?dl=0).
* `Grader.scala` for reading user preferences.
* `MovieLensALS.scala` contains main class for training recommendation system.


## General description
Initially, the movie recommendation system takes a dataset for training (`rating2.csv` and `movie2.csv`), files for generating ratings (`user_ratings.tsv` and `user_rating.tsv`) and a "true"or "false" flag, where the first tells the system to take into account the user's personal preferences while generating recommendations, and the second does not. In the first case the system asks whether user wants to load preferences from file or not. After that, regarding the answer, the system adds the preferences to the dataset for training the model or takes list of movies from `user_ratings.tsv` otherwise. After training the model predicts recommendations, shows baseline and test errors.

## Implementation
This function parses movies titles. As some of them have quotes, the code replaces the quotes with "" and splits the string by comma.
```xml
  def parseTitle(filmEntry: String) = {
    "\".*\"".r findFirstIn filmEntry match {
      case Some(s) => s.replace("\"","")
      case None => filmEntry.split(",")(1)
    }
  }
```
The next function is for computing test and baseline errors.
```xml
def rmse(test: RDD[Rating], prediction: scala.collection.Map[Int, Double], sparkContext: SparkContext) = {
    val z = sparkContext.parallelize(prediction.toSeq)
    val rating_pairs = z
      .map(x => (x._1, x._2))
      .join(test.map[(Int, Double)](x => (x.product, x.rating)))

    math.sqrt(rating_pairs.values
      .map(x => (x._1 - x._2) * (x._1 - x._2))
      .reduce(_ + _) / test.count())
  }

}

```
Here we do sorting to avoid intersections with list from `user_rating.csv`.
```xml
  // if user preferences are specified, predict top 20 movies for the user
    // default user id = 0. the same user id is set in Grader.scala
    if (doGrading){
      println("Predictions for user\n")
      val ids = sc
        .textFile(ratingsPath + "/user_rating.tsv")
        .map {
          _.split("\t")
        }
        .map { x => x(0).toInt }
        .collect()
        .toSet
      // for input data format refer to documentation
      // get movie ids to rank from baseline map
      model.predict(sc.parallelize(baseline.keys.map(x => (0, x)).toSeq))
        .filter(x => !ids.contains(x.product))
        // sort by ratings in descending order
        .sortBy(_.rating, false)
        // take first 20 items
        .take(20)
        // print title and predicted rating
        .foreach(x => println(s"${filmId2Title(x.product)}\t\t${x.rating}"))
    }

    sc.stop()
  }
```
Loading the file with the previously saved movie ratings to not enter them again.
```xml
println("Do you want to load save? y/[n]")
  var a = scala.io.StdIn.readLine()

  if (a == "y" || a == "yes" || a == "Yes" || a == "YES") {
    graded = sc
      .textFile(path + "/user_rating.tsv")
      .map {
        _.split("\t")
      }
      .map { x => (x(0).toInt, x(1).toDouble) }
      .collect()
      .toSeq
```
Here we set a threshold of 50 ratings so that the system can recommend a movie.
```xml
  def loadRatings(path: String, sc: SparkContext) = {

    // the file of interest (that contains ratings) is located in HDFS
    // files from HDFS are read with SparkContext's method textFile
    // by default textFile return RDD[String] ([] - specify template parameters)
    val ratingsDataPre = sc.textFile(path).map { line =>
      val fields = line.split(",")
      (fields(1).toInt, Rating(fields(0).toInt, fields(1).toInt, fields(2).toDouble)) // Rating = userID, movieID, movieRating
    }

    val ratingsDataFilter = ratingsDataPre
      .map(x => (x._1, 1)) // x._1 = movieID
      .reduceByKey(_+_)
      .filter(x => x._2 >= 50)

    val ratingsData =
      ratingsDataPre.join(ratingsDataFilter)
        .map(x => x._2._1)
```


## Clustering
**The following part describes the process of running Hadoop cluster (2 devices) on Private Network**

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

### ssh connection

*Ensure that all the machines are connected to the same network!*

**Hostnames**

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

### Passwordless ssh

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

### Hadoop configs

To share hadoop configs between nodes, we can use

```
sudo scp -r hadoop@roma:~/hadoop/etc/hadoop/* hadoop@evgen:~/hadoop/etc/hadoop
```

### workers

```
hadoop@evgen
hadoop@roma
```

### core-site.xml

Specify the namenode address

```xml
<property>
	<name>fs.defaultFS</name>
	<value>hdfs://roma:9000</value>
</property>
```

### hdfs-site.xml

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


### yarn-site.xml

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

### mapred-site.xml

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

### Commands

**Basic**

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

**Order**

After connecting the hosts to the cluster

```
hdfs namenode -format (if we want to erase file system)

start-dfs.sh

start-yarn.sh
```


### Tasks
**Firstly we tried to do some easy tasks on the cluster to check that that everything works correctly**

Estimating Pi value using Monte Carlo algorithm based on MapReduce.

```
hadoop jar ~/hadoop/share/hadoop/mapreduce/hadoop-mapreduce-examples-3.3.1.jar pi 100 40
```
**The main task with running the movie recommendation system**
Spark job for the assignment.

```
spark-submit --master yarn spark-recommendation_2.12-3.2.0_1.0.jar hdfs://roma:9000/user/hadoop/files -user true
```
