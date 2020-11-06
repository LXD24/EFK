## 简述
最近需要用到容器日志收集,目前比较流行的应该是EL(Logstash)K,EF(Fluentd)K,相比之下Fluentd要比Logstash轻量级,所以决定采用Fluentd。

本文用于记录如何使用Docker Compose部署 EFK（Elasticsearch + Fluentd + Kibana） 收集Docker容器日志,使用EFK,可以获得灵活,易用的日志收集和分析。
fluentd镜像构建相关文件、docker-compose.yml文件都放在 https://github.com/LXD24/EFK 仓库里。

## 1、首先弄个fluentd镜像
因为Fluentd需要fluent-plugin-elasticsearch插件才能将日志传输到Elasticsearch,所以需要根据fluentd基础镜像构建一个集成插件的镜像。

相关资料：

fluentd官方镜像：https://hub.docker.com/_/fluentd?tab=description
fluentd官方构建镜像实例：https://github.com/fluent/fluentd-docker-image/blob/master/README.md

新建个dockerfile文件构建镜像,dockerfile内容很简单,基础镜像上在装个 `fluent-plugin-elasticsearch` 插件,装插件需要切换到root用户,否则会提示没权限(没有sudo命令)
```
FROM fluent/fluentd:v1.9.1-debian-1.0
User root
RUN gem install fluent-plugin-elasticsearch
User fluent
```

然后执行`docker build -t custom-fluentd:latest ./` 构建镜像,下载fluentd基础镜像的时间可能会稍长

## 2、准备一个会输出日志的镜像

这里我就用了nginx官方镜像,每次访问nginx的时候它都会输出一段日志。

## 3、编写docker-compose.yml

内容如下：
```
version: '2'
services:
  nginx:
    image: nginx
    container_name: nginx
    ports:
      - '8001:80'
    links:
      - fluentd
    logging:
      driver: 'fluentd'
      options:
        fluentd-address: localhost:24224
        tag: nginx

  fluentd:
    image: custom-fluentd
    container_name: fluentd
    volumes:
      - ./fluentd/conf:/fluentd/etc
    links:
      - 'elasticsearch'
    ports:
      - '24224:24224'
      - '24224:24224/udp'

  elasticsearch:
    image: elasticsearch:6.6.2
    container_name: elasticsearch
    ports:
      - '9200:9200'
    environment:
      - 'discovery.type=single-node'
      - 'cluster.name=docker-cluster'
      - 'bootstrap.memory_lock=true'
      - 'ES_JAVA_OPTS=-Xms512m -Xmx512m'
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - ./data:/usr/share/elasticsearch/data

  kibana:
    image: kibana:6.6.2
    container_name: kibana
    links:
      - 'elasticsearch'
    ports:
      - '5601:5601'

```
nginx映射本地端口8001,需要配置日志驱动为fluentd; 
kibana映射本地5601端口；
fluentd需要挂载`fluent.conf`配置文件,主要配置链接es的信息和存储日志的格式。内容如下：
```
<source>
  @type forward
  port 24224
  bind 0.0.0.0
</source>
<match *.**>
  @type copy
  <store>
    @type elasticsearch
    host elasticsearch
    port 9200
    logstash_format true
    logstash_prefix fluentd
    logstash_dateformat %Y%m%d
    include_tag_key true
    type_name access_log
    tag_key @log_name
    flush_interval 1s
  </store>
  <store>
    @type stdout
  </store>
</match>
```

## 4、启动
到yml文件夹目录下敲 `docker-compose up` 启动。
![](https://img2020.cnblogs.com/blog/1624324/202011/1624324-20201106182742228-843225386.png)

先访问下nginx主页 ` curl http://localhost:8001 ` ,让他输出点访问日志

打开kibana配置下Index Patterns
![](https://img2020.cnblogs.com/blog/1624324/202011/1624324-20201106183917648-1189431736.png)

![](https://img2020.cnblogs.com/blog/1624324/202011/1624324-20201106183929062-486750569.png)



然后进入discover,就可以看到fluentd收集上来的日志（container_name为/nginx）
![](https://img2020.cnblogs.com/blog/1624324/202011/1624324-20201106183234686-1660253737.png)
