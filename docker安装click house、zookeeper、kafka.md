- [拉取镜像](#拉取镜像)
- [zookeeper](#zookeeper)
  - [docker安装zookeeper集群](#docker安装zookeeper集群)
- [kafka](#kafka)
  - [docker安装kafka集群](#docker安装kafka集群)
- [click house](#click-house)
  - [docker安装click house集群](#docker安装click-house集群)
# 拉取镜像
```sh
下载镜像
docker pull docker.io/yandex/clickhouse-server
docker pull docker.io/yandex/zookeeper
docker pull bitnami/kafka

cat /etc/containers/registries.conf
设置
unqualified-search-registries = ["docker.io"]
```
# zookeeper
## docker安装zookeeper集群
```sh
docker pull docker.io/zookeeper
docker network create --subnet=10.206.13.0/24 zookeeper-network

docker run --restart=always --name zk01 -d --net=zookeeper-network --ip 10.206.13.2 docker.io/library/zookeeper
docker run --restart=always --name zk02 -d --net=zookeeper-network --ip 10.206.13.3 docker.io/library/zookeeper
docker run --restart=always --name zk03 -d --net=zookeeper-network --ip 10.206.13.4 docker.io/library/zookeeper

#进入docker设置myid
docker exec -it zk01 bash
echo 1 >/data/myid
docker exec -it zk02 bash
echo 2 >/data/myid
docker exec -it zk03 bash
echo 3 >/data/myid

#配置zoo.cfg
#vim /tmp/zoo_1.cfg
server.1=localhost:2888:3888;2181
dataDir=/data
dataLogDir=/datalog
tickTime=2000
initLimit=5
syncLimit=2
autopurge.snapRetainCount=3
autopurge.purgeInterval=0
maxClientCnxns=60
standaloneEnabled=true
admin.enableServer=true
server.1=0.0.0.0:2888:3888;2181
server.2=10.206.13.3:2888:3888;2181
server.3=10.206.13.4:2888:3888;2181

#vim /tmp/zoo_2.cfg
dataDir=/data
dataLogDir=/datalog
tickTime=2000
initLimit=5
syncLimit=2
autopurge.snapRetainCount=3
autopurge.purgeInterval=0
maxClientCnxns=60
standaloneEnabled=true
admin.enableServer=true
server.1=10.206.13.2:2888:3888;2181
server.2=0.0.0.0:2888:3888;2181
server.3=10.206.13.4:2888:3888;2181

#vim /tmp/zoo_3.cfg
dataDir=/data
dataLogDir=/datalog
tickTime=2000
initLimit=5
syncLimit=2
autopurge.snapRetainCount=3
autopurge.purgeInterval=0
maxClientCnxns=60
standaloneEnabled=true
admin.enableServer=true
server.1=10.206.13.2:2888:3888;2181
server.2=10.206.13.3:2888:3888;2181
server.3=0.0.0.0:2888:3888;2181

#修改docker里的zk配置
docker cp /tmp/zoo_1.cfg zk01:/conf/zoo.cfg
docker cp /tmp/zoo_2.cfg zk02:/conf/zoo.cfg
docker cp /tmp/zoo_3.cfg zk03:/conf/zoo.cfg

#启动zk
docker restart zk01
docker restart zk02
docker restart zk03

#查看zk状态
/apache-zookeeper-3.8.0-bin/bin/zkServer.sh status
/apache-zookeeper-3.8.0-bin/bin/zkCli.sh

#其他操作
docker stop zk01
docker stop zk02
docker stop zk03
docker rm zk01
docker rm zk02
docker rm zk03
```








dataDir=/data
dataLogDir=/datalog
tickTime=2000
initLimit=5
syncLimit=2
autopurge.snapRetainCount=3
autopurge.purgeInterval=0
maxClientCnxns=60
standaloneEnabled=true
admin.enableServer=true
server.1=10.206.13.2:2888:3888;2181
server.2=10.206.13.3:2888:3888;2181
server.3=0.0.0.0:2888:3888;2181


docker cp /tmp/zoo_1.cfg zk01:/conf/zoo.cfg
docker cp /tmp/zoo_2.cfg zk02:/conf/zoo.cfg
docker cp /tmp/zoo_3.cfg zk03:/conf/zoo.cfg

docker restart zk01
docker restart zk02
docker restart zk03

/apache-zookeeper-3.8.0-bin/bin/zkServer.sh status
/apache-zookeeper-3.8.0-bin/bin/zkCli.sh

docker stop zk01
docker stop zk02
docker stop zk03
docker rm zk01
docker rm zk02
docker rm zk03
# kafka
## docker安装kafka集群
```sh

docker run -it --restart=always \
--name kafka01 --net=zookeeper-network --ip 10.206.13.8 -e KAFKA_BROKER_ID=0 \
-e KAFKA_ZOOKEEPER_CONNECT=10.206.13.2:2181,10.206.13.3:2181,10.206.13.4:2181 \
-e ALLOW_PLAINTEXT_LISTENER=yes \
-e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://10.206.13.8:9092 \
-e KAFKA_LISTENERS=PLAINTEXT://10.206.13.8:9092 \
-d docker.io/bitnami/kafka

docker run -it --restart=always \
--name kafka02 --net=zookeeper-network --ip 10.206.13.9 -e KAFKA_BROKER_ID=1 \
-e KAFKA_ZOOKEEPER_CONNECT=10.206.13.2:2181,10.206.13.3:2181,10.206.13.4:2181 \
-e ALLOW_PLAINTEXT_LISTENER=yes \
-e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://10.206.13.9:9092 \
-e KAFKA_LISTENERS=PLAINTEXT://10.206.13.9:9092 \
-d docker.io/bitnami/kafka

docker run -it --restart=always \
--name kafka03 --net=zookeeper-network --ip 10.206.13.10 -e KAFKA_BROKER_ID=2 \
-e KAFKA_ZOOKEEPER_CONNECT=10.206.13.2:2181,10.206.13.3:2181,10.206.13.4:2181 \
-e ALLOW_PLAINTEXT_LISTENER=yes \
-e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://10.206.13.10:9092 \
-e KAFKA_LISTENERS=PLAINTEXT://10.206.13.10:9092 \
-d docker.io/bitnami/kafka

#查看zookeeper
docker exec -it zk01 bash
./bin/zkCli.sh
ls /brokers/ids
[2, 1, 0]

#其他操作
docker stop kafka01
docker stop kafka02
docker stop kafka03
docker rm kafka01
docker rm kafka02
docker rm kafka03

#创建topic
docker exec -it kafka01 bash
/opt/bitnami/kafka/bin/kafka-topics.sh --bootstrap-server 10.206.13.8:9092 --create --topic test --partitions 1 --replication-factor 1

# 连接另外两台查看新创建的test topics
/opt/bitnami/kafka/bin/kafka-topics.sh --list --bootstrap-server 10.206.13.9:9092
/opt/bitnami/kafka/bin/kafka-topics.sh --list --bootstrap-server 10.206.13.10:9092

# 写（CTRL+D结束写内容）
/opt/bitnami/kafka/bin/kafka-console-producer.sh --broker-list 10.206.13.8:9092 --topic test
>hello
# 读（CTRL+C结束读内容）
/opt/bitnami/kafka/bin/kafka-console-consumer.sh --bootstrap-server 10.206.13.10:9092 --topic test --from-beginning
hello
```

# click house
## docker安装click house集群
```sh
#自定义目录
rm -rf /data/clickhouse_1 /data/clickhouse_2 /data/clickhouse_3
mkdir /data/clickhouse_1
mkdir /data/clickhouse_2
mkdir /data/clickhouse_3
mkdir -p /data/clickhouse_1/logs /data/clickhouse_2/logs /data/clickhouse_3/logs

#启动
docker run -d --name clickhouse-server01 --net=zookeeper-network --ip 10.206.13.5 --ulimit nofile=262144:262144 -v /data/clickhouse_1/:/var/lib/clickhouse docker.io/yandex/clickhouse-server
docker run -d --name clickhouse-server02 --net=zookeeper-network --ip 10.206.13.6 --ulimit nofile=262144:262144 -v /data/clickhouse_2/:/var/lib/clickhouse docker.io/yandex/clickhouse-server
docker run -d --name clickhouse-server03 --net=zookeeper-network --ip 10.206.13.7 --ulimit nofile=262144:262144 -v /data/clickhouse_3/:/var/lib/clickhouse docker.io/yandex/clickhouse-server

#修改clickhouse配置文件
docker cp clickhouse-server01:/etc/clickhouse-server/ /data/clickhouse_1/
docker cp clickhouse-server02:/etc/clickhouse-server/ /data/clickhouse_2/
docker cp clickhouse-server03:/etc/clickhouse-server/ /data/clickhouse_3/
#修改关键节点
vim /data/clickhouse_1/clickhouse-server/config.xml
    <remote_servers>
        <test_cluster_three_shards>
            <shard>
                <replica>
                    <host>10.206.13.5</host>
                    <port>9000</port>
                </replica>
            </shard>
            <shard>
                <replica>
                    <host>10.206.13.6</host>
                    <port>9000</port>
                </replica>
            </shard>
			<shard>
                <replica>
                    <host>10.206.13.7</host>
                    <port>9000</port>
                </replica>
            </shard>
        </test_cluster_three_shards>
    </remote_servers>

	<timezone>Asia/Shanghai</timezone>

#拷贝config.xml
cp /tmp/config.xml /data/clickhouse_1/clickhouse-server/config.xml
cp /tmp/config.xml /data/clickhouse_2/clickhouse-server/config.xml
cp /tmp/config.xml /data/clickhouse_3/clickhouse-server/config.xml

#重启click house
docker run --restart always -d \
--name clickhouse-server01 \
--net=zookeeper-network --ip 10.206.13.5 \
--ulimit nofile=262144:262144 \
-v /data/clickhouse_1/data/:/var/lib/clickhouse/ \
-v /data/clickhouse_1/clickhouse-server/:/etc/clickhouse-server/ \
-v /data/clickhouse_1/logs/:/var/log/clickhouse-server/ \
docker.io/yandex/clickhouse-server

docker run --restart always -d \
--name clickhouse-server02 \
--net=zookeeper-network --ip 10.206.13.6 \
--ulimit nofile=262144:262144 \
-v /data/clickhouse_2/data/:/var/lib/clickhouse/ \
-v /data/clickhouse_2/clickhouse-server/:/etc/clickhouse-server/ \
-v /data/clickhouse_2/logs/:/var/log/clickhouse-server/ \
docker.io/yandex/clickhouse-server

docker run --restart always -d \
--name clickhouse-server03 \
--net=zookeeper-network --ip 10.206.13.7 \
--ulimit nofile=262144:262144 \
-v /data/clickhouse_3/data/:/var/lib/clickhouse/ \
-v /data/clickhouse_3/clickhouse-server/:/etc/clickhouse-server/ \
-v /data/clickhouse_3/logs/:/var/log/clickhouse-server/ \
docker.io/yandex/clickhouse-server

#测试集群可用性
docker exec -it clickhouse-server01 bash
clickhouse-client
clickhouse-client -h 10.206.13.5 --port 9000 --password 密码
select * from system.clusters;

# 创建库表，三台主机一起创建
create database chainmaker_test;
use chainmaker_test;
CREATE TABLE logs (cur Date, size Int32, message String) ENGINE = MergeTree(cur, message, 8192);
CREATE TABLE logs_dist AS logs ENGINE = Distributed(test_cluster_three_shards,chainmaker_test,logs,rand());
		
# 下列命令在Clickhouse的一个主机上执行即可；
insert into logs_dist values(now(), 1, '1');
# 在任何一个Clickhouse的主机上查询数据一致即表示集群 状态正常
select * from logs_dist;

#关闭、删除
docker stop clickhouse-server01
docker stop clickhouse-server02
docker stop clickhouse-server03
docker rm clickhouse-server01
docker rm clickhouse-server02
docker rm clickhouse-server03
```