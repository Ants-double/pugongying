# Kafka的安装及验证

## kafka安装

1. 查看镜像

   ``` shell
   docker search zookeeper
   docker search kafka
   ```

   

2. 下载镜像

   ``` shell
   docker pull zookeeper
   docker pull wurstmeister/kafka
   docker pull hlebalbau/kafka-manager #管理工具
   ```

   

3. 创建必要的文件及文件夹（docker-compose.yaml同一目录下)

   3.1 创建根目录

   ``` shell
   mkdir /home/ants/dev/mq
   ```

   

   3.2 创建kafka文件夹

   ``` shell
   mkdir kafka1
   mkdir kafka2 
   mkdir kafka3
   ```

   3.3  创建zookeeper 文件夹

   ``` shell
   mkdir zookeeper1
   mkdir zookeeper2
   mkdir zookeeper3
   ```

   3.4  在zoo1,zoo2,zoo3中分别创建myid文件，并写入分别写入id数字，如zoo1中的myid中写入1

4. 创建zoo配置文件zoo.cfg

   ``` shell
   # The number of milliseconds of each tick
   tickTime=2000
   # The number of ticks that the initial 
   # synchronization phase can take
   initLimit=10
   # The number of ticks that can pass between 
   # sending a request and getting an acknowledgement
   syncLimit=5
   # the directory where the snapshot is stored.
   # do not use /tmp for storage, /tmp here is just 
   # example sakes.
   dataDir=/data
   dataLogDir=/datalog
   # the port at which the clients will connect
   clientPort=2181
   # the maximum number of client connections.
   # increase this if you need to handle more clients
   #maxClientCnxns=60
   #
   # Be sure to read the maintenance section of the 
   # administrator guide before turning on autopurge.
   #
   # http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance
   #
   # The number of snapshots to retain in dataDir
   autopurge.snapRetainCount=3
   # Purge task interval in hours
   # Set to "0" to disable auto purge feature
   autopurge.purgeInterval=1
   server.1= zoo1:2888:3888
   server.2= zoo2:2888:3888
   server.3= zoo3:2888:3888
   ```

   

5. 创建网络

   ``` shell
   docker network create --driver bridge --subnet 172.23.0.0/25 --gateway 172.23.0.1  zookeeper_network
   ```

   

6. 编写docker-compose.yaml文件

   ``` yaml
   version: '3'
   
   services:
   
     zoo1:
       image: zookeeper # 镜像
       restart: always # 重启
       container_name: zoo1
       hostname: zoo1
       ports:
       - "2181:2181"
       volumes:
       - "./zooConfig/zoo.cfg:/conf/zoo.cfg" # 配置
       - "/home/ants/dev/zk/zookeeper1/data:/data"
       - "/home/ants/dev/zk/zookeeper1/datalog:/datalog"
       environment:
         ZOO_MY_ID: 1 # id
         ZOO_SERVERS: server.1=zoo1:2888:3888 server.2=zoo2:2888:3888 server.3=zoo3:2888:3888
       networks:
         default:
           ipv4_address: 172.23.0.11
   
     zoo2:
       image: zookeeper
       restart: always
       container_name: zoo2
       hostname: zoo2
       ports:
       - "2182:2181"
       volumes:
       - "./zooConfig/zoo.cfg:/conf/zoo.cfg"
       - "/home/ants/dev/zk/zookeeper2/data:/data"
       - "/home/ants/dev/zk/zookeeper2/datalog:/datalog"
       environment:
         ZOO_MY_ID: 2
         ZOO_SERVERS: server.1=zoo1:2888:3888 server.2=zoo2:2888:3888 server.3=zoo3:2888:3888
       networks:
         default:
           ipv4_address: 172.23.0.12
   
     zoo3:
       image: zookeeper
       restart: always
       container_name: zoo3
       hostname: zoo3
       ports:
       - "2183:2181"
       volumes:
       - "./zooConfig/zoo.cfg:/conf/zoo.cfg"
       - "/home/ants/dev/zk/zookeeper3/data:/data"
       - "/home/ants/dev/zk/zookeeper3/datalog:/datalog"
       environment:
         ZOO_MY_ID: 3
         ZOO_SERVERS: server.1=zoo1:2888:3888 server.2=zoo2:2888:3888 server.3=zoo3:2888:3888
       networks:
         default:
           ipv4_address: 172.23.0.13
   
     kafka1:
       image: wurstmeister/kafka # 镜像
       restart: always
       container_name: kafka1
       hostname: kafka1
       ports:
       - 9092:9092
       # - 9988:9988
       environment:
         KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://192.168.15.207:9092 # 暴露在外的地址
         KAFKA_ADVERTISED_HOST_NAME: kafka1 # 
         KAFKA_HOST_NAME: kafka1
         KAFKA_ZOOKEEPER_CONNECT: zoo1:2181,zoo2:2181,zoo3:2181
         KAFKA_ADVERTISED_PORT: 9092 # 暴露在外的端口
         KAFKA_BROKER_ID: 0 # 
         KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092
       #  JMX_PORT: 9988 # jmx
       volumes:
       - /etc/localtime:/etc/localtime
       - "/home/ants/dev/mq/kafka1/logs:/kafka"
       links:
       - zoo1
       - zoo2
       - zoo3
       networks:
         default:
           ipv4_address: 172.23.0.14
   
     kafka2:
       image: wurstmeister/kafka
       restart: always
       container_name: kafka2
       hostname: kafka2
       ports:
       - 9093:9093
      # - 9998:9998
       environment:
         KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://192.168.15.207:9093
         KAFKA_ADVERTISED_HOST_NAME: kafka2
         KAFKA_HOST_NAME: kafka2
         KAFKA_ZOOKEEPER_CONNECT: zoo1:2181,zoo2:2181,zoo3:2181
         KAFKA_ADVERTISED_PORT: 9093
         KAFKA_BROKER_ID: 1
         KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9093
        # JMX_PORT: 9998
       volumes:
       - /etc/localtime:/etc/localtime
       - "/home/ants/dev/mq/kafka2/logs:/kafka"
       links:
       - zoo1
       - zoo2
       - zoo3
       networks:
         default:
           ipv4_address: 172.23.0.15
   
     kafka3:
       image: wurstmeister/kafka
       restart: always
       container_name: kafka3
       hostname: kafka3
       ports:
       - 9094:9094
      # - 9997:9997
       environment:
         KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://192.168.15.207:9094
         KAFKA_ADVERTISED_HOST_NAME: kafka3
         KAFKA_HOST_NAME: kafka3
         KAFKA_ZOOKEEPER_CONNECT: zoo1:2181,zoo2:2181,zoo3:2181
         KAFKA_ADVERTISED_PORT: 9094
         KAFKA_BROKER_ID: 2
         KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9094
         #JMX_PORT: 9997
       volumes:
       - /etc/localtime:/etc/localtime
       - "/home/ants/dev/mq/kafka3/logs:/kafka"
       links:
       - zoo1
       - zoo2
       - zoo3
       networks:
         default:
           ipv4_address: 172.23.0.16
   
     kafka-manager:
       image: hlebalbau/kafka-manager
       restart: always
       container_name: kafka-manager
       hostname: kafka-manager
       ports:
       - 9000:9000
       links:
       - kafka1
       - kafka2
       - kafka3
       - zoo1
       - zoo2
       - zoo3
       environment:
         ZK_HOSTS: zoo1:2181,zoo2:2181,zoo3:2181
         KAFKA_BROKERS: kafka1:9092,kafka2:9093,kafka3:9094
         APPLICATION_SECRET: letmein
         KAFKA_MANAGER_AUTH_ENABLED: "true" # 开启验证
         KAFKA_MANAGER_USERNAME: "admin" # 用户名
         KAFKA_MANAGER_PASSWORD: "admin" # 密码
         KM_ARGS: -Djava.net.preferIPv4Stack=true
       networks:
         default:
           ipv4_address: 172.23.0.10
   
   networks:
     default:
       external:
         name: zookeeper_network
   ```

   

7. 启用集群

   ``` shell
   docker-compose -f docker-compose.yaml up -d
   ```

   

8. 停止集群

   ``` shell
   docker-compose -f docker-compose.yaml stop
   ```

   


## kafka验证

1. 验证每个list都可以看到新建的topic

   ```shell
   docker exec -it kafka1 bash
   kafka-topics.sh --create --zookeeper zoo1:2181 --replication-factor 1 --partitions 3 --topic test001
   kafka-topics.sh --list --zookeeper zoo1:2181
   kafka-topics.sh --list --zookeeper zoo2:2181
   kafka-topics.sh --list --zookeeper zoo3:2181
   ```

   

2. 生产消息

   ``` shell
   
   kafka-console-producer.sh --broker-list kafka1:9092 --topic test001
   ```

   

3. 消费消息

   ``` shell
   docker exec -it kafka2 bash
   
   kafka-console-consumer.sh --bootstrap-server kafka1:9092,kafka2:9093,kafka3:9094 --topic test001 --from-beginning
   ```

   

   

## 可能出现的问题

1. 端口没有开放 (需要外网访问的端口都要开放)

   ```shell
   firewall-cmd --zone=public --add-port=9000/tcp --permanent
   firewall-cmd --reload
   ```

   

2. 端口被占用

   https://www.jianshu.com/p/de4b4cbb0f3c

3. 参考资料

   https://www.cnblogs.com/360minitao/p/12180427.html

   



