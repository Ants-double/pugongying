

## docker安装kibana

2. 确定es版本

   ```shell
   docker pull kibana:7.10.1
   ```

   

3. 执行运行命令

   ```shell
   docker run --name kibana_es -e ELASTICSEARCH_URL=http://192.168.15.207:9200 -p 5601:5601 -d kibana:7.10.1
   ```

   

4. 开放5601端口，然后就可以访问了



## docker-compose安装

1. 命令如下

   ```yaml
     kibana:
       image: docker.elastic.co/kibana/kibana:7.10.1
       container_name: kibana
       depends_on:
         - es01
       ports:
         - 5601:5601  
   ```

   

2. 运行

   ``` shell
   docker-compose up 
   ```

   



## 常见问题

1. server is not ready yet

   一. 版本不一致

   二、kibana配置

   a.获取es容器的ip

   ```shell
   docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' es01
   ```

   

   b.进入容器configs文件夹修改

   ``` shell
   $ docker exec -it kibana容器id /bin/bash
   $ cd config
   $ vi kibana.yml
   ```

   ```shell
   #
   # ** THIS IS AN AUTO-GENERATED FILE **
   #
   # Default Kibana configuration for docker target
   server.name: kibana
   server.host: "0"
   #只需要将上面的 "http://elasticsearch:9200" 中的 elasticsearch 替换成上一步的es容器内部ip就可以了。
   elasticsearch.hosts: [ "http://elasticsearch:9200" ]
   xpack.monitoring.ui.container.elasticsearch.enabled: true
   ```

   

   c. 重启容器

   三、网络问题

   四、进入es容器执行如下命令

   ```shell
   
   
   curl -u elastic:changeme -XDELETE 192.168.15.207:9200/_xpack/security/privilege/kibana-.kibana/space_all
   curl -u elastic:changeme -XDELETE 192.168.15.207:9200/_xpack/security/privilege/kibana-.kibana/space_read
   
   ```

   五、执行如下命令

   ```shell
   curl -XDELETE http://192.168.15.207:9200/.kibana
   
   curl -XDELETE http://192.168.15.207:9200/.kibana*
   
   curl -XDELETE http://192.168.15.207:9200/.kibana_2
   
   curl -XDELETE http://192.168.15.207:9200/.kibana_1
   
   ```

   

   

