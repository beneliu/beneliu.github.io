# Nacos集群搭建


{{&lt; admonition type=info title=&#34;内容简介&#34; open=true &gt;}}
详细介绍nacos集群搭建步骤和过程。
{{&lt; /admonition &gt;}}

## 环境准备

本文采用的Linux系统为 Rocky Linux release 8.10 版本，nacos版本为 2.2.3，数据库使用mysql的 5.7.43 版本。
| 服务器    | 节点功能    | 软件包版本   |
| --------- | ----------- | ------ |
| 10.0.0.10 | nacos节点1  | 2.2.3  |
| 10.0.0.11 | nacos节点2  | 2.2.3  |
| 10.0.0.12 | nacos节点3  | 2.2.3  |
| 10.0.0.13 | mysql数据库 | 5.7.43 |

在10.0.0.10、10.0.0.11、10.0.0.12服务器安装java1.8：
```bash
# 安装java1.8
yum list java* --showduplicate | sort -r
yum install -y java-1.8.0-openjdk-devel.x86_64
java -version  #查看java版本
```
在10.0.0.13服务器安装mysql：
```bash
# 安装mysql 5.7.43
rpm -ivh https://dev.mysql.com/get/mysql57-community-release-el7-11.noarch.rpm
sed -i &#39;s/gpgcheck=1/gpgcheck=0/&#39; /etc/yum.repos.d/mysql-community.repo
yum -y install mysql-community-server
systemctl enable --now mysqld
password=`sed -n &#39;/temporary password/s/.*root@localhost: \(.*\)/\1/p&#39; /var/log/mysqld.log`
mysql -uroot -p&#34;$password&#34; --connect-expired-password -e&#34;set global validate_password_policy=0;&#34;
mysql -uroot -p&#34;$password&#34; --connect-expired-password -e&#34;set global validate_password_length=6;&#34;
mysql -uroot -p&#34;$password&#34; --connect-expired-password -e&#34;alter user &#39;root&#39;@&#39;localhost&#39; identified by &#39;123456&#39;;&#34;
```
这里将mysql密码策略设置为简单，仅仅为了演示方便，生产不建议这么做。同时生产中nacos一般用作注册中心和配置中心，所以nacos用的数据库最好使用集群方式。

## 安装nacos集群

### 安装nacos

[官网下载安装包](https://nacos.io/download/release-history/?spm=5238cd80.2ef5001f.0.0.3f613b7c5qOhNG)，然后上传到10.0.0.10、10.0.0.11、10.0.0.12服务器。在这三台服务器上执行：
```bash
# 将nacos解压到/usr/local/目录下
tar xf nacos-server-2.2.3.tar.gz -C /usr/local/
```
在10.0.0.13服务器上配置nacos使用的mysql数据库：
```bash
# 提前将nacos数据sql文件复制到本机的/tmp目录下，sql文件在nacos安装目录的conf目录下：/usr/local/nacos/conf/
mysql -uroot -p123456
create database nacos_config default character set utf8;
create user &#39;nacos&#39;@&#39;%&#39; identified by &#39;123456&#39;;  #演示用，生产请保证密码复杂度
grant all privileges on nacos_config.* to &#39;nacos&#39;@&#39;%&#39;;
flush privileges;
use nacos_config;
source /tmp/mysql-schema.sql;  #执行nacos数据库脚本
show tables;  #查看创建的表
```

### 配置并启动nacos

```bash
vim /usr/local/nacos/conf/application.properties
spring.sql.init.platform=mysql
db.num=1
db.url.0=jdbc:mysql://10.0.0.13:3306/nacos_config?characterEncoding=utf8&amp;connectTimeout=1000&amp;socketTimeout=3000&amp;autoReconnect=true&amp;useUnicode=true&amp;useSSL=false&amp;serverTimezone=UTC
db.user.0=nacos
db.password.0=123456
management.endpoints.web.exposure.include=prometheus  #开启prometheus监控
nacos.core.auth.enabled=true  #开启权限认证
nacos.core.auth.system.type=nacos
nacos.core.auth.plugin.nacos.token.secret.key=${自定义，保证所有节点一致}
nacos.core.auth.server.identity.key=${自定义，保证所有节点一致}
nacos.core.auth.server.identity.value=${自定义，保证所有节点一致}

cd /usr/local/nacos/conf/
cp -a cluster.conf.example cluster.conf
vim cluster.conf
10.0.0.10:8848
10.0.0.11:8848
10.0.0.12:8848

# 将配置文件复制到另外两个主机
scp /usr/local/nacos/conf/application.properties 10.0.0.11:/usr/local/nacos/conf/
scp /usr/local/nacos/conf/cluster.conf 10.0.0.11:/usr/local/nacos/conf/
scp /usr/local/nacos/conf/application.properties 10.0.0.12:/usr/local/nacos/conf/
scp /usr/local/nacos/conf/cluster.conf 10.0.0.11:/usr/local/nacos/conf/

# 依次启动三台服务器上的nacos
sh /usr/local/nacos/bin/startup.sh
tail -f /usr/local/nacos/logs/start.out  #查看启动日志，如果日志最后显示如下，则成功
...
2023-08-19 15:55:33,901 INFO Nacos started successfully in cluster mode. use external storage
```

## nacos集群访问

使用nginx作为负载均衡来访问nacos集群，nginx配置如下：
```nginx
upstream nacos-cluster {
        server 10.0.0.9:8848;
        server 10.0.0.10:8848;
        server 10.0.0.12:8848;
}

server {
    listen       80;
    server_name  www.nacos.com;

    #access_log  /var/log/nginx/host.access.log  main;

    location / {
         #root   html;
         #index  index.html index.htm;
         proxy_pass http://nacos-cluster/nacos/;  #&#39;/&#39;斜杠不能去掉
    }
}
```
正如上面所说，nacos用作注册中心和配置中心，是非常重要的服务，不仅数据库要使用集群来避免单点故障，用来反向代理nacos集群的nginx也需要高可用部署。推荐使用nginx&#43;keepalived来实现，不过这里就不再演示了。

**参考资料：**
1. [Nacos官网](https://nacos.io/)

---

> 作者: [beneliu](https://github.com/beneliu)  
> URL: https://beneliu.github.io/posts/devops/338be2a/  

