# Docker Compose部署 EFK（Elasticsearch + Fluentd + Kibana）收集日志
## 简述
最近需要用到容器日志收集，目前比较流行的应该是EL(Logstash)K,EF(Fluentd)K，相比之下Fluentd要比Logstash轻量级，所以决定采用Fluentd。
本文用于记录如何使用Docker Compose部署 EFK（Elasticsearch + Fluentd + Kibana） 收集Docker容器日志，使用EFK，可以获得灵活，易用的日志收集和分析。
fluentd镜像构建相关文件、docker-compose.yml文件都放在 https://github.com/LXD24/EFK 仓库里。

## 1、首先弄个fluentd镜像
因为Fluentd需要fluent-plugin-elasticsearch插件才能将日志传输到Elasticsearch，所以需要根据fluentd基础镜像构建一个集成fluent-plugin-elasticsearch插件的镜像，当然也可以在网上找一个已经集成的镜像，这里懒得找就自己构建了。
按照 https://github.com/fluent/fluentd-docker-image/blob/master/README.md 上的说明创建个Dockerfile文件,看了说明需要先下载两个文件（`fluent.conf` 和 `entrypoint.sh`），上面都有下载地址。

Dockerfile内容如下，因为我想着到时挂载`fluent.conf`配置文件，所以删掉了 `COPY fluent.conf /fluentd/etc/` 这句复制配置文件的命令。
```
FROM fluent/fluentd:v1.11-1

# Use root account to use apk
USER root

# below RUN includes plugin as examples elasticsearch is not required
# you may customize including plugins as you wish
RUN apk add --no-cache --update --virtual .build-deps \
        sudo build-base ruby-dev \
 && sudo gem install fluent-plugin-elasticsearch \
 && sudo gem sources --clear-all \
 && apk del .build-deps \
 && rm -rf /tmp/* /var/tmp/* /usr/lib/ruby/gems/*/cache/*.gem 

#COPY fluent.conf /fluentd/etc/
COPY entrypoint.sh /bin/

USER fluent
```
然后就是`docker build -t custom-fluentd:latest ./` 看着一顿下载构建镜像。

## 2、准备一个会输出日志的镜像
这里我随便弄了个.net core web服务，输出下访问接口的日志到控制台。

## 3、编写docker-compose.yml

内容如下：
```
version: '2'
services:
  webapplication1:
    image: webapplication1
    container_name: webapplication1
    ports:
      - '8001:80'
    links:
      - fluentd
    logging:
      driver: 'fluentd'
      options:
        fluentd-address: localhost:24224
        tag: httpd.access

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
webapplication1是我创建的web服务，需要配置日志驱动为fluentd
fluentd需要挂载`fluent.conf`配置文件,`fluent.conf`内容如下：
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
![](https://img2020.cnblogs.com/blog/1624324/202007/1624324-20200707105743483-973345932.png)
看到四个服务都是done的就可以了。
先访问下webapplication1造点日志，然后访问 http://localhost:5601 ，为Kibana设置匹配的索引名

![](https://img2020.cnblogs.com/blog/1624324/202007/1624324-20200707113843514-1785472362.png)

然后就能看到收集的日志了。

![](https://img2020.cnblogs.com/blog/1624324/202007/1624324-20200707114012637-1951440371.png)

