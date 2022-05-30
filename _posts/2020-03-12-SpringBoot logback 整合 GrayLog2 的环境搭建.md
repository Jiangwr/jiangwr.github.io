---
layout: post
title: SpringBoot logback 整合 GrayLog2 的环境搭建
---
首先需要说明的是，安装的环境是 CentOS 7.2。

<!--more-->

## 安装 GrayLog2

### 1.1 、安装 JDK 1.8

```
yum -y install java-1.8.0-openjdk-headless.x86_64
```

### 1.2 、安装两个工具(后面会用到)

```
sudo yum install epel-release
```

```
sudo yum install pwgen
```

### 1.3 、安装 MongoDB

编辑 `/etc/yum.repos.d/mongodb-org-3.2.repo`

```
vi /etc/yum.repos.d/mongodb-org-3.2.repo
```

加入以下内容:

```
[mongodb-org-3.2]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/3.2/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-3.2.asc
```

安装

```
sudo yum install mongodb-org
```

注册并启动服务

```
sudo chkconfig --add mongod
```

```
sudo systemctl daemon-reload
```

```
sudo systemctl enable mongod.service
```

```
sudo systemctl start mongod.service
```

### 1.4 、安装 Elasticsearch

```
 rpm --import https://packages.elastic.co/GPG-KEY-elasticsearch
```

编辑 `/etc/yum.repos.d/elasticsearch.repo` 

```
vi /etc/yum.repos.d/elasticsearch.repo 
```

添加以下内容:

```
[elasticsearch-2.x]
name=Elasticsearch repository for 2.x packages
baseurl=https://packages.elastic.co/elasticsearch/2.x/centos
gpgcheck=1
gpgkey=https://packages.elastic.co/GPG-KEY-elasticsearch
enabled=1
```

安装

```
sudo yum install elasticsearch
```

修改配置文件  `/etc/elasticsearch/elasticsearch.yml`

```
vi  /etc/elasticsearch/elasticsearch.yml
```

把 cluster**.**name 前面的# 去掉 然后 改成  cluster**.**name:graylog (这个名字也可以自己定义)

注册并启动服务

```
sudo chkconfig --add elasticsearch
```

```
sudo systemctl daemon-reload
```

```
sudo systemctl enable elasticsearch.service
```

```
sudo systemctl restart elasticsearch.service
```

测试

```
curl -X GET http://localhost:9200
```

显示结果类似于以下内容，表示成功安装

```
{
  "name" : "Rebel",
  "cluster_name" : "graylog",
  "cluster_uuid" : "fV0AQNctSAiQ48UAKGZRew",
  "version" : {
    "number" : "2.4.6",
    "build_hash" : "5376dca9f70f3abef96a77f4bb22720ace8240fd",
    "build_timestamp" : "2017-07-18T12:17:44Z",
    "build_snapshot" : false,
    "lucene_version" : "5.5.4"
  },
  "tagline" : "You Know, for Search"
}

```

### 1.5 、安装 GrayLog2

```
sudo rpm -Uvh https://packages.graylog2.org/repo/packages/graylog-2.2-repository_latest.rpm
```

```
sudo yum install graylog-server
```

配置

生成password_secret密码

```
pwgen -N 1 -s 96
```

生成root_password_sha2密码

```
echo -n 123456 | sha256sum
```

这里生存的root_password_sha2类似于`8d969eef6ecad3c29a3a629280e686cf0c3f5d5a86aff3ca12020c923adc6c92  -`，放到配置文件时**横杠**不要加入

修改 `/etc/graylog/server/server.conf`

```
vi /etc/graylog/server/server.conf
```

修改为以下内容

```
password_secret =  刚才生成的密钥字符串
root_password_sha2 = 刚才生成的密码字符串
root_timezone = Asia/Shanghai
web_listen_uri = http://0.0.0.0:9000/
rest_listen_uri = http://0.0.0.0:12900/
rest_transport_uri = http://192.168.128.131:12900/   (IP就是服务器的IP)
elasticsearch_cluster_name = graylog(和前面定义的cluster.name一致)
elasticsearch_shards = 1
elasticsearch_replicas = 0
```

注册服务并启动

```
sudo chkconfig --add graylog-server
```

```
sudo systemctl daemon-reload
```

```
sudo systemctl enable graylog-server.service
```

```
sudo systemctl start graylog-server.service
```

关闭防火墙

```
systemctl stop firewalld.service
```

```
systemctl disable firewalld.service
```

```
setenforce 0
```

修改 `/etc/sysconfig/selinux`

```
vi /etc/sysconfig/selinux
```

SELINUX=disable。

至此，GrayLog2 安装完成，浏览器输入 http://ip:9000 即可看到界面。然后通过 admin/123456 作为账号密码登录。

### GragLog 新增一个 input 

如图所示
<img width="1603" alt="添加一个 inputs 第1步" src="https://user-images.githubusercontent.com/19362571/170969283-0f701132-97c3-4547-bdd0-dee404a88da8.png">
<img width="1576" alt="添加一个 inputs 第2步" src="https://user-images.githubusercontent.com/19362571/170969299-ada0afbb-46f0-4cab-822a-7a1a1583288e.png">
<img width="591" alt="添加一个 inputs 第3步" src="https://user-images.githubusercontent.com/19362571/170969314-06e83956-bab7-430c-b21d-df2629f3b3fc.png">


## Springboot 配置

### pom 添加依赖

```
 <dependency>
            <groupId>de.siegmar</groupId>
            <artifactId>logback-gelf</artifactId>
            <version>1.1.0</version>
 </dependency>
```

### 配置日志输出

在resources目录下（application.yml同级目录）添加 logback.xml

```
<?xml version="1.0" encoding="UTF-8"?>
<configuration>

    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <!--发送GELF UDP 到 graylog-->
    <!--使用第三方组件 https://github.com/osiegmar/logback-gelf -->
    <appender name="GELF" class="de.siegmar.logbackgelf.GelfUdpAppender">
        <graylogHost>192.168.1.217</graylogHost>
        <graylogPort>12201</graylogPort>
        <!--以下为可选配置-->
        <maxChunkSize>508</maxChunkSize>
        <useCompression>true</useCompression>
        <layout class="de.siegmar.logbackgelf.GelfLayout">
            <originHost>logback-graylog</originHost>
            <includeRawMessage>false</includeRawMessage>
            <includeMarker>true</includeMarker>
            <includeMdcData>true</includeMdcData>
            <includeCallerData>false</includeCallerData>
            <includeRootCauseData>false</includeRootCauseData>
            <includeLevelName>false</includeLevelName>
            <shortPatternLayout class="ch.qos.logback.classic.PatternLayout">
                <pattern>%m%nopex</pattern>
            </shortPatternLayout>
            <fullPatternLayout class="ch.qos.logback.classic.PatternLayout">
                <pattern>%m</pattern>
            </fullPatternLayout>
            <staticField>app_name:backend</staticField>
            <staticField>os_arch:${os.arch}</staticField>
            <staticField>os_name:${os.name}</staticField>
            <staticField>os_version:${os.version}</staticField>
        </layout>
    </appender>

    <root level="INFO">
        <appender-ref ref="GELF" />
        <appender-ref ref="STDOUT"/>
    </root>

</configuration>
```

其中 **graylogHost** 需要改为自己的 GrayLog IP地址。

效果展示
<img width="1589" alt="GrayLog效果展示" src="https://user-images.githubusercontent.com/19362571/170969340-cb046742-39d2-45d2-be15-b7b5c1a7831a.png">

## 参考

1.[SpringBoot logback 整合 GrayLog](https://segmentfault.com/a/1190000015767703)

2.[Graylog2+rsyslog+log4j 全过程日志管理环境搭建](
