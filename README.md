# 1. Install Java (OpenJDK 1.8)

1. Edit `/etc/apt/sources.list`, add the line: `deb http://security.ubuntu.com/ubuntu bionic-security main universe`
2. `sudo add-apt-repository ppa:openjdk-r/ppa`
3. `sudo apt-get update`
4. `apt-cache search openjdk` (check if openjdk-8-jdk is in the list)
5. `sudo apt-get install openjdk-8-jdk`
6. Set `JAVA_HOME`: add the line `JAVA_HOME="/usr/lib/jvm/java-8-openjdk-amd64"` in `/etc/environment`

# 2. Install Hadoop

*Do the following for all machines unless explicitly stated*
1. Extract `hadoop-2.9.2.tar.gz` under `/home/megaa/stuff`
2. Put the following content in `/etc/profile.d/hadoop.sh`
```
HADOOP_PREFIX=/home/megaa/stuff/hadoop-2.9.2
export HADOOP_PREFIX
```
3. Edit `etc/hadoop/core-site.xml`:
```
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://10.8.0.103:9000</value>
    </property>
</configuration>
```
4. Edit `etc/hadoop/hdfs-site.xml`:
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
5. Edit `etc/hadoop/yarn-site.xml`:
```
<configuration>
    <property>
        <description>The hostname of the RM.</description>
        <name>yarn.resourcemanager.hostname</name>
        <value>10.8.0.103</value>
    </property>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
</configuration>
```
6. Edit `etc/hadoop/capacity-scheduler.xml` (only on the master machine):
```
  <property>
    <name>yarn.scheduler.capacity.maximum-am-resource-percent</name>
    <value>0.7</value>
    <description>
      Maximum percent of resources in the cluster which can be used to run
      application masters i.e. controls number of concurrent running
      applications.
    </description>
  </property>
```
7. Edit `etc/hadoop/hadoop-env.sh`:
```
export JAVA_HOME="/usr"
```
8. Setup passphraseless SSH (refer to http://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/SingleCluster.html), make sure you can ssh to each other without passphrase for all machines
9. Edit `etc/hadoop/slaves` (only on the master machine):
```
10.8.0.104
10.8.0.105
```
or
```
10.8.0.103
10.8.0.104
10.8.0.105
```
for 3 slaves configuration

10. On master machine (10.8.0.103), `mkdir -p /home/megaa/hadoop/data/namenode`
11. On slave machines, `mkdir -p /home/megaa/hadoop/data/datanode`
12. Format the HDFS volume: on master machine, `/home/megaa/stuff/hadoop-2.9.2`, run
```
bin/hdfs namenode -format
```

# 3. Run Hadoop
On master machine (10.8.0.103), `/home/megaa/stuff/hadoop-2.9.2`, run the following
1. `sbin/start-dfs.sh`
2. `sbin/start-yarn.sh`

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

Take care of scheduler v.s nodemanager memory/vcore setting!!! They must exist in separate `yarn-site.xml`!!!
```
Scheduler(@master) yarn.scheduler.maximum-allocation-mb 
Nodemanager(@slaves) yarn.nodemanager.resource.memory-mb
```
Also note the maximum percentage of resources setting of capacity scheduler!!! Refer to capacity-scheduler.xml setting in Step 2-6 above.

# 4. Install Spark

# 5. Configure Spark as standalone cluster

1. Set environment variable `SPARK_SSH_FOREGROUND` to let it explicitly ask for the SSH password
2. Make sure your hostname is valid (e.g not containing the `_` character), otherwise comment out lines like `10.8.0.107  ncku_intelligent_energy_7` in `/etc/hosts`

## 5.1 Test one master + one slave hosted by the same machine
1. Make sure there is no file named `slaves` in `/usr/local/spark/conf`
2. In `/usr/local/spark`, run `sbin/start-all.sh`
3. In `/usr/local/spark`, run `MASTER=spark://10.8.0.107:7077 ./bin/run-example JavaWordCount /usr/local/spark/LICENSE`

## 5.2 Test one master by one machine + two slaves by the other two machines
1. Put the following lines in `/usr/local/spark/conf/slaves`:
```
10.8.0.107
10.8.0.106
10.8.0.108
```
2. (same as 5.1.2)
3. (same as 5.1.3)
4. In `/usr/local/spark`, run `MASTER=spark://10.8.0.107:7077 ./bin/run-example SparkPi 10000`
   (it will take about 96 seconds to finish)
5. In `/usr/local/spark`, run `MASTER=spark://10.8.0.107:7077 ./bin/run-example JavaWordCount [FilePath]`
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
5. In `Spark conf/spark-defaults.conf`, add the line `spark.yarn.archive hdfs://10.8.0.103:9000/spark/spark-libs-2.4.0.jar`
6. When submitting, add the option `--files hdfs://10.8.0.103:9000/spark/pyspark.zip,hdfs://10.8.0.103:9000/spark/py4j-0.10.7-src.zip`

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

# 8. Running on K8s cluster
(note: Should use Spark 2.4.5+)
## Run Spark-bundled examples
1. Install minikube (Refer to https://kubernetes.io/docs/tasks/tools/install-minikube/)
2. Install kubectl (Refer to https://kubernetes.io/docs/tasks/tools/install-kubectl/)
3. Start a minikube cluster with appropriate CPUs and memory:<br/>
`minikube start --driver='docker' --kubernetes-version='v1.15.3' --cpus=4 --memory='8000mb'`
4. Set docker daemon to be using minikube's:<br/>
`eval $(minikube -p minikube docker-env)`
5. Build Spark's docker image:<br/>
In Spark root directory, `bin/docker-image-tool.sh -m -t testing build`
6. Put jar file to HDFS:<br/>
In Spark root directory, `cp examples/jars/spark-examples_2.11-2.4.5.jar /mnt/hdfs/spark/`<br/>
(HDFS mount should be done first)
7. Create service account for K8s RBAC permission control:<br/>
`kubectl create serviceaccount spark`<br/>
`kubectl create clusterrolebinding spark-role --clusterrole=edit --serviceaccount=default:spark --namespace=default`
8. Submit job
```
spark-submit --master k8s://https://HOST:PORT \
    --deploy-mode cluster \
    --name spark-pi \
    --class org.apache.spark.examples.SparkPi \
    --conf spark.executor.instances=2 \
    --conf spark.kubernetes.container.image=spark:testing \
    --conf spark.kubernetes.authenticate.driver.serviceAccountName=spark \
    hdfs://10.8.0.103:9000/spark/spark-examples_2.11-2.4.5.jar
```
where HOST:PORT can be obtained by `kubectl cluster-info`
## Run SparkStreaming
(1-4,7: same as above)<br/>
5. Build Spark's docker image:<br/>
Replace `/usr/local/spark/kubernetes/dockerfiles/spark/Dockerfile` first.<br/>
Copy `docker-imgtool.sh` to `/usr/local/spark/bin`.<br/>
Copy the 3 `*.jar` files to `/usr/local/spark/jars`.<br/>
Copy `jars_m/` to `/usr/local/spark`.<br/>
Copy `log4j.properties` to `/usr/local/spark/conf`.<br/>
In `/usr/local/spark`,<br/>
`bin/docker-imgtool.sh -m -t testing build`<br/>
6. Put jar file to HDFS:<br/>
`cp megaa-spark-test_2.11-1.0.jar /mnt/hdfs/spark/`<br/>
8. Submit job
```
spark-submit --master k8s://https://HOST:PORT \
    --deploy-mode cluster \
    --name spark-TEPCO \
    --class com.test.O_UDCStreaming \
    --conf spark.executor.instances=2 \
    --conf spark.kubernetes.container.image=spark:testing \
    --conf spark.kubernetes.authenticate.driver.serviceAccountName=spark \
    --conf spark.executor.extraJavaOptions=-Dlog4j.configuration=file:///opt/spark/conf/log4j.properties \
    hdfs://10.8.0.103:9000/spark/megaa-spark-test_2.11-1.0.jar
```

Known issues:
* After upgrading K8s to v1.17.2 (also tried v1.17.3, v1.18.2, v1.18.3), there will be the following error:
```
Exception in thread "main" io.fabric8.kubernetes.client.KubernetesClientException: Operation: [create]  for kind: [Pod]  with name: [null]  in namespace: [default]  failed.
	at io.fabric8.kubernetes.client.KubernetesClientException.launderThrowable(KubernetesClientException.java:64)
...
Caused by: java.net.SocketException: Broken pipe (Write failed)
	at java.net.SocketOutputStream.socketWrite0(Native Method)
...
```
which is caused by incompatible Spark version v.s K8s version. The solution is downgrade K8s to v1.15.3.
* With K8s v1.15.3, HDFS URL can't be accessed via hostname (ncku-intelligent-energy3) and IP address should be used instead. But with the original K8s version (I don't know it ><, maybe v1.17.0?), there is no such a problem. However, the executor log can now be controlled by the customized log4j configuration, which did not work in the original K8s version.
