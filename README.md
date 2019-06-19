# 1. Install Java (OpenJDK 1.8)

1. sudo vi /etc/apt/sources.list, add the line: `deb http://security.ubuntu.com/ubuntu bionic-security main universe`
2. sudo add-apt-repository ppa:openjdk-r/ppa
3. sudo apt-get update
4. apt-cache search openjdk (check if openjdk-8-jdk is in the list)
5. sudo apt-get install openjdk-8-jdk
6. Set JAVA_HOME: add the line `JAVA_HOME="/usr/lib/jvm/java-8-openjdk-amd64"` in /etc/environment

# 2. Install Hadoop

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
or
```
192.168.0.103
192.168.0.104
192.168.0.105
```
for 3 slaves configuration

9. On master machine (192.168.0.103), mkdir -p /home/megaa/hadoop/data/namenode
10. On slave machines, mkdir -p /home/megaa/hadoop/data/datanode
11. Format the HDFS volume: on master machine, /home/megaa/stuff/hadoop-2.9.2, run
```
bin/hdfs namenode -format
```

# 3. Run Hadoop
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

# 4. Install Spark

# 5. Configure Spark as standalone cluster

1. Set environment variable `SPARK_SSH_FOREGROUND` to let it explicitly ask for the SSH password
2. Make sure your hostname is valid (e.g not containing the `_` character), otherwise comment out lines like `192.168.0.107  ncku_intelligent_energy_7` in `/etc/hosts`

## 5.1 Test one master + one slave hosted by the same machine
1. Make sure there is no file named `slaves` in `/usr/local/spark/conf`
2. In `/usr/local/spark`, run `sbin/start-all.sh`
3. In `/usr/local/spark`, run `MASTER=spark://192.168.0.107:7077 ./bin/run-example JavaWordCount /usr/local/spark/LICENSE`

## 5.2 Test one master by one machine + two slaves by the other two machines
1. Put the following lines in `/usr/local/spark/conf/slaves`:
```
192.168.0.107
192.168.0.106
192.168.0.108
```
2. (same as 5.1.2)
3. (same as 5.1.3)
4. In `/usr/local/spark`, run `MASTER=spark://192.168.0.107:7077 ./bin/run-example SparkPi 10000`
   (it will take about 96 seconds to finish)
5. In `/usr/local/spark`, run `MASTER=spark://192.168.0.107:7077 ./bin/run-example JavaWordCount [FilePath]`
   (`FilePath` can be a local file which should exist on all nodes or an HDFS URL)

# 6. Submitting to YARN
1. `export HADOOP_CONF_DIR=/home/megaa/stuff/hadoop-2.9.2/etc/hadoop`
2. Run
```
spark-submit --class org.apache.spark.examples.SparkPi --master yarn --deploy-mode cluster --driver-memory 4g --executor-memory 2g --executor-cores 1 --queue default examples/jars/spark-examples*.jar 10
spark-submit --master yarn --deploy-mode cluster --driver-memory 8g --executor-memory 1g --executor-cores 1 ~/test/spark/python/tit.py
```
## Note:
1. It is suggested that `--executor-memory` should be at least 1g
2. Add `--num-executors` to specify the number of executors to be allocated

## YARN debug utility
In `/home/megaa/stuff/hadoop-2.9.2`,
```
bin/yarn jar share/hadoop/yarn/hadoop-yarn-applications-distributedshell-2.9.2.jar -jar share/hadoop/yarn/hadoop-yarn-applications-distributedshell-2.9.2.jar -shell_command python -shell_args -V
```

## Prevent Spark from uploading resource files for each submission
1. Create the spark jar files: `jar cv0f spark-libs-2.4.0.jar -C /usr/local/spark/jars/ .`
2. Put it to HDFS: `hdfs dfs -put spark-libs-2.4.0.jar /spark`
3. Set the replication count: `hdfs dfs -setrep -w 2 /spark/spark-libs-2.4.0.jar`
4. Do 2 & 3 for `py4j-0.10.7-src.zip` and `pyspark.zip`
5. In `Spark conf/spark-defaults.conf`, add the line `spark.yarn.archive hdfs://192.168.0.103:9000/spark/spark-libs-2.4.0.jar`
6. When submitting, add the option `--files hdfs://192.168.0.103:9000/spark/pyspark.zip,hdfs://192.168.0.103:9000/spark/py4j-0.10.7-src.zip`

# 7. Miscellaneous Items

## Mount HDFS as a local filesystem (via Fuse)
1. Download Hadoop source file and extract it (`hadoop-2.9.2-src.tar.gz`)
2. Edit `hadoop-2.9.2-src/dev-support/docker/Dockerfile`, comment out the lines related to Node.js (#141~#145)
3. Edit `hadoop-2.9.2-src/start-build-env.sh`, change `${USER_NAME}` to `root` in line #49
4. Install docker and start the service
5. In `hadoop-2.9.2-src/`, run `sudo ./start-build-env.sh`
6. In the docker environment, unlink `/usr/bin/mvn` and create another one linking to `/opt/maven/bin/mvn`
7. Build the Hadoop source by `mvn clean package -Pdist,native -DskipTests`
8. Copy `hadoop-2.9.2-src/hadoop-hdfs-project/hadoop-hdfs-native-client/target/main/native/fuse-dfs/fuse_dfs` and `hadoop-2.9.2-src/hadoop-hdfs-project/hadoop-hdfs-native-client/src/main/native/fuse-dfs/fuse_dfs_wrapper.sh` to the `sbin/` of your Hadoop installation
9. Edit `fuse_dfs_wrapper.sh`, change the following lines:
```
export FUSEDFS_PATH="$HADOOP_PREFIX/sbin"
export LIBHDFS_PATH="$HADOOP_PREFIX/lib/native"
done < <(find "$HADOOP_PREFIX/share/hadoop" -name "*.jar" -print0)
```
10. Mount HDFS by `$HADOOP_PREFIX/sbin/fuse_dfs_wrapper.sh dfs://host:port /mnt/hdfs` as root (be sure to set `$HADOOP_PREFIX` and `$JAVA_HOME` first)
