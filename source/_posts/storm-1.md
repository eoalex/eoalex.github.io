---
title: 使用Docker快速安装简单Storm集群环境
tags:
  - Storm
  - zookeeper
id: 1544
categories:
  - 大数据
  - 运维
date: 2017-02-23 10:59:05
---

1.环境准备
主机：虚拟机Ubuntu 16.04 内存2G 硬盘20GB
Docker镜像：zookeeper版本3.4.6 Storm版本1.0.2
2.安装
2.1 docker file
zookeeper的dockerfile

    FROM java:openjdk-8-jre-alpine
    ARG MIRROR=http://mirrors.aliyun.com/
    ARG VERSION=3.4.6
    LABEL name="zookeeper" version=$VERSION
    RUN apk update &amp;&amp; apk add ca-certificates &amp;&amp; \
        apk add tzdata &amp;&amp; \
        ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime &amp;&amp; \
        echo "Asia/Shanghai" &gt; /etc/timezone

    RUN apk add --no-cache wget bash \
        &amp;&amp; mkdir /opt \
        &amp;&amp; wget -q -O - $MIRROR/apache/zookeeper/zookeeper-$VERSION/zookeeper-$VERSION.tar.gz | tar -xzf - -C /opt \
        &amp;&amp; mv /opt/zookeeper-$VERSION /opt/zookeeper \
        &amp;&amp; cp /opt/zookeeper/conf/zoo_sample.cfg /opt/zookeeper/conf/zoo.cfg \
        &amp;&amp; mkdir -p /tmp/zookeeper

    EXPOSE 2181
    WORKDIR /opt/zookeeper
    VOLUME ["/opt/zookeeper/conf", "/tmp/zookeeper"]
    ENTRYPOINT ["/opt/zookeeper/bin/zkServer.sh"]
    CMD ["start-foreground"]`</pre>
    Storm的dockerfile
    <pre>`FROM openjdk:8-jre-alpine
    # Install required packages
    RUN apk add --no-cache \
        bash \
        python \
        su-exec
    ENV STORM_USER=storm \
        STORM_CONF_DIR=/conf \
        STORM_DATA_DIR=/data \
        STORM_LOG_DIR=/logs
    # Add a user and make dirs
    RUN set -x \
        && adduser -D "$STORM_USER" \
        && mkdir -p "$STORM_CONF_DIR" "$STORM_DATA_DIR" "$STORM_LOG_DIR" \
        && chown -R "$STORM_USER:$STORM_USER" "$STORM_CONF_DIR" "$STORM_DATA_DIR" "$STORM_LOG_DIR"
    ARG GPG_KEY=ACEFE18DD2322E1E84587A148DE03962E80B8FFD
    ARG DISTRO_NAME=apache-storm-1.0.2
    # Download Apache Storm, verify its PGP signature, untar and clean up
    RUN set -x \
        && apk add --no-cache --virtual .build-deps \
            gnupg \
        && wget -q "http://www.apache.org/dist/storm/$DISTRO_NAME/$DISTRO_NAME.tar.gz" \
        && wget -q "http://www.apache.org/dist/storm/$DISTRO_NAME/$DISTRO_NAME.tar.gz.asc" \
        && export GNUPGHOME="$(mktemp -d)" \
        && gpg --keyserver ha.pool.sks-keyservers.net --recv-key "$GPG_KEY" \
        && gpg --batch --verify "$DISTRO_NAME.tar.gz.asc" "$DISTRO_NAME.tar.gz" \
        && tar -xzf "$DISTRO_NAME.tar.gz" \
        && chown -R "$STORM_USER:$STORM_USER" "$DISTRO_NAME" \
        && rm -r "$GNUPGHOME" "$DISTRO_NAME.tar.gz" "$DISTRO_NAME.tar.gz.asc" \
        && apk del .build-deps
    WORKDIR $DISTRO_NAME
    ENV PATH $PATH:/$DISTRO_NAME/bin
    COPY docker-entrypoint.sh /
    ENTRYPOINT ["/docker-entrypoint.sh"]`</pre>
    2.2 启动
    step1 启动zookeeper
    <pre>`docker run -d --restart always --name zookeeper zookeeper:3.4.6`</pre>
    step2 启动Nimbus
    <pre>`docker run -d --restart always --name nimbus --link zookeeper storm:1.0.2 storm nimbus`</pre>
    step3 启动Storm UI
    <pre>`docker run -d -p 8080:8080 --restart always --name ui --link nimbus storm:1.0.2 storm ui`</pre>
    此时进入127.0.0.1:8080,我们看到nimbus已启动。
    [![](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2017/02/2017-02-23_09-26-09.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2017/02/2017-02-23_09-26-09.png)
    step4 启动Supervisor
    <pre>`docker run -d --restart always --name supervisor1 --link zookeeper --link nimbus storm:1.0.2 storm supervisor`</pre>
    [![](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2017/02/2017-02-23_09-29-34.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2017/02/2017-02-23_09-29-34.png)
    我们看到Supervisor1已正常启动，如果需要我们可以启动多个Supervisor，至此简单Storm集群环境安装完毕。
    3.提交Topology
    进入nimbus的docker,使用storm的example WordCountTopology提交Topology
    <pre>`docker exec -it nimbus bash
    cd examples/storm-starter
    storm jar storm-starter-topologies-1.0.2.jar org.apache.storm.starter.WordCountTopology first-topology

运行情况
[![](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2017/02/2017-02-23_10-16-42.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2017/02/2017-02-23_10-16-42.png)
详细信息
[![](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2017/02/2017-02-23_11-01-19.png)](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2017/02/2017-02-23_11-01-19.png)