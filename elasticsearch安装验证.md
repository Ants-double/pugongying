













## 常见错误

1. max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]

   解决方法：

   [root@chenxi elasticsearch]# vim /etc/sysctl.conf

   在文件末尾追加：vm.max_map_count=655360

   保存后执行

   [root@chenxi elasticsearch]# sysctl -p

2. ERROR: Elasticsearch did not exit normally - check the logs at /usr/share/elasticsearch/logs/docker-cluster.log

   解决方法

   在运行命令中添加 -e "discovery.type=single-node"

3. 