# Table of Contents
- [Introduction](#introduction)
- [1. Build Hadoop Docker Image](#1-build-hadoop-docker-image)
  * [1.1 Hadoop Installation](#11-hadoop-installation)
  * [1.2 Setup Hadoop clusters](#12-setup-hadoop-clusters)
  * [1.3 Start Hadoop clusters](#13-start-hadoop-clusters)
  * [1.4 TODO](#14-todo)
  * [1.5 Test Hadoop grep example](#15-test-hadoop-grep-example)
- [2. Build Hive Docker Image](#2-build-hive-docker-image)
  * [2.1 Hive Installation](#21-hive-installation)
  * [2.2 Setup Hive clusters](#22-setup-hive-clusters)
  * [2.3 Start Hive clusters](#23-start-hive-clusters)
- [3. Build Spark Docker Image](#3-build-spark-docker-image)
  * [3.1 Spark Installation](#31-spark-installation)
  * [3.2 Start Spark clusters](#32-start-spark-clusters)
  * [3.3 Test Spark Standalone mode](#33-test-spark-standalone-mode)
  
# Introduction

The purpose of this repository is to setup virtual clusters on a single machine for learning big-data tools.

The clusters design is 1 master + 2 slaves. 

# 1. Build Hadoop Docker Image

## 1.1 Hadoop Installation

1. Pull ubuntu image from docker

   ```shell
   docker pull ubuntu
   ```

2.  Run ubuntu image and link local disk to ubuntu container

    ```shell
    docker run -it -v C:\hadoop\build:/root/build --name ubuntu ubuntu
    ```

3. Enter ubuntu container and update necessary components 

   ```shell
   # Update system
   apt-get update
   # Install vim
   apt-get install vim
   # Install ssh (need ssh to connect to slave)
   apt-get install ssh
   ```

4. Add the followings to .bashrc to start sshd service automatically

   ```shell
   vim ~/.bashrc
   # Add the following
   /etc/init.d/ssh start
   # save and quit
   :wq
   ```

5. Setup ssh to connect the workers without password

   ```shell
   # Enter the following and keep pressing enter
   ssh-keygen -t rsa
   # save to authorized keys
   cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
   ```

6. Test ssh connection

   ```shell
   ssh localhost
   # Example:
   ```

   ![image-20220803184446255](/images/ssh_connection.png)

7. Install java JDK

   ```shell
   # Use JDK 8 because Hive only support up to 8
   apt-get install openjdk-8-jdk
   # Setup environment variable
   vim ~/.bashrc
   # Add the followings
   export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64/
   export PATH=$PATH:$JAVA_HOME/bin
   # save and quit
   :wq
   # Make .bashrc effective immediately
   source ~/.bashrc
   ```

8.  Save this container as jdkinstalled image for later use

   ```shell
   # Commit the ubuntu container and rename it as ubuntu/jdkinstalled
   docker commit ubuntu ubuntu/jdkinstalled
   ```

9. Run ubuntu/jdkinstalled image and link local disk to  container

   ```shell
   docker run -it -v C:\hadoop\build:/root/build --name ubuntu-jdkinstalled ubuntu/jdkinstalled
   ```

10. Download Hadoop installation package from Apache (I use Hadoop-3.3.3) and put it to C:\hadoop\build

    https://dlcdn.apache.org/hadoop/common/stable/

11. Enter jdkinstalled container

    ```shell
    docker exec -it ubuntu-jdkinstalled bash
    # switch to build and look for the installation package
    cd /root/build
    # extract the package to /usr/local
    tar -zxvf hadoop-3.3.3.tar.gz -C /usr/local
    ```

12. Switch to Hadoop folder

    ```shell
    # switch to hadoop folder
    cd /usr/local/hadoop-3.3.3/
    # Test if hadoop is running properly
    ./bin/hadoop version
    ```

13. Setup Hadoop environment variables

    ```shell
    vim /etc/profile
    # Add the followings
    export HADOOP_HOME=/usr/local/hadoop-3.3.3
    export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
    # save and quit
    :wq
    
    vim ~/.bashrc
    # Add the following
    source /etc/profile
    # save and quit
    :wq
    ```

## 1.2 Setup Hadoop clusters

1. Switch to Hadoop configuration path

   ```shell
   cd /usr/local/hadoop-3.3.3/etc/hadoop/
   ```

2. Setup hadoop-env.sh

   ```shell
   vim hadoop-env.sh
   
   # Add the followings
   export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64/
   export HDFS_NAMENODE_USER="root"
   export HDFS_DATANODE_USER="root"
   export HDFS_SECONDARYNAMENODE_USER="root"
   export YARN_RESOURCEMANAGER_USER="root"
   export YARN_NODEMANAGER_USER="root"
   
   # save and quit
   :wq
   ```

3. setup core-site.xml

   ```xml
   <configuration>
       <property>
           <name>hadoop.tmp.dir</name>
           <value>file:/usr/local/hadoop/data</value>
           <description>Abase for other temporary directories.</description>
       </property>
       <property>
           <name>fs.defaultFS</name>
           <value>hdfs://master:9000</value>
       </property>
       <property>
           <name> hadoop.hyyp.staticuser.user</name>
           <value>root</value>
       </property>
       <property>
           <name>hadoop.proxyuser.root.hosts</name>
           <value>*</value>
       </property>
       <property>
           <name>hadoop.proxyuser.root.groups</name>
           <value>*</value>
       </property>
       <property>
           <name>fs.trash.interval</name>
           <value>1440</value>
       </property>
   </configuration>
   ```

4.  Setup hdfs-site.xml

   ```xml
   <configuration>
       <property>
           <name>dfs.namenode.name.dir</name>
           <value>file:/usr/local/hadoop/namenode_dir</value>
       </property>
       <property>
           <name>dfs.datanode.data.dir</name>
           <value>file:/usr/local/hadoop/datanode_dir</value>
       </property>
       <property>
           <name>dfs.replication</name>
           <value>3</value>
       </property>
       <property>
           <name>dfs.namenode.secondary.http-address</name>
           <value>slave01:9868</value>
       </property>
   </configuration> 
   ```

5. Setup madpred-site.xml

   ```xml
   <configuration>
       <property>
           <name>mapreduce.framework.name</name>
           <value>yarn</value>
       </property>
       <property>
           <name>mapreduce.jobhistory.address</name>
           <value>master:10020</value>
       </property>
       <property>
           <name>mapreduce.jobhistory.webapp.address</name>
           <value>master:19888</value>
       </property>
       <property>
           <name>yarn.app.mapreduce.am.env</name>
           <value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
       </property>
       <property>
           <name>mapreduce.map.env</name>
           <value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
       </property>
       <property>
           <name>mapreduce.reduce.env</name>
           <value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
       </property>
   </configuration> 
   ```

6. Setup yarn-site.xml

   ```xml
   <configuration>
       <!-- Site specific YARN configuration properties -->
       <property>
           <name>yarn.nodemanager.aux-services</name>
           <value>mapreduce_shuffle</value>
       </property>
       <property>
           <name>yarn.resourcemanager.hostname</name>
           <value>master</value>
       </property>
       <property>
           <name>yarn.nodemanager.pmem-check-enabled</name>
           <value>false</value>
       </property>
       <property>
           <name>yarn.nodemanager.vmem-check-enabled</name>
           <value>false</value>
       </property>
       <property>
           <name>yarn.log-aggregation-enable</name>
           <value>true</value>
       </property>
       <property>
           <name>yarn.log.server.url</name>
           <value>http://master:19888/jobhistory/logs</value>
       </property>
       <property>
           <name>yarn.log-aggregation.retain-seconds</name>
           <value>604800</value>
       </property>
    </configuration>
   ```

7. Save this container as hadoop-installed image for later use

   ```shell
   docker commit ubuntu-jdkinstalled ubuntu/hadoopinstalled
   ```

8. Open 3 terminals and enter the following cmd separately

   Terminal 1 (master):

   ```shell
   # 8088 is the default WEB UI port for YARN
   # 9870 is the default WEB UI port for HDFS
   # 9864 is for inspecting file content inside HDFS WEB UI
   docker run -it -h master  -p 8088:8088 -p 9870:9870 -p 9864:9864 --name master ubuntu/hadoopinstalled 
   ```

   Terminal 2 (slave01):

   ```shell
   docker run -it -h slave01 --name slave01 ubuntu/hadoopinstalled
   ```

   Terminal 3 (slave02):

   ```shell
   docker run -it -h slave02 --name slave02 ubuntu/hadoopinstalled
   ```

9. For every terminal, inspect the IP for each container enter the ip to every container.

   ```shell
   # Enter the hosts
   vim /etc/hosts
   # Example
   ```

   ![image-20220803205511246](/images/ip_hosts_example.png)

10. Open the workers file in master container

    ```shell
    vim /usr/local/hadoop-3.3.3/etc/hadoop/workers
    # Change localhost to the followings (I put master in workers to add datanode and NodeManager to master as well)
    master
    slave01
    slave02
    # Example
    ```

    ![image-20220803205936807](/images/workers.png)

11. Test to see if master can enter slave01 and slave02 without password 

    ```shell
    # Need to enter yes for first time connection
    ssh slave01
    # Example
    ```

    ![image-20220803210337631](/images/ssh_slave01.jpg)

    ![image-20220803211116028](/images/ssh_slave02.jpg)

## 1.3 Start Hadoop clusters

1. Need to format HDFS system if starting for the first time

   ```
   hdfs namenode -format
   ```

2. Start HDFS clusters + YARN clusters

   ```
   start-all.sh
   ```

3. Enter jps to check if the corresponding java processes have started successfully

![image-20220803211621660](/images/master_jps.png)

![image-20220803211658947](/images/slave01_jps.png)

![image-20220803211734165](/images/slave02_jps.png)

3. To properly use the WEB UI, need to modify hosts file.

   For windows:

   ```txt
   Open C:\Windows\System32\drivers\etc\hosts
   
   # Add the followings
   127.0.0.1 master
   127.0.0.1 slave01
   127.0.0.1 slave02
   
   # Save the changes
   ```

4. Open YARN WEB UI

   http://localhost:8088/

5. Open HDFS WEB UI

   http://localhost:9870/

## 1.4 TODO

1. HDFS WEB UI upload has problem

   ![image-20220803212604079](/images/hdfs_upload.png)

2. HDFS WEB UI make folders has problem

   ![image-20220803212736367](/images/hdfs_mkdir.png)

## 1.5 Test Hadoop grep example

```shell
# Make input folder in HDFS
hdfs fs -mkdir -p /user/hadoop/input

# Upload local hadoop xml file to HDFS
hdfs fs -put ./etc/hadoop/*.xml /user/hadoop/input

# Use ls to check if files are uploaded to HDFS properly
hdfs fs -ls /user/hadoop/input

# Run grep example. Use all files in HDFS input folder and filter dfs[a-z.]+ word counts to output folder
hadoop jar ./share/hadoop/mapreduce/hadoop-mapreduce-examples-*.jar grep /user/hadoop/input output 'dfs[a-z.]+'

# Check results
./bin/hdfs dfs -cat output/*
```



# 2. Build Hive Docker Image

## 2.1 Hive Installation

1. Run Hadoop image

   ```shell
   docker run -it -h master -p 8088:8088 -p 9870:9870 -p 9864:9864 --name master ubuntu/hadoopinstalled
   ```

2. Download Apache Hive installation package

   https://dlcdn.apache.org/hive/hive-3.1.3/

3. Copy Hive installation package to master container

   ```shell
   docker cp apache-hive-3.1.3-bin.tar.gz master:/usr/local/
   ```

4. Install sudo

   ```shell
   apt-get install sudo -y
   ```

5. Install net-tools

   ```
   sudo apt install net-tools
   ```

6. Install mysql-server for meta-store

   ```shell
   # Install mysql-server 
   sudo apt install mysql-server -y
   
   # Grant permission
   usermod -d /var/lib/mysql mysql
   
   # Check mysql version
   mysql --version
   ```

   ![image-20220803214235469](/images/mysql_version.png)

6. Extract Hive package

   ```shell
   # Enter master container
   docker exec -it master bash
   
   # Change to package location
   cd /usr/local/
   
   # Extract Hive package
   tar -zxvf apache-hive-3.1.3-bin.tar.gz
   
   # Rename to hive
   mv apache-hive-3.1.3-bin hive
   
   # Remove Hive installation package
   rm apache-hive-3.1.3-bin.tar.
   
   # Grant root to hive folder
   sudo chown -R root:root hive
   ```

7. Setup Hive environment variables

   ```shell
   # Open ./bashrc
   vim ~/.bashrc
   
   # Enter the followings:
   export HIVE_HOME=/usr/local/hive
   export PATH=$PATH:$HIVE_HOME/bin
   export HADOOP_HOME=/usr/local/hadoop-3.3.3
   
   # save and quit
   :wq
   
   # make .bashrc effective immediately
   source ~/.bashrc
   ```

## 2.2 Setup Hive clusters

1. Change to Hive configuration folder

   ```shell
   cd /usr/local/hive/conf
   ```

2. Use hive-default.xml template

   ```
   mv hive-default.xml.template hive-default.xml
   ```

3. Setup hive-site.xml

   ```xml
   <?xml version="1.0" encoding="UTF-8" standalone="no"?>
   <?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
   <configuration>
       <property>
           <name>javax.jdo.option.ConnectionURL</name>
           <value>jdbc:mysql://localhost:3306/hive?createDatabaseIfNotExist=true&amp;useSSL=false&amp;useUnicode=true&amp;characterEncoding=UTF-8&amp;allowPublicKeyRetrieval=true</value>
           <description>JDBC connect string for a JDBC metastore</description>
       </property>
       <property>
           <name>javax.jdo.option.ConnectionDriverName</name>
           <value>com.mysql.cj.jdbc.Driver</value>
           <description>Driver class name for a JDBC metastore</description>
       </property>
       <property>
           <name>javax.jdo.option.ConnectionUserName</name>
           <value>hive</value>
           <description>username to use against metastore database</description>
       </property>
       <property>
           <name>javax.jdo.option.ConnectionPassword</name>
           <value>hive</value>
           <description>password to use against metastore database</description>
       </property>
       <property>
           <name>hive.server2.thrift.bind.host</name>
           <value>master</value>
           <description> H2S bind to host </description>
       </property>
       <property>
           <name>hive.metastore.uris</name>
           <value>thrift://master:9083</value>
           <description> metastore url address </description>
       </property>
       <property>
           <name>hive.metastore.event.db.notification.api.auth</name>
           <value>false</value>
           <description> close metadata storage authorization </description>
       </property>
   </configuration>
   ```

4. Start mysql database

   ```shell
   # Enter mysql as root user without password
   mysql -u root  -p
   ```

5. Enter the following commands

   ```mysql
   create database hive;
   
   create user hive@localhost identified by 'hive';
   
   GRANT ALL PRIVILEGES ON *.* TO hive@localhost with grant option;
   
   FLUSH PRIVILEGES;
   ```

6. Enter ctrl + d to quit mysql

7. Remove jar from Hive as it conflicts with Hadoop

   ```shell
   rm /usr/local/hive/lib/log4j-slf4j-impl-2.17.1.jar
   ```

8. Download mysql connector driver (mysql version is 8.0.29)

   https://dev.mysql.com/downloads/connector/j/

9. Copy to master container and extract

   ```shell
   # Copy to master container
   docker cp mysql-connector-java_8.0.29-1ubuntu21.10_all.deb master:/usr/local/
   
   # Switch to /usr/local
   cd /usr/local
   
   # Extract package
   dpkg -i mysql-connector-java_8.0.29-1ubuntu21.10_all.deb
   
   # Copy jar file to Hive library
   cp /usr/share/java/mysql-connector-java-8.0.29.jar /usr/local/hive/lib/
   ```

10. Commit the container to a new image

    ```shell
    docker commit master ubuntu/hiveinstalled
    ```

    

## 2.3 Start Hive clusters

1. Open 3 terminals and enter the following cmd separately

   Terminal 1 (master):

   ```shell
   docker run -it -h master  -p 8088:8088 -p 9870:9870 -p 9864:9864 -p 10000:10000 -p 8080:8080 --name master ubuntu/hiveinstalled
   ```

   Terminal 2 (slave01):

   ```shell
   docker run -it -h slave01 --name slave01 ubuntu/hadoopinstalled
   ```

   Terminal 3 (slave02):

   ```shell
   docker run -it -h slave02 --name slave02 ubuntu/hadoopinstalled
   ```

2. For every terminal, inspect the IP for each container enter the ip to every container.

   ```shell
   # Enter the hosts
   vim /etc/hosts
   # Example
   ```

   ![image-20220803205511246](/images/ip_hosts_example.png)

3. Open the workers file in master container

   ```shell
   vim /usr/local/hadoop-3.3.3/etc/hadoop/workers
   # Change localhost to the followings (I put master in workers to add datanode and NodeManager to master as well)
   master
   slave01
   slave02
   # Example
   ```

   ![image-20220803205936807](/images/workers.png)

4. Enter master container and start mysql

   ```shell
   # Enter master container
   docker exec -it master bash
   
   # Start mysql
   service mysql start
   ```

5. Start HDFS service

   ```shell
   # Start HDFS
   start-dfs.sh
   
   # Create the following folders
   hadoop fs -mkdir /tmp
   hadoop fs -mkdir -p /user/hive/warehouse
   
   # Grant read write permission
   hadoop fs -chmod g+w /tmp
   hadoop fs -chmod g+w /user/hive/warehouse
   ```

6. Start metastore service and hiveserver2

   ```shell
   # Start metastore
   nohup hive --service metastore > nohup.out &
   # Use net-tools to check if metastore has started properly
   netstat -nlpt |grep 9083
   
   # Start hiveserver2
   nohup hiveserver2 > nohup2.out &
   # Use net-tools to check if hiveserver2 has started properly
   netstat -nlpt|grep 10000
   
   ```

   Should see the followings if metastore and hiveserver2 have started properly

   ![image-20220804092739739](/images/hive_metastore_port.png)

   ![image-20220804092831807](/images/hiveserver2.png)

   

7. Test hiveserver2 with beeline client

   ```shell
   # Enter beeline in terminal
   beeline
   
   # Enter the following to connect to metastore
   !connect jdbc:hive2://master:10000
   
   # Enter root as username; no need to enter password
   ```

   ![image-20220804093303295](/images/connect_metastore.png)

# 3. Build Spark Docker Image

## 3.1 Spark Installation

1. Download Spark from Apache Spark (I use spark-3.3.0 for this setup)

   https://www.apache.org/dyn/closer.lua/spark/spark-3.3.0/spark-3.3.0-bin-hadoop3.tgz

2. Install Spark installation package in master container

   ```shell
   # Copy Spark installation package from local machine to master container
   docker cp spark-3.3.0-bin-hadoop3.tgz master:/usr/local/
   
   # Enter master container
   docker exec -it master bash
   
   # Change to /usr/local
   cd /usr/local
   
   # Extract package
   sudo tar -zxf spark-3.3.0-bin-hadoop3.tgz -C /usr/local/
   
   # rm package
   rm spark-3.3.0-bin-hadoop3.tgz
   
   # Rename Spark folder
   sudo mv ./spark-3.3.0-bin-hadoop3/ ./spark
   ```

3. Download Anaconda3 Linux distribution

   https://www.anaconda.com/products/distribution#Downloads

4. Install Anaconda3 in master container (I use Anaconda3-2022.05 for this setup)

   ```shell
   # Copy Anaconda3 installation package from local machine to master container
   docker cp Anaconda3-2022.05-Linux-x86_64.sh master:/usr/local/
   
   # Enter master container
   docker exec -it master bash
   
   # Change to /usr/local
   cd /usr/local
   
   # Install Anaconda3 package
   sh ./ Anaconda3-2022.05-Linux-x86_64.sh
   # Type yes
   
   # Enter installation path
   /usr/local/anaconda3
   # Type yes
   
   # Remove Anaconda3 package
   rm Anaconda3-2022.05-Linux-x86_64.sh
   ```

5. Setup python virtual environment named "pyspark"

   ```shell
   conda create -n pyspark python=3.8
   # Type y
   
   # Activate pyspark virtual env
   conda activate pyspark
   ```

   ![image-20220804095615061](/images/activate_pyspark_env.png)

6. Install necessary python packages

   ```shell
   conda install pyspark
   conda install pyhive
   ```

7. Enter the followings into /etc/profile

   ```shell
   # Open /etc/profile
   vim /etc/profile
   
   # Make sure the followings are in the file
   export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64/
   
   export HADOOP_HOME=/usr/local/hadoop-3.3.3
   export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
   export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
   
   export SPARK_HOME=/usr/local/spark
   export PYSPARK_PYTHON=/usr/local/anaconda3/envs/pyspark/bin/python
   
   # Save and quit
   :wq
   ```

8. Configure workers file

   ```shell
   # Switch to spark configuration folder
   cd /usr/local/spark/conf
   
   # cp workers template
   cp workers.template workers
   
   # Open workers file
   vim workers
   
   # Change localhost to the followings:
   master
   slave01
   slave02
   
   # Save and quit
   :wq
   ```

9. Configure spark-env.sh file

   ```shell
   # Open spark-env.sh file
   vim spark-env.sh
   
   # Enter the followings
   HADOOP_CONF_DIR=/usr/local/hadoop-3.3.3/etc/hadoop
   YARN_CONF_DIR=/usr/local/hadoop-3.3.3/etc/hadoop
   export SPARK_MASTER_HOST=master
   export SPARK_MASTER_PORT=7077
   SPARK_MASTER_WEBUI_PORT=8080
   SPARK_WORKER_PORT=7078
   SPARK_WORKER_WEBUI_PORT=8081
   SPARK_HISTORY_OPTS="-Dspark.history.fs.logDirectory=hdfs://master:9000/sparklog/ -Dspark.history.fs.cleaner.enabled=true -Dspark.history.ui.port=18080"
   
   # Save and quit
   :wq
   ```

10. Create sparklog folder in HDFS

    ```shell
    hadoop fs -mkdir /sparklog
    hadoop fs -chmod 777 /sparklog
    ```

11. Configure spark-defaults.conf file

    ```shell
    # Copy spark-defaults.conf template
    cp spark-defaults.conf.template spark-defaults.conf
    
    # Open the file
    vim spark-defaults.conf
    
    # Enter the followings
    spark.eventLog.enabled true
    spark.eventLog.dir hdfs://master:9000/sparklog/
    spark.eventLog.compress true
    
    # Save and quit
    :wq
    ```

12. Configure log4j2.properties file

    ```shell
    # Copy log4j2.properties template
    cp log4j2.properties.template log4j2.properties
    
    # Open the file
    vim log4j2.properties
    
    # Change rootLogger.level to warn
    rootLogger.level = warn
    
    # Save and quit
    :wq
    ```

13. Commit the container to a new image

    ```
    docker commit master ubuntu/sparkinstalled
    ```

## 3.2 Start Spark clusters

1. Open 3 terminals and enter the following cmd separately

   Terminal 1 (master):

   ```shell
   docker run -it -h master  -p 8088:8088 -p 9870:9870 -p 9864:9864 -p 10000:10000 -p 8080:8080 -p 18080:18080 -p 4040:4040 -p 19888:19888 --name master ubuntu/sparkinstalled
   ```

2. Terminal 2 (slave01)

   ```shell
   docker run -it -h slave01 --name slave01 ubuntu/sparkinstalled
   ```

3. Terminal 3 (slave02)

   ```shell
   docker run -it -h slave02 --name slave02 ubuntu/sparkinstalled
   ```

4. Enter master container and start spark clusters

   ```shell
   # Enter master container
   docker exec -it master bash
   
   # Make sure to Hadoop clusters first
   start-all.sh
   
   # Follow 2.3 instruction to start Hive clusters (For Spark on Hive)
   
   # Switch to spark sbin folder
   cd /usr/local/spark/sbin
   
   # Start spark clusters
   ./start-all.sh
   
   # Start history server
   ./start-history-server.sh
   
   # Start mr-jobhistory server
   cd /usr/local/hadoop-3.3.3/sbin
   mr-jobhistory-daemon.sh start historyserver
   ```

## 3.3 Test Spark Standalone mode

```shell
cd /usr/local/spark
# Start pyspark in Standalone mode
bin/pyspark --master spark://master:7077
```

```python
# Type the following
sc.parallelize([1,2,3,4,5]).map(lambda x: x + 1).collect()
```

![image-20220804103924331](/images/spark_standalone.png)
