# Install Java (OpenJDK 1.8)

1. sudo vi /etc/apt/sources.list, add the line: `deb http://security.ubuntu.com/ubuntu bionic-security main universe`
2. sudo add-apt-repository ppa:openjdk-r/ppa
3. sudo apt-get update
4. apt-cache search openjdk (check if openjdk-8-jdk is in the list)
5. sudo apt-get install openjdk-8-jdk
6. Set JAVA_HOME: add the line `JAVA_HOME="/usr"` in /etc/environment

# Install Hadoop

*Do the following for all machines unless explicitly stated*
1. Extract hadoop-2.9.2.tar.gz under /home/megaa/stuff
2. Put the following content in /etc/profile.d/hadoop.sh
```
HADOOP_PREFIX=/home/megaa/stuff/hadoop-2.9.2
export HADOOP_PREFIX
```
3. etc/hadoop/core-site.xml
```
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://192.168.0.103:9000</value>
    </property>
</configuration>
```
4. etc/hadoop/hdfs-site.xml
```
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
    
    <property>
	<name>dfs.namenode.name.dir</name>
	<value>/home/megaa/hadoop/data/namenode</value>
    </property>

    <property>
	<name>dfs.datanode.data.dir</name>
	<value>/home/megaa/hadoop/data/datanode</value>
    </property>
</configuration>
```
5. etc/hadoop/yarn-site.xml
```
<configuration>
    <property>
        <description>The hostname of the RM.</description>
        <name>yarn.resourcemanager.hostname</name>
        <value>192.168.0.103</value>
    </property>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
</configuration>
```
6. etc/hadoop/hadoop-env.sh
```
export JAVA_HOME="/usr"
```
7. Setup passphraseless SSH (refer to http://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/SingleCluster.html), make sure you can ssh to each other without passphrase for all machines
8. etc/hadoop/slaves (only on the master machine)
```
192.168.0.104
192.168.0.105
```
9. On master machine (192.168.0.103), mkdir -p /home/megaa/hadoop/data/namenode
10. On slave machines, mkdir -p /home/megaa/hadoop/data/datanode
11. Format the HDFS volume: on master machine, /home/megaa/stuff/hadoop-2.9.2, run
```
bin/hdfs namenode -format
```

# Run Hadoop
On master machine (192.168.0.1.3), /home/megaa/stuff/hadoop-2.9.2, run the following
1. sbin/start-dfs.sh
2. sbin/start-yarn.sh

Then, use jps to check if the following processes are on the master
```
NameNode
ResourceManager
SecondaryNameNode
```
and the following processes are on the slaves
```
DataNode
NodeManager
```

## Note:

Take care of scheduler v.s nodemanager memory/vcore setting!!! They must exist in separate yarn-site.xml!!!
```
Scheduler(@master) yarn.scheduler.maximum-allocation-mb 
Nodemanager(@slaves) yarn.nodemanager.resource.memory-mb
```

# Install Spark

# Configure Spark

1. Set environment variable SPARK_SSH_FOREGROUND to let it explicitly ask for the SSH password
2. Uncomment this line "192.168.0.107  ncku_intelligent_energy_7" in /etc/hosts

## Test one master + one slave hosted by the same machine
1. Make sure there is no file named "slaves" in /usr/local/spark/conf
2. In /usr/local/spark, run "sbin/start-all.sh"
3. In /usr/local/spark, run "MASTER=spark://192.168.0.107:7077 ./bin/run-example JavaWordCount /usr/local/spark/LICENSE"

## Test one master by one machine + two slaves by the other two machines
1. Put the following lines in /usr/local/spark/conf/slaves:
     192.168.0.107
     192.168.0.106
     192.168.0.108
2. (same as the previous section)
3. (same as the previous section)
4. In /usr/local/spark, run "MASTER=spark://192.168.0.107:7077 ./bin/run-example SparkPi 10000"
   (it will take about 96 seconds to finish)
5. In /usr/local/spark, run "MASTER=spark://192.168.0.107:7077 ./bin/run-example JavaWordCount [FilePath]"
   (FilePath can be a local file which should exist on all nodes or an hdfs URL)

## Test submitting to YARN
1. export HADOOP_CONF_DIR=/home/megaa/stuff/hadoop-2.9.2/etc/hadoop
2. Run
```
spark-submit --class org.apache.spark.examples.SparkPi --master yarn --deploy-mode cluster --driver-memory 4g --executor-memory 2g --executor-cores 1 --queue default examples/jars/spark-examples*.jar 10
spark-submit --master yarn --deploy-mode cluster --driver-memory 8g --executor-memory 1g --executor-cores 1 ~/test/spark/python/tit.py
```
### Note:
It is suggested that `--executor-memory` should be at least 1g

## YARN debug utility
In /home/megaa/stuff/hadoop-2.9.2,
```
bin/yarn jar share/hadoop/yarn/hadoop-yarn-applications-distributedshell-2.9.2.jar -jar share/hadoop/yarn/hadoop-yarn-applications-distributedshell-2.9.2.jar -shell_command python -shell_args -V
```
