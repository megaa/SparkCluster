# Install Java (OpenJDK 1.8)

1. sudo vi /etc/apt/sources.list, add the line:
     deb http://security.ubuntu.com/ubuntu bionic-security main universe
2. sudo add-apt-repository ppa:openjdk-r/ppa
3. sudo apt-get update
4. apt-cache search openjdk (check if openjdk-8-jdk is in the list)
5. sudo apt-get install openjdk-8-jdk

# Install Spark

# Configure Spark

1. Set environment variable SPARK_SSH_FOREGROUND to let it explicitly ask for the SSH password
2. Uncomment this line "192.168.0.107  ncku_intelligent_energy_7" in /etc/hosts

## Test one master + one slave hosted by the same machine
1. Make sure there is no file named "slaves" in /usr/local/spark/conf
2. In /usr/local/spark, run "sbin/start-all.sh"
3. In /usr/local/spark, run "MASTER=spark://192.168.0.107:7077 ./bin/run-example JavaWordCount ./LICENSE"

## Test one master by one machine + two slaves by the other two machines
1. Put the following lines in /usr/local/spark/conf/slaves:
     192.168.0.107
     192.168.0.106
     192.168.0.108
2 & 3: (same as the previous section)
