## 安装IK分词器插件

#### 我们先判断能否访问插件下载地址，如果网速较好可以采取在线安装，否则建议离线安装

[插件地址：https://github.com/medcl/elasticsearch-analysis-ik/releases](https://github.com/medcl/elasticsearch-analysis-ik/releases)

**版本一定要一致**



- 在线安装

  ``` shell
  // 集群
  docker-compose exec es01 elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.10.1/elasticsearch-analysis-ik-7.10.1.zip
  docker-compose exec es02 elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.10.1/elasticsearch-analysis-ik-7.10.1.zip
  docker-compose exec es03 elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.10.1/elasticsearch-analysis-ik-7.10.1.zip
  //然后要重启es容器
  docker-compose restart es01
  docker-compose restart es02
  docker-compose restart es03
  ```

  

- 离线安装

1. 下载版本一致的压缩包

2. 使用如下命令复制到容器中

   ``` shell
   docker cp /home/ants/temp/elasticsearch-analysis-ik-7.10.1.zip es02:/usr/share/elasticsearch/plugins/ik
   
   ```

   

3. 进入容器操作

   ```shell
   docker exec -it es01 /bin/bash
   mkdir /usr/share/elasticsearch/plugins/ik
   unzip elasticsearch-analysis-ik-7.10.1.zip
   rm elasticsearch-analysis-ik-7.10.1.zip
   
   ```

   

4. 重启容器，所有的都是这样操作一下。

