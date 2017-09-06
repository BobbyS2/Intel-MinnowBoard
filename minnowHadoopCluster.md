# The MinnowBoard HADOOP Cluster

[Published October 11, 2015 by Nicolas Di Tullio](http://blog.ditullio.fr/2015/10/11/mini-cluster-part-i-technical-financial-choices/)  
[Nico's Githib Files](https://github.com/nicomak)

The max speed between computers using this setup should be 1 Gbit/s = 125 MB/s  

## Setup
1. Install OS
2. Disable IPv6 to avoid network-related problems with hadoop  
    sudo vi /etc/ sysctl.conf
3. verify IPv6 was deactivated (command returns 1)  
    cat /proc/sys/net/ipv6/conf/all/disable_ipv6
4. Edit /etc/hosts file so that each host sees each other  
	sudo vi /etc/hosts 127.0.0.1 localhost  
	* [IP Address 0] [host0]
	* [IP Address 1] [host1]
	* [IP Address 2] [host2]
5. Install Java 1.7.0_21  
6. Create installation directory  
	* sudo mkdir /opt/jdk  
	* cd /opt  
7. Download archive  
	* sudo wget --header "Cookie: oraclelicense=accept-securebackup-cookie" http:// download.oracle.com/otn-pub/java/jdk/7u21-b11/jdk-7u21-linux-x64.tar.gz  
8. Unpack  
	* sudo tar -zxf jdk-7u21-linux-x64.tar.gz -C /opt/jdk
9. Update Ubuntu alternatives symbolic links  
	* sudo update-alternatives --install /usr/bin/java java /opt/jdk/jdk1.7.0_21/bin/java 100
	* sudo update-alternatives --install /usr/bin/javac javac /opt/jdk/jdk1.7.0_21/bin/javac 100  

## Hadoop & Spark Install
Install and configure Hadoop (2.7.1) and Spark (1.5.1) to have 1 master and 2 slaves.  The configurations in this part are adapted for MinnowBoards.
1. Create group hadoop and add hduser to the group $ sudo addgroup hadoop
	* sudo adduser --ingroup hadoop hduser
2. Edit the sudoers file to add hduser
	* sudo visudo
3. Add this to the last line of the visudo file 
	* hduser ALL=(ALL:ALL) ALL
4. login with hduser - generate password-less SSH key - add to authorized keys list
	* su - hduser
	* ssh-keygen -t rsa -P "" -f id_rsa
	* cat .ssh/id_rsa.pub >> .ssh/authorized_keys
5. verify [host0] - run for each of the other hosts
	* hduser@[host0]:  ssh-copy-id -i ~/.ssh/id_rsa.pub [host1] 
	* hduser@[host0]:  ssh-copy-id -i ~/.ssh/id_rsa.pub [host2]
6. Change data partition mount point ownership on each host
	* sudo chown -R hduser:hadoop /data
7. Create directories for HDFS
	* master host
		* mkdir /data/tmp 
		* mkdir /data/namenode
		* mkdir /data/namenode
	* slave hosts
		* mkdir /data/tmp 
		* mkdir /data/datanode 
		* mkdir /data/userlogs

## Hadoop Setup
1. Download and extract Hadoop to hduserʼs home folder.
	* $ cd ~
	* wget http://www.us.apache.org/dist/hadoop/common/hadoop-2.7.1/ hadoop-2.7.1.tar.gz
2. Extract the archive
	* tar -zxf hadoop-2.7.1.tar.gz
3. Rename to simple name 'hadoop'
	* mv hadoop-2.7.1 hadoop
4. define environment variables. 
	* Edit ~/.bashrc to add the following lines at the end # Java and Hadoop variables
		* export HADOOP_HOME=/home/hduser/hadoop
		* export JAVA_HOME=/opt/jdk/jdk1.7.0_21/
		* export PATH=$PATH:$JAVA_HOME/bin:$HADOOP_HOME/bin:$HADOOP_HOME/ sbin
		* export HADOOP_CLASSPATH=${JAVA_HOME}/lib/tools.jar
5. edit ~/hadoop/etc/hadoop/hadoop-env.sh - set the JAVA_HOME line to
		* export JAVA_HOME=/opt/jdk/jdk1.7.0_21/
6. Configure master and slaves to have HDFS and YARN up and running. Configuration files are always in 
	* ~/hadoop/etc/hadoop/ (master xml configuration files
		* core-site.xml 
		* hdfs-site.xml
		* yarn-site.xml
		* mapred-site.xml
	* Configure Master Update files 
		* hadoop/etc/hadoop/core-site.xml
		* fs.defaultFS hdfs://ubuntu0{9000
		* hadoop.tmp.dir /data/tmp
		* hadoop/etc/hadoop/hdfs-site.xml
		* dfs.replication 2
		* dfs.namenode.name.dir /data/namenode
		* hadoop/etc/hadoop/yarn-site.xml
		* yarn.resourcemanager.hostname ubuntu0
		* yarn.scheduler.minimum-allocation-mb 512
		* yarn.scheduler.maximum-allocation-mb 2048
		* yarn.scheduler.minimum-allocation-vcores 1
		* yarn.scheduler.maximum-allocation-vcores 4
		* yarn.log-aggregation-enable true
		* yarn.log-aggregation.retain-seconds 86400
	* How long to keep aggregation logs. Used by History Server.
		* hadoop/etc/hadoop/mapred-site.xml
		* mapreduce.framework.name yarn
		* yarn.app.mapreduce.am.resource.mb 512
		* yarn.app.mapreduce.am.command-opts -Xmx500m
		* mapreduce.map.memory.mb 512
		* mapreduce.map.cpu.vcores 1
		* mapreduce.reduce.memory.mb 512
		* mapreduce.reduce.cpu.vcores 1
		* mapreduce.job.reduces 2
		* mapreduce.jobhistory.address ubuntu0{10020
		* mapreduce.jobhistory.webapp.address ubuntu0{19888
		* hadoop/etc/hadoop/slaves ubuntu1
		* ubuntu2
		* ubuntu3
		* ubuntu4
		* Slaves Configuration hadoop/etc/hadoop/core-site.xml
			* fs.defaultFS hdfs://ubuntu0{9000
			* hadoop.tmp.dir /data/tmp
			* hadoop/etc/hadoop/hdfs-site.xml
			* dfs.datanode.data.dir /data/datanode
			* hadoop/etc/hadoop/yarn-site.xml
			* yarn.nodemanager.aux-services mapreduce_shuffle
			* yarn.resourcemanager.hostname ubuntu0
			* yarn.nodemanager.resource.memory-mb 2048
			* yarn.nodemanager.resource.cpu-vcores 4
			* yarn.log-aggregation-enable true
			* yarn.nodemanager.log-dirs /data/userlogs
			* yarn.log.server.url ubuntu0{19888
			* yarn.nodemanager.delete.debug-delay-sec 86400

## Configuration Notes
Here are a few explanations about the different properties, and how/why I chose these values for the MinnowBoard cluster.  
### HDFS Config
The replication factor dfs.replication defines on how many nodes a block of HDFS data is replicated across the cluster. The default value is 3, but I decided to lower it to 2. Here are the reasons why :
* Higher replication factor means more writing time, so lower performance. I have weak computers with only 1 HDD each, so I shouldnʼt abuse of it.
* It also means more disk space used ! Each node has only a small (~400 GB) data partition, so this can become a problem too. It is part of HDFS design to have at least one replica in a node from the same rack, and one replica in a node from another rack. This is in case a whole rack fails (for example broken switch), and that is why the default value is   * In my case there is no need for this rack-aware principle since I only have one rack.
* Having a replication factor of 2 means that I can still loose any slave, or choose to disconnect one slave to do other stuff, and its data will be recovered.
* The values dfs.namenode.name.dir on the master, and dfs.datanode.data.dir on the slaves didnʼt really need to be specified because by default they are set in the hadoop.tmp.dir directory, which we have defined in the “core” properties to be in our external HDDʼs data partition.
* However if you have multiple HDDs per node, these values can take a comma-separated list of URIs so that the 
* DataNode and NameNode can have better performance by spreading the I/O to multiple disks.

### YARN Config
The values set in the master are only lower and upper limits for container resources. YARN applications (such as MapReduce) wonʼt be able to ask YARN to create containers with amounts of memory and vcores out of these limits.  

In the slaves, the memory and vcore values define how much resources the node can lease to YARN containers. For example, you can have 8 GB of RAM on your computer and decide to let YARN use up to 4 GB of RAM by setting yarn.nodemanager.resource.memory-mb to 4096.  

####  MapReduce Config
The numerical values (memory, cpu, number of reducers, etc...) have been chosen after testing and tuning the cluster for the WordCount MapReduce job.  
- With 2 GB of RAM, best performance is 512 MB containers (defined in mapreduce.map.memory.mb and mapreduce.reduce.memory.mb ) giving 4 containers per node.  

#### Starting HDFS
When talking about HDFS, the master is called the NameNode, and a slave is called a DataNode.
1.  Format a new distributed file system using the command on the NameNode hduser@[node0]:~$ hdfs namenode -format
2. Then use this command on the namenode to start the DFS (Distributed File System)
- hduser@[node0]:~$ start-dfs.sh
- Starting namenodes on[node0]
- [node0] starting namenode, logging to /home/hduser/hadoop/logs/hadoop- hduser-namenode-[node0].out
- [node1] starting datanode, logging to /home/hduser/hadoop/logs/hadoop- hduser-datanode-[node1].out
- [node2] starting datanode, logging to /home/hduser/hadoop/logs/hadoop- hduser-datanode-[node2].out
- [node3] starting datanode, logging to /home/hduser/hadoop/logs/hadoop- hduser-datanode-[node3].out
- Starting secondary namenodes [0.0.0.0]
- 0.0.0.0: starting secondarynamenode, logging to /home/hduser/hadoop/logs/ hadoop-hduser-secondarynamenode-[node0].out
- stop HDFS
	* stop-dfs.sh  
- After starting, the NameNode has a web UI to view information about the distributed file system, located by default at http://{MASTER}:50070
- Go to the “Datanodes” page and check that all DataNodes are all in operation. If DataNodes are missing from this page, check the logs for errors on both master and slaves in the directory ~/hadoop/logs .
- You may also use this UI to browse the contents of the distributed file system.
- Before running any jobs, we must create a directory for our user in HDFS, which will be used for relative paths in MapReduce jobs
		* hdfs dfs -mkdir /user
		* hdfs dfs -mkdir /user/hduser
- HDFS shell reference https://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/
- FileSystemShell.html
- Start YARN
		* hduser@[node0]: /data/namenode$ start-yarn.sh
- Stop HDFS
		* stop-yarn.sh
- clusterʼs jobs, located at http://{MASTER}:8088
- If the “Cluster Metrics” on the first page show 0 active nodes, check the logs for errors on both master and slaves in the directory ~/hadoop/logs  

Resource Manager UI  
- You may also start the history server
	* mr-jobhistory-daemon.sh start historyserver starts a UI at address : http://{MASTER}:19888 to view information and aggregated logs for all previous MapReduce jobs  

### Running a MapReduce example  
This example uses MR to estimate the value of Pi using a statistic-based method called QuasiMonteCarlo
	* yarn jar $HADOOP_HOME/share/hadoop/mapreduce/*examples*.jar pi 50 100  

This MapReduce application automatically creates input files in the DFS at /user/ hduser/QuasiMonteCarlo_/in and then results are written in /user/hduser/ QuasiMonteCarlo_/out. You can view this data in the HDFS NameNode UI.  

This execution took about 2min 40s on my cluster. During that time, you can check the YARN ResourceManager UI to see that the job has been created. You can also view the created containers on each YARN ResourceManager UI, at http://{SLAVE}: 8042  

When the execution if finished, the command line will output the result (estimated value of Pi). And after that you can view the summary of all previous MR jobs in the MR History Server UI. For each job you may view information and logs about all containers that were created, across all nodes.  

You may also get all the aggregated logs on one node by running the command  
	* yarn logs --applicationId application_1445875607327_0008 > pi_app.log
	* This will create a file pi_app.log containing the aggregation of all container logs.  
	* The application_ parameter used here can be found in the console ouput, in the ResourceManager UI, or in the History Server UI.
	* We have run a MapReduce job from hadoop-mapreduce-examples-2.7.1.jar which contains a number of basic application examples to test Hadoop out of the box. 

### Install Spark
While logged in as hduser, download the archive and extract it:
	* cd ~
	* wget http://www.eu.apache.org/dist/spark/spark-1.5.1/spark-1.5.1-bin- hadoop2.6.tgz
	* tar -zxf spark-1.5.1-bin-hadoop2.6.tgz
	* mv spark-1.5.1-bin-hadoop2.6/ spark

Update environment variables in ~.bashrc
- export HADOOP_HOME=/home/hduser/hadoop
- export JAVA_HOME=/opt/jdk/jdk1.7.0_21/
- export HADOOP_CLASSPATH=${JAVA_HOME}/lib/tools.jar
- export PATH=$PATH:$JAVA_HOME/bin:$HADOOP_HOME/bin:$HADOOP_HOME/ sbin
- # Spark variables
- export YARN_CONF_DIR=$HADOOP_HOME/etc/hadoop
- export SPARK_HOME=/home/hduser/spark
- export PATH=$PATH:$SPARK_HOME/bin

To run Spark on YARN, you only need to do this on the master. To run Spark in Standalone mode, the binaries need to be installed on all nodes, so repeat these steps on each slave.

### Running a job
Spark also includes an example jar, containing pre-packaged examples and the QuasiMonteCarlo Pi estimation example.

* hduser@ubuntu0:~$ spark-submit \ --master yarn-client \ --num-executors 4 \ --executor-cores 2 \ --executor-memory 640M \
* --class org.apache.spark.examples.JavaSparkPi \ ~/spark/lib/spark-examples-1.5.1-hadoop2.6.0.jar \ 100

Options and parameters are  
– master : when running on YARN, 2 modes exist
- yarn-client : runs the Driver on the client which submits the spark job. In this case, this creates a process on the master, from which I executed the command.  
- yarn-cluster : runs the Driver on a slave node. This will create a process (for the driver) inside the ApplicationMaster container of one of the slave nodes.  
– num-executors : the number of executors. Each executor runs in its own YARN container.  
– executor-memory : the memory used for executors. This value is systematically padded with 384 MB overhead, so to reach exactly 1024 MB I chose the value 640 MB.  
– executor-cores : the number of cores to be used by each executor. The number of cores is also the number of parallel tasks that an executor can run.  
– class : the name of the Java class to use.  
- 1st parameter : the jar containing the class to be executed and eventually other dependency libraries. Here it is the Spark example jar.  
- 2nd to n-th parameters : input parameters for the Java application. Here we pass 100 as the number of samples to calculate Pi.  
- Each slave node runs 1 executor in a 1024 MB container. Each executor uses 2 cores.

### Spark in Standalone
Configuration and Startup  
Define Spark configuration
1. Create ~/spark/conf/spark-env.sh using template in the conf directory.  
2. Define the following values in each node worker memory  
	* SPARK_WORKER_MEMORY=2g  
3. Start stop spark master and slaves commands on master host
	*  hduser@ubuntu0:~$ ~/spark/sbin/start-all.sh  
	* hduser@ubuntu0:~$ ~/spark/sbin/stop-all.sh
4. Check the cluster information and jobs on the Spark web UI at http://{MASTER}:8080
5. Running a job in standalone is the same as on YARN, except that the “master” option takes in parameter the URI of the Spark master  
	* hduser@ubuntu0:~$ spark-submit \--master spark://ubuntu0{7077 \--class org.apache.spark.examples.JavaSparkPi \ ~/spark/lib/spark-examples-1.5.1-hadoop2.6.0.jar \ 100  

## Word Count Benchmark
### Word Count : Definition, Data Files and Choices
Word Count is a simple program which as its name suggests, is used to count the number of times each word is found in a text.  The input text is naturally split into different pieces, called blocks, in HDFS (in the case of Hadoop and Spark). Each block is processed line by line to count the number of words. It is a reference program for both MapReduce and Spark. Apache uses this program on both framework websites in their tutorial or examples to illustrate how they work.  
Create 5 benchmark text files  
|File | Size | HDFS Blocks |  
|----|-----|---------------|  
|tiny.txt | 200 MB |2 |  
|small.txt |760 MB |6|  
| medium.txt |2.6 GB |21 |  
|large.txt |9.6 GB |77|  
|huge.txt |26.2 GB |210|  

Download .txt files from Project Gutenberg in a background task, I used the following commands:
- mkdir /data/gutenberg
- cd /data/gutenberg
- wget -bqc -w 2 -m -H 'http://www.gutenberg.org/robot/harvest? filetypes[]=txt&langs[]=en'
- mkdir extracted
- find . -name '*.zip' -exec sh -c 'unzip -d extracted {}' ';'
- cat extracted/*.txt > bigfile.txt		// Ctrl+C to stop when big enough

The HDFS block size is 128 MB. Number of blocks determines number of MapReduce mappers or Spark tasks.

Word count improvements  
- Replacement of all characters which are not ASCII alpha-characters by a white space  
- Transform to lower case  
- After splitting line into words, ignore all words which are 50 or more characters long  

These are ethically questionable choices on many aspects (for Word Count purists) but here are the reasons why I decided to do so  
- Adds extra computing steps, which makes it look more like a real task. Otherwise there is nothing to process and the performance will depend only on disk read speed.  
- Makes the result more readable, gets rid of all punctuation (e.g “Hello!” ≠ “,hello” ≠ “hello;“).  
- Transforming to lowercase cuts down the number of different words.  
- Getting rid of the 50+ character words removes all science amino-acid sequences and other weird stuff which are not words anyways.  


