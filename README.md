# KafkaHadoopSinkExample

This is a simple example of writing to Hadoop HDFS file system from Kafka using Confluent sink.

**Prereqs**

This assumes you already possess some know-how in AWS (SSH into boxes, create or use an appropriate VPC,create or use an appropriate security group) as well as run some basic linux commands. You probably would not be here if that was foreign.

You can use whatever boxes and O/S you want.

In this example, we will use a CentOS box in AWS for Kafka and Hadoop using the convenient Confluent CLI for illustration.

**Install Java, Confluent, Docker**

Install Java

```
sudo yum install java-1.8.0-openjdk
```

Install CP

```
curl -O http://packages.confluent.io/archive/5.1/confluent-5.1.1-2.11.tar.gz
tar -xvf confluent-5.1.1-2.11.tar.gz
```

Install Docker on CentOS

```
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo  
sudo yum install docker-ce docker-ce-cli containerd.io
sudo systemctl start docker
sudo docker run hello-world
```

Install Docker on Ubuntu (pasted here for convenience in case you chose Ubuntu instead)

```
sudo apt-get update
sudo apt-get install apt-transport-https ca-certificates curl gnupg-agent software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo apt-key fingerprint 0EBFCD88
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt-get install docker-ce docker-ce-cli containerd.io
sudo docker run hello-world
```

**Run Hadoop**

Note, we are running this from the same localhost machine as Kafka so we will expose port 9000.

```
sudo docker pull sequenceiq/hadoop-docker:2.7.1
sudo docker images
sudo docker run -p 127.0.0.1:9000:9000 --name scottsHadoop -it sequenceiq/hadoop-docker:2.7.1 /etc/bootstrap.sh -bash
```

You may want to expose other ports like the Hadoop UI available in this image on port 50070. If you ran this on your local Mac instead of AWS in this example, you will find the GUI convenient for browsing the HDFS filesystem.

```
docker run -p 127.0.0.1:9000:9000 -p 127.0.0.1:50070:50070 --name scottsHadoop -it sequenceiq/hadoop-docker:2.7.1 /etc/bootstrap.sh -bash
```

In the bash terminal, do a quick sanity check.

```
bash-4.1# jps
```

**Set up your HDFS Kafka sink**

Back in CP terminal, install Hadoop connector

```
./confluent-hub install confluentinc/kafka-connect-hdfs:latest
sudo vi ../etc/kafka-connect-hdfs/localhdfstest.properties
```

Insert something that resembles the following. Notice we have 1 topic at this time listed, test_hdfs. Later, we will show you a regex example.

```
name=hdfs-sink
connector.class=io.confluent.connect.hdfs.HdfsSinkConnector
tasks.max=1
topics=test_hdfs
hdfs.url=hdfs://localhost:9000
flush.size=3
hadoop.conf.dir=/usr/local/hadoop-2.7.1/etc/hadoop/
partitioner.class=io.confluent.connect.hdfs.partitioner.FieldPartitioner
partition.field.name=f1
rotate.interval.ms=120000
hadoop.home=/usr/local/hadoop-2.7.1/share/hadoop/common/
logs.dir=/opt/hadoop-2.7.1/test_hdfs/logs
schema.compatibility=BACKWARD
```

Start your connector and do a sanity check. Check logs too, don't simply rely on status.

```
./confluent start connect
./confluent load hdfs-sink -d ../etc/kafka-connect-hdfs/localhdfstest.properties
curl -X GET localhost:8083/connectors
curl -X GET localhost:8083/connectors/hdfs-sink/status
```

Note in this simple example, if you get into an issue with permissions at the local HDFS level, it may be easiest to unlock the permissions unless you want to debug that more. In the hadoop bash terminal:

```
cd $HADOOP_PREFIX
cd bin/
./hdfs dfs -chmod 777  /
```

Restart connector

```
curl -X POST localhost:8083/connectors/hdfs-sink/restart
curl -X GET localhost:8083/connectors/hdfs-sink/status
```

**Let's produce some sample data**

This is just sample data. Notice the partition field name f1 that we used in the HDFS sink configuration above.

```
./kafka-avro-console-producer --broker-list localhost:9092 --topic test_hdfs --property value.schema='{"type":"record","name":"myrecord","fields":[{"name":"f1","type":"string"}]}'
{"f1": "value1"}
{"f1": "value2"}
{"f1": "value3"}
{"f1": "value4"}
{"f1": "value5"}

./kafka-avro-console-consumer --topic test_hdfs --bootstrap-server localhost:9092
```

The connector output resembles something along the lines of:

```
[2019-02-23 22:11:14,755] INFO HDFS connector does not commit consumer offsets to Kafka. Upon startup, HDFS Connector restores offsets from filenames in HDFS. In the absence of files in HDFS, the connector will attempt to find offsets for its consumer group in the '__consumer_offsets' topic. If offsets are not found, the consumer will rely on the reset policy specified in the 'consumer.auto.offset.reset' property to start exporting data to HDFS. (io.confluent.connect.hdfs.HdfsSinkTask:98)
[2019-02-23 22:11:14,963] INFO Opening record writer for: hdfs://localhost:9000/topics//+tmp/test_hdfs/f1=value1/2818f261-c4b3-42bf-b5e7-e2022f6aabcf_tmp.avro (io.confluent.connect.hdfs.avro.AvroRecordWriterProvider:65)
[2019-02-23 22:11:15,760] INFO Successfully acquired lease for hdfs://localhost:9000//opt/hadoop-2.7.1/test_hdfs/logs/test_hdfs/0/log (io.confluent.connect.hdfs.wal.FSWAL:75)
[2019-02-23 22:11:15,808] INFO Committed hdfs://localhost:9000/topics/test_hdfs/f1=value3/test_hdfs+0+0000000002+0000000002.avro for test_hdfs-0 (io.confluent.connect.hdfs.TopicPartitionWriter:771)
```

In the hadoop shell, let's take a look at the HDFS filesystem.

```
bash-4.1# ./hdfs dfs -ls /topics/test_hdfs
drwxr-xr-x   - centos supergroup          0 2019-02-23 17:11 /topics/test_hdfs/f1=value1
drwxr-xr-x   - centos supergroup          0 2019-02-23 17:11 /topics/test_hdfs/f1=value2
drwxr-xr-x   - centos supergroup          0 2019-02-23 17:11 /topics/test_hdfs/f1=value3
drwxr-xr-x   - centos supergroup          0 2019-02-23 17:14 /topics/test_hdfs/f1=value4
drwxr-xr-x   - centos supergroup          0 2019-02-23 17:14 /topics/test_hdfs/f1=value5

bash-4.1# ./hdfs dfs -ls /topics/test_hdfs/f1=value1
-rw-r--r--   3 centos supergroup        199 2019-02-23 17:11 /topics/test_hdfs/f1=value1/test_hdfs+0+0000000000+0000000000.avro
```

Schema Registry for your topic would have entry like so:

```
{
 "fields": [
   {
     "name": "f1",
     "type": "string"
   }
 ],
 "name": "myrecord",
 "type": "record"
}
```

(Optional) Let's evolve the schema a bit and produce a little more data.

```
./kafka-avro-console-producer --broker-list localhost:9092 --topic test_hdfs --property value.schema='{"type":"record","name":"myrecord","fields":[{"name":"f1","type":"string"},{"name":"f2","type":"string","default":" "}]}'
{"f1": "value6","f2": "value7"}
{"f1": "value8","f2": "value9"}
{"f1": "value10","f2": "value11"}
```

```
bash-4.1# ./hdfs dfs -ls /topics/test_hdfs          
drwxr-xr-x   - centos supergroup          0 2019-02-23 17:11 /topics/test_hdfs/f1=value1
drwxr-xr-x   - centos supergroup          0 2019-02-23 18:18 /topics/test_hdfs/f1=value10
drwxr-xr-x   - centos supergroup          0 2019-02-23 17:11 /topics/test_hdfs/f1=value2
drwxr-xr-x   - centos supergroup          0 2019-02-23 17:11 /topics/test_hdfs/f1=value3
drwxr-xr-x   - centos supergroup          0 2019-02-23 17:14 /topics/test_hdfs/f1=value4
drwxr-xr-x   - centos supergroup          0 2019-02-23 17:14 /topics/test_hdfs/f1=value5
drwxr-xr-x   - centos supergroup          0 2019-02-23 18:18 /topics/test_hdfs/f1=value6
drwxr-xr-x   - centos supergroup          0 2019-02-23 18:18 /topics/test_hdfs/f1=value8

bash-4.1# ./hdfs dfs -ls /topics/test_hdfs/f1=value6/
-rw-r--r--   3 centos supergroup        281 2019-02-23 18:18 /topics/test_hdfs/f1=value6/test_hdfs+0+0000000005+0000000005.avro
```

```
{
 "fields": [
   {
     "name": "f1",
     "type": "string"
   },
   {
     "default": " ",
     "name": "f2",
     "type": "string"
   }
 ],
 "name": "myrecord",
 "type": "record"
}
```

**Let's see a regex in action**

Now let's modify the new HDFS connector and write 2 tables out to HDFS as a regex test.

Normally you would just PUT a new config, but we will just delete it.

```
curl -X DELETE localhost:8083/connectors/hdfs-sink
```

```
sudo vi ../etc/kafka-connect-hdfs/localhdfstest2.properties
```

Notice instead of just 1 topic listed, test_hdfs, we are using a regex test_hdfs.*

```
name=hdfs-sink
connector.class=io.confluent.connect.hdfs.HdfsSinkConnector
tasks.max=1
topics.regex=test_hdfs.*
hdfs.url=hdfs://localhost:9000
flush.size=3
hadoop.conf.dir=/usr/local/hadoop-2.7.1/etc/hadoop/
partitioner.class=io.confluent.connect.hdfs.partitioner.FieldPartitioner
partition.field.name=f1
rotate.interval.ms=120000
hadoop.home=/usr/local/hadoop-2.7.1/share/hadoop/common/
logs.dir=/opt/hadoop-2.7.1/test_hdfs/logs
schema.compatibility=BACKWARD
```

```
./confluent load hdfs-sink -d ../etc/kafka-connect-hdfs/localhdfstest2.properties
curl -X GET localhost:8083/connectors
```

Produce and consume some sample data to the new topic.

```
./kafka-avro-console-producer --broker-list localhost:9092 --topic test_hdfs2 --property value.schema='{"type":"record","name":"myrecord","fields":[{"name":"f1","type":"string"}]}'
{"f1": "value21"}
{"f1": "value22"}
{"f1": "value23"}
{"f1": "value24"}
{"f1": "value25"}

./kafka-avro-console-consumer --topic test_hdfs2 --bootstrap-server localhost:9092
```

Let's look at the HDFS filesystem. Output resembles something like:

```
bash-4.1# ./hdfs dfs -ls /topics
drwxr-xr-x   - centos supergroup          0 2019-02-24 10:04 /topics/+tmp
drwxr-xr-x   - centos supergroup          0 2019-02-24 10:04 /topics/test_hdfs
drwxr-xr-x   - centos supergroup          0 2019-02-24 10:04 /topics/test_hdfs2

bash-4.1# ./hdfs dfs -ls /topics/test_hdfs2
drwxr-xr-x   - centos supergroup          0 2019-02-24 10:04 /topics/test_hdfs2/f1=value21
drwxr-xr-x   - centos supergroup          0 2019-02-24 10:04 /topics/test_hdfs2/f1=value22
drwxr-xr-x   - centos supergroup          0 2019-02-24 10:04 /topics/test_hdfs2/f1=value23
drwxr-xr-x   - centos supergroup          0 2019-02-24 10:04 /topics/test_hdfs2/f1=value24
drwxr-xr-x   - centos supergroup          0 2019-02-24 10:04 /topics/test_hdfs2/f1=value25
```

That's it. You have now seen a simple example in action.

Also, here are some tips that may help depending on your environment.

Remember the permissions issue above in case you hit it.

```
cd $HADOOP_PREFIX
cd bin/
./hdfs dfs -chmod 777  /
```

If your NN goes into safemode, you can check that status and leave safemode. Often the case in a test environment is if you don't have adequate space on the host or you set your dfs.replication specifically and it's larger than the DN nodes you have available.

```
./hdfs dfsadmin -safemode get
./hdfs dfsadmin -safemode leave
```

Make sure you have enough space on your system as well.

```
sudo du -a / | sort -n -r | head -n 5
sudo du -a /tmp | sort -n -r | head -n 5
```

Re-entering your running container.

```
sudo docker exec -it scottsHadoop /bin/bash
```

If you need to tear down your container and start over.

```
sudo docker ps -all
sudo docker rm <containerid>
```

Clean out your HDFS directories for testing.

```
./hdfs dfs -rm -r /topics/*
```
