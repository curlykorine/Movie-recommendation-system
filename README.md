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

![image](https://user-images.githubusercontent.com/69847727/143784504-45f1828f-fbbc-4576-be80-c7e10043d379.png)

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
![image](https://user-images.githubusercontent.com/69847727/143784445-56663648-7fea-4b8e-97c1-8f3aa232ce6b.png)


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
![image](https://user-images.githubusercontent.com/69847727/143784432-714dc67c-9960-433d-b7f3-67b44b0d976e.png)


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

## Results

**The system works correctly both on the local Hadoop cluster and on the Private Network Hadoop cluster**
Here the movie recommendation system's example of output:

```xml
Movie                                                   | Index
--------------------------------------------------------|--------------------------------------------------------------------
Twilight (2008)                                         |8.044078222166666
Om Shanti Om (2007)                                     |7.957910635685204
Tracey Fragments, The (2007)                            |7.834045418517325
Beauty and the Bastard (Tyttö sinä olet tähti) (2005)   |7.692157305866834
Twilight Saga: Breaking Dawn - Part 2, The (2012)       |7.565078935400862
Twilight Saga: Eclipse, The (2010)               	|7.499100136439937
Hav Plenty (1997)               			|7.485961814981529
Twilight Saga: New Moon, The (2009)             	|7.317487277492542
Sweet Nothing (1996)            			|7.270011924082028
Twilight Saga: Breaking Dawn - Part 1, The (2011)       |7.224499331625905
Boys Don't Cry (Chlopaki nie placza) (2000)             |7.1108298527268285
Man's Job (Miehen työ) (2007)           		|6.948141601243929
For the Moment (1994)           			|6.692121242253773
Expelled: No Intelligence Allowed (2008)                |6.675079740325394
Collapse (2009)         				|6.656899751837235
Frank and Ollie (1995)          			|6.469878687838555
Jump Tomorrow (2001)            			|6.3356285061956115
Big Bang Theory, The (2007-)            		|6.289137893006393
Roll Bounce (2005)              			|6.269551702720389
United 93 (2006)                			|6.210990904798921

Error after training: 0.797378165762901
Baseline error: 0.9394505454302877

(local cluster with rank = 20)
```

## Analisys
**In this part we tried different ranks and checked how errors was changing**

```xml
rank 10 = Error after training: 0.7623745639126164
Baseline error: 0.9395930856067447

rank 20 = Error after training: 0.6953836128039833
Baseline error: 0.9395316247597332

rank 50 = Error after training: 0.7386361280398333
Baseline error: 0.9390870344101426

rank 100 = Error after training: 0.805983612803983
Baseline error: 0.940003462475963
```

We can conclude that the baseline error almostly did not change while test error differs from case to case.  
It is difficult to determine the boundaries of a "good" rank, but, according to the results, we can say that the model with rank <=20 underfits and the model with a rank >= 50 overfits. 

## Conclusion
We have completed the implementation of the movie recommendation system, tested the model on a local Hadoop cluster and on Private Network Hadoop cluster. 
We also identified the most suitable configurations of a model to minimize the test error.
