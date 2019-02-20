#Install Java (OpenJDK 1.8)

1. sudo vi /etc/apt/sources.list, add the line:
     deb http://security.ubuntu.com/ubuntu bionic-security main universe
2. sudo add-apt-repository ppa:openjdk-r/ppa
3. sudo apt-get update
4. apt-cache search openjdk (check if openjdk-8-jdk is in the list)
5. sudo apt-get install openjdk-8-jdk

#Install Spark

#Configure Spark

1. set environment variable SPARK_SSH_FOREGROUND to let it explicitly ask for the SSH password

