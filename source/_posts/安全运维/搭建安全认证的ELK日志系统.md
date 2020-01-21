---
title: 搭建安全认证的ELK日志系统
date: 2018-05-13 15:29:01
updated: 2018-05-13 15:29:01
categories:
- 安全运维
tags: ELK
---

# 1. 背景

由于网络安全法要求信息系统运营单位的网络日志保留时间不低于6个月，在信息安全等级保护中也有相应要求，日志记录的保存也是执法检查部门近年的关注的重点，原因即是为了安全事件的事后追查，因此信息系统产生的各种日志必须进行安全的保存，规避违法违规风险，尽到作为信息系统运营单位安全保护责任。因本人在某政府单位驻场安全运维，该单位没有日志系统，因此我决定使用著名的ELK日志系统搭建了日志系统，用于收集全网日志，包括网络设备syslog、操作系统syslog、apahce、tomcat、iis、was、weblogic等。但ELK日志系统默认是没有认证功能的，默认情况下日志存储安全是无法保证的，ELK自身的漏洞也可能增加风险，很多人忽略了这一点，网上也没有文章强调认证。本文中我采用了nginx进行代理认证，安全认证也是本文的重点，写此文也是希望用自己的实验成果能帮到需要的人，提高安全管理人员安全意识。

# 2. 运行环境

windows 2008 R2  两台作为日志系统服务端， 安装Elasticsearch 5.6.9、Logstash 5.6.9、Nginx1.14.0、JDK1.8+。

centos 6.9 一台作为日志系统客户端，安装filebeat 5.6.9

# 3. 方案

filebeat：
作为客户端采集
网站A 网站B....的apache、tomcat、nginx等中间件log文件,并配置tags（用于区分日志类型）和fields:service（用于区分日志来源网站建立不同索引）参数，不使用logstash作为客户端的原因是logstash太占用资源了，还需要java环境。filebeat接收syslog和filebeat发来的日志在filter中根据tags类型格式化，在output中根据fields:service配置索引输出到Elasticsearch不同网站建立不同索引。客户端不会直接请求Elasticsearch的9200端口，确保安全。

Elasticsearch：
安装到windows服务器，作日志存储处理，所在服务器开启防火墙。

策略1：允许集群节点之间tcp9200、tcp9300端口、ping通信。

策略2：允许filebeat客户端访问服务器tcp54320和udp514端口（logstash中定义）。
策略3：允许管理员ip访问经过nginx代理认证后的head插件端口tcp19100（nginx中自定义）、Elasticsearch的tcp19200端口和kibana的tcp15601端口，方便管理员远程管理。所有策略配置好后客户端无法直接访问Elasticsearch的9100、9200、9300端口和kibana的5601端口，仅管理员可访问代理后的19100、19200、15601端口，对应9100、9200、5601端口。

Kibana：
安装到windows服务器，作为日志展示和查询。

Elasticsearch-head：Elasticsearch的图形管理插件。

Nginx：
安装到主windows服务器节点，因为Elasticsearch默认是无认证的，日志可被恶意删除，无论在内网还是公网都是非常危险的，因此有必要采用nginx进行反向代理认证。

Nssm：
将免安装的logstah、kibana、nginx安装为服务，配置为自动启动。

*对于性能有需求的可考虑使用redis，非本文关注重点，不再介绍了。

# 4. 下载必要软件和版本选择
*ELK5.6.9支持到windows server 2008 操作系统，更高版本需要windows server 2012操作系统。

Elasticsearch 5.6.9
 [https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-5.6.9.zip](https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-5.6.9.zip)

Logstash 5.6.9
 [https://artifacts.elastic.co/downloads/logstash/logstash-5.6.9.zip](https://artifacts.elastic.co/downloads/logstash/logstash-5.6.9.zip)

filebeat 5.6.9 linux
 [https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-5.6.9-x86_64.rpm](https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-5.6.9-x86_64.rpm)

filebeat 5.6.9 windows
 [https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-6.3.2-windows-x86_64.zip](https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-6.3.2-windows-x86_64.zip)


Kibana 5.6.9
 [https://artifacts.elastic.co/downloads/kibana/kibana-5.6.9-windows-x86.zip](https://artifacts.elastic.co/downloads/kibana/kibana-5.6.9-windows-x86.zip)

JDK（1.8以上） [http://download.oracle.com/otn-pub/java/jdk/8u171-b11/512cd62ec5174c3487ac17c61aaa89e8/jdk-8u171-windows-x64.exe](http://download.oracle.com/otn-pub/java/jdk/8u171-b11/512cd62ec5174c3487ac17c61aaa89e8/jdk-8u171-windows-x64.exe)

elasticsearch-head
 [https://github.com/mobz/elasticsearch-head#running-with-built-in-server](https://github.com/mobz/elasticsearch-head#running-with-built-in-server)

nginx 1.14.0
 [http://nginx.org/download/nginx-1.14.0.zip](http://nginx.org/download/nginx-1.14.0.zip)

nssm
 [http://www.nssm.cc/release/nssm-2.24.zip](http://www.nssm.cc/release/nssm-2.24.zip)


# 5. 安装ELK


## 5.1 Elasticsearch安装配置

解压修改配置文件即可运行。

坑：elasticsearch必须是两个组成集群，不然会出现错误，也可在同一台服务器运行两个elasticsearch，如果启动报错，是因为复制两个elasticsearch时data目录已有数据，造成冲突，清空data里面的数据即可启动两个elasticsearch。本文采用两台服务器搭建，配置如下：

### 主节点配置文件elasticsearch.yml

```
#禁用虚拟内存，提高性能
bootstrap.memory_lock: true
#节点名称自定义：
cluster.name: elasticsearch
#数据通信端口：
http.port: 9200
#监听网卡ip
network.host: 192.168.1.1
#是否是数据节点：
node.data: true
#关闭即可：
node.ingest: true
#是否是主节点,不定义的话先启动的是主节点：
node.master: true
#最大存储节点：
node.max_local_storage_nodes: 1
#节点名字自定义：
node.name: Win-Master-1
#数据文件路径
path.data: D:\elk\elasticsearch\data
path.logs: D:\elk\elasticsearch\logs
#节点间通信端口：
transport.tcp.port: 9300
#节点ip，节点之间要允许ping和9300端口通信
discovery.zen.ping.unicast.hosts: ["192.168.1.1", "192.168.1.2"]
#head插件相关：
http.cors.enabled: true
http.cors.allow-origin: "*"
```

### 备节点配置文件elasticsearch.yml

```
#禁用虚拟内存，提高性能
bootstrap.memory_lock: true
#节点名称自定义：
cluster.name: elasticsearch
#数据通信端口：
http.port: 9200
#监听网卡ip
network.host: 192.168.1.2
#是否是数据节点：
node.data: true
#关闭即可：
node.ingest: true
#是否是主节点,不定义的话先启动的是主节点：
node.master: false
#最大存储节点：
node.max_local_storage_nodes: 1
#节点名字自定义：
node.name: Win-salve-1
#数据文件路径
path.data: D:\elk\elasticsearch\data
path.logs: D:\elk\elasticsearch\logs
#节点间通信端口：
transport.tcp.port: 9300
#节点ip，节点之间要允许ping和9300端口通信
discovery.zen.ping.unicast.hosts: ["192.168.1.1", "192.168.1.2"]
#head插件相关：
http.cors.enabled: true
http.cors.allow-origin: "*"
```
### 安装服务

进入bin目录在cmd中运行：
```
elasticsearch-service.bat install
#图形服务管理工具或windows服务中启动服务
elasticsearch-service-mgr.exe
```
### 安装head插件

1.下载安装node.js ,网址：https://nodejs.org/en/
2.nodejs安装目录下运行：
> npm install -g grunt-cli

3.下载head插件，网址：https://github.com/mobz/elasticsearch-head/archive/master.zip
4.解压到elasticsearch下，自定义目录名称head，修改以下源码：

head/Gruntfile.js
```
connect: {
    server: {
        options: {
            port: 9100,
            hostname: '*',
            base: '.',
            keepalive: true
        }
    }
}

```
head/_site/app.js

```
this.base_uri = this.config.base_uri || this.prefs.get("app-base_uri") || "http://192.168.1.1:19100/elasticsearch/";
#这里的地址和第六章nginx代理配置对应。
```

## 5.2 logstash 安装配置
解压修改配置文件即可。
### 新建配置文件logstash.conf
```

input{
    stdin{
    }
    #配置端口，接收syslog日志：
    syslog {
        type => syslog
        port => 514
    }
    #配置端口，接收filebeat发来的日志。
    beats {
        port => 54320
        
    }
    #logstash做客户端时的配置举例，本方案不采用，故注释掉。
    #file {
    #   path => "D:/log/apache/access-*.log"
    #   type => "apache_access"
    #   start_position => "beginning"
    #   ignore_older => 0 #读取已有的日志文件
    #}
    
}
filter {
    #logstash做客户端时的配置举例，本方案不采用，故注释掉。
    #if [type] == "apache_access" {
    #   grok {
    #   match => {"message" => "%{COMBINEDAPACHELOG}" }
    #   }
    #}
    
    #filebeat做客户端时的配置
    #filter其实也可以不配置，但不利于根据字段搜索和分析日志。
    if "apache_access" in [tags] {
        grok {
        match => {"message" => "%{COMBINEDAPACHELOG}" }
        }
        #%{COMBINEDAPACHELOG}内置的apcahe日志匹配规则。
    }
    if "nginx_access" in [tags] {
        grok {
        match => {"message" => "%{COMBINEDAPACHELOG} %{QS:x_forwarded_for} %{HOSTNAME:server_name}" }
        }
        #为了便于搜索多二级域名的网站，这里修改nginx默认日志字段后，添加了{HOSTNAME:server_name}规则，nginx比apahce增加了一个{QS:x_forwarded_for}这里也加上了。
    }
    if "tomcat_access" in [tags] {
        grok {
        match => {"message" => "%{COMBINEDAPACHELOG}" }
        }
        #%{COMBINEDAPACHELOG}内置的日志匹配规则，可自定义。
    }

}

output{
    #logstash做客户端时的配置，输出到elasticsearch，，本方案不采用，故注释掉。
    #if [type] == "apache_access" {
    #   elasticsearch{
    #       hosts => ["10.0.21.194:9200"]
    #       index => "apache-%{+YYYY.MM.dd}"
    #   }
    #}
    
    #syslog到elasticsearch
    if [type] == "syslog" {
        elasticsearch{
            hosts => ["192.168.1.1:9200"]
            index => "syslog-%{+YYYY.MM.dd}"
        }
    }
    #根据filebeat中service不同域名建立不同索引，输出到elasticsearch
    if [fields][service] == "www.a.com" {
        elasticsearch{
            hosts => ["192.168.1.1:9200"]
            index => "www.a.com-%{+YYYY.MM.dd}"
        }
    }
    #根据filebeat中service不同域名建立不同索引，输出到elasticsearch
    if [fields][service] == "www.b.com" {
        elasticsearch{
            hosts => ["192.168.1.1:9200"]
            index => "www.c.com-%{+YYYY.MM.dd}"
        }
    }
    #其他日志
    elasticsearch{
        hosts => ["192.168.1.1:9200"]
        index => "other-%{+YYYY.MM.dd}"
    }
    #输出到屏幕
    stdout{
        codec=>rubydebug
    }
}

#提示：如果你想在业务服务器上使用logstash做客户端,但服务器是低版本java，logstash又需要以JDk1.8以上运行，可在logstash/startup.option配置文件中的set参数指定高版本JAVACMD路径，亲测可用。
```
### 安装服务
在bin目录新建一个logstash-start.bat文件写入：

```
logstash.bat logstash.conf
```
在cmd中使用nssm安装服务，选择logstash-start.bat路径，并配置启动依赖于“elasticsearch-service”服务（确保开启自启动正常）：
```
nssm install logstash
```
### 关于filter的grok正则处理，单独说一下：
1.apahce日志
Logstash 默认自带了 apache 标准日志的 grok 正则:
```
COMMONAPACHELOG %{IPORHOST:clientip} %{USER:ident} %{NOTSPACE:auth} \[%{HTTPDATE:timestamp}\] "(?:%{WORD:verb} %{NOTSPACE:request}(?: HTTP/%{NUMBER:httpversion})?|%{DATA:rawrequest})" %{NUMBER:response} (?:%{NUMBER:bytes}|-)
COMBINEDAPACHELOG %{COMMONAPACHELOG} %{QS:referrer} %{QS:agent}
```
nginx 标准日志的 grok 正则定义是：
```
MAINNGINXLOG %{COMBINEDAPACHELOG} 
```

2.对于 nginx 标准日志格式
默认的nginx的log_format配置可能是下面这样：
```
log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                '$status $body_bytes_sent "$http_referer" '
                '"$http_user_agent" "$http_x_forwarded_for"';
```
nginx比Apache多了一个 $http_x_forwarded_for 变量，所以 nginx 标准日志的 grok 正则定义是：
```
MAINNGINXLOG %{COMBINEDAPACHELOG} %{QS:x_forwarded_for}
```
3.为了便于搜索日志，在可日志中加一个server name字段，也即网站的域名，适用于网站群的情况，便于区分不同网站日志。

```
log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                '$status $body_bytes_sent "$http_referer" '
                '"$http_user_agent" "$http_x_forwarded_for" $server_name';

```
添加server_name后的grok 正则定义：
```
MAINNGINXLOG %{COMBINEDAPACHELOG} %{QS:x_forwarded_for} %{HOSTNAME:server_name}
```
对于默认正则的修改可以这样在默认正则后面追加，前提是x_forwarded_for、server_name的正则在默认正则中是存在的：

```
match => {"message" => "%{COMBINEDAPACHELOG} %{QS:x_forwarded_for} %{HOSTNAME:server_name}" }

```
4.如何自定义规则。

默认的grok正则定义文件在下面路径：logstash/vendor/bundle/jruby/1.9/gems/logstash-patterns-core-0.3.0/patterns下。
如果要自定义新的，在logstah.conf中指定规则文件路径即可，自定义规则优先级大于默认规则，举例如下：
```
grok {
            patterns_dir => "/usr/local/logstash/patterns"        //设置自定义正则路径
            match => {
                        "message" => "%{NGINXACCESS}"
                    }
    }
```
## 5.3 kibana 安装配置

解压修改配置文件即可。

### 修改配置文件config/kibana.yml
因为采用了nginx进行本地代理，只允许localhost本地访问即可。
```
server.port: 5601
server.host: "localhost"
elasticsearch.url: "http://192.168.1.1:9200"
```

### 安装服务

在bin目录新建一个kibana-start.bat文件写入：

```
kibana.bat
```
在cmd中使用nssm安装服务，选择kibana.bat路径，并配置启动依赖于“logstash”服务（确保开机自启动正常）：
```
nssm install kibana
```


## 6、filebeat 安装配置

### filebeat安装
仅介绍linux下安装，windows同理。

> rpm -ivh filebeat-5.6.9-x86_64.rpm


### 修改配置文件filebeat.yml

```
filebeat.prospectors:
- input_type: log
  paths:
    - /usr/local/nginx/logs/*.log
  tags: ["nginx_access"]
  fields:
      service : www.a.com
output.logstash:
  hosts: ["192.168.1.1:54320"]
#参数说明：
#tages是标记日志类型，便于logstah进行filter格式化，对应logstah配置。
#service 是为了在logstash中根据不同网站发来的日志建立不同的Elasticsearch索引，方便日志查询，建议填写系统域名或别名，对应logstah配置。
#yml格式是根据缩进判断参数，相同缩进为一类参数，注意缩进。
```
### 自启动
> systemctl enable filebeat或chkconfig filebeat on

### logrotate 配置
由于我们将日志远程存储，本地日志就没有必要一直保留，占用存储空间，apahce自带日志分割功能，不再写具体方法了，默认情况下nginx、tomcat等产生的日志一直保存，我们可以使用logrotate分割日志，logrotate的配置文件在/etc/logrotate.conf。

举例配置如下：
/usr/local/nginx/logs/*.log位置是新添加的，前面的可注释掉。

```
# see "man logrotate" for details
# rotate log files weekly
weekly

# keep 4 weeks worth of backlogs
rotate 4

# create new (empty) log files after rotating old ones
create

# use date as a suffix of the rotated file
dateext

# uncomment this if you want your log files compressed
compress

# RPM packages drop log rotation information into this directory
include /etc/logrotate.d

# no packages own wtmp and btmp -- we'll rotate them here
#/var/log/wtmp {
#    monthly
#    create 0664 root utmp
#    minsize 1M
#    rotate 1
#}

#/var/log/btmp {
#    missingok
#    monthly
#    create 0600 root utmp
#    rotate 1
#}
/usr/local/nginx/logs/*.log{
daily
rotate 90
missingok
notifempty
dateext
compress
delaycompress
create 600 root root
sharedscripts
postrotate
    if [ -f /usr/local/nginx/logs/nginx.pid ]; then
        kill -USR1 `cat /usr/local/nginx/logs/nginx.pid`
    fi
endscript
}
# system-specific logs may be also be configured here.

```
logrotate主要参数如下表：
```
参数 功能
compress 通过gzip 压缩转储以后的日志
nocompress 不需要压缩时，用这个参数
copytruncate 用于还在打开中的日志文件，把当前日志备份并截断
nocopytruncate 备份日志文件但是不截断
create mode owner group 转储文件，使用指定的文件模式创建新的日志文件
nocreate 不建立新的日志文件
delaycompress 和 compress 一起使用时，转储的日志文件到下一次转储时才压缩
nodelaycompress 覆盖 delaycompress 选项，转储同时压缩。
errors address 专储时的错误信息发送到指定的Email 地址
ifempty 即使是空文件也转储，这个是 logrotate 的缺省选项。
notifempty 如果是空文件的话，不转储
mail address 把转储的日志文件发送到指定的E-mail 地址
nomail 转储时不发送日志文件
olddir directory 转储后的日志文件放入指定的目录，必须和当前日志文件在同一个文件系统
noolddir 转储后的日志文件和当前日志文件放在同一个目录下
prerotate/endscript 在转储以前需要执行的命令可以放入这个对，这两个关键字必须单独成行
postrotate/endscript 在转储以后需要执行的命令可以放入这个对，这两个关键字必须单独成行
daily 指定转储周期为每天
weekly 指定转储周期为每周
monthly 指定转储周期为每月
rotate count 指定日志文件删除之前转储的次数，0 指没有备份，5 指保留5 个备份
tabootext [+] list 让logrotate 不转储指定扩展名的文件，缺省的扩展名是：.rpm-orig, .rpmsave, v, 和 ~ 
size size 当日志文件到达指定的大小时才转储，Size 可以指定 bytes (缺省)以及KB (sizek)或者MB (sizem).
```


# 6. 配置nginx反向代理及认证
## 修改配置文件nginx.conf
```
    upstream kibana {
                server  127.0.0.1:5601;
        }
        
    upstream head {
                server  127.0.0.1:9100;
        }
    upstream elasticsearch {
                server  192.168.1.1:9200;           
        }
    server  {
                    listen          15601;
                    server_name     192.168.1.1;
                    location / {
                        proxy_set_header Host $host;
                        proxy_set_header X-Real-IP $remote_addr;
                        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                        
                        #配置帐号密码,在linux上使用命令，生成一个密码文件#命令：htpasswd -bc ./password admin 123456
                        #复制到windows上D:\elk\head-nginx\conf\password
                        auth_basic "login";#提示信息
                        auth_basic_user_file D:\elk\head-nginx\conf\password; #密码文件（写绝对路径，注意\n转义）
                        autoindex on;
                        proxy_pass http://kibana;
                        
                    }
                    error_log           logs/kinaba._route_error.log;
                    access_log          logs/kinaba._route_access.log main;    

            }
    server  {
                    listen          19100;
                    server_name     192.168.1.1 localhost;
                    
                    #配置帐号密码，同上
                    auth_basic "login";#提示信息
                    auth_basic_user_file D:\elk\head-nginx\conf\password; #密码文件（写绝对路径，别相对路径，注意转义）
                    autoindex on;
                    
                    location / {
                        proxy_pass http://head/;
                        proxy_set_header Host $host;
                        proxy_set_header X-Real-IP $remote_addr;
                        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                    }
                    #这里使用http://192.168.1.1:19100/elasticsearch/请求9200服务，因此在head插件中配置elasticsearch服务路径是http://192.168.1.1:19100/elasticsearch/。
                    location /elasticsearch/ {
                        proxy_pass http://elasticsearch/;
                        proxy_set_header Host $host;
                        proxy_set_header X-Real-IP $remote_addr;
                        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                    }
                    error_log           logs/kinaba._route_error.log;
                    access_log          logs/kinaba._route_access.log main;    

            }

```
## 安装服务

在根目录新建一个nginx-start.bat文件写入：

```
nginx.exe
```
在cmd中使用nssm安装服务，选择nginx.exe路径：
```
nssm install nginx
```

---
参考连接：

> https://blog.csdn.net/yx1214442120/article/details/55102298
>
> https://blog.csdn.net/pilihaotian/article/details/52452010

---

本文发表在freebuf的连接：
http://www.freebuf.com/security-management/179736.html
---
本文为原创：转载请注明:https://wiki.viewcn.cn/