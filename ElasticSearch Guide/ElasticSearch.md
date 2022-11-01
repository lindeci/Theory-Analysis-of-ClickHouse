- [](#)
- [1](#1)
# 1、网上资料
- 官网 https://www.elastic.co/guide/index.html 
- 书籍  
  《Elasticsearch权威指南（中文版）》
- 社区  
  中文社区 https://elasticsearch.cn/  
  Elastic 社区 Meetup https://meetup.elasticsearch.cn/event/index.html （最后一次举办时间时间：2019-11-16）  
  Elastic 中国社区官方博客 https://blog.csdn.net/UbuntuTouch  
  
# 1、安装Elasticsearch
```sh

wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.16.3-linux-x86_64.tar.gz
tar xvf elasticsearch-7.16.3-linux-x86_64.tar.gz
groupadd es
useradd es -g es
chown -R es:es elasticsearch-7.16.3
cd elasticsearch-7.16.3

允许远程访问控制台
vi ./config/elasticsearch.yml
network.host: 0.0.0.0

在后台以守护进程模式运行
su - es -c"/data/elasticsearch-7.16.3/bin/elasticsearch -d"
遇到问题1：
    max file descriptors [4096] for elasticsearch process is too low
解决1:
  vi /etc/security/limits.conf
* hard nofile 65536
* soft nofile 65536

问题2：
    max virtual memory areas vm.max_map_count [65530] is too low
解决3:
  vi /etc/sysctl.conf
  vm.max_map_count=655360
  sysctl -p

问题4:
  memory locking requested for elasticsearch process but memory is not locked
解决4:
  vi /etc/systemd/system.conf
  DefaultLimitMEMLOCK=infinity

测试
http://172.16.13.196:9200/?pretty
```
# 2、安装Marvel
```sh
下载和安装
./bin/plugin -i elasticsearch/marvel/latest

关闭
echo 'marvel.agent.enabled: false' >> ./config/elasticsearch.yml
```
