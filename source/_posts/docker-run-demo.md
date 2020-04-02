---
title: docker-run-demo
date: 2020-04-02 23:59:44
tags:
  - docker
categories:
  - 技术
---

> 使用Docker可以快速搭建你的开发环境，以下是我经常用到的几个常用软件。文章的最后会总结下使用docker run的套路。

默认的，Docker会从官方的 Docker Hub 拉取镜像，国内用户想要提升访问 Docker Hub 拉取镜像的速度及稳定性，需要配置镜像站，这里使用的是DaoCloud的镜像站<br />以Linux系统为例：
```
$ curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s http://f1361db2.m.daocloud.io
```
更多详情，请访问：https://www.daocloud.io/mirror<br />

## MySQL
#### 1. 运行MySQL容器
```
$ docker run -d -p 3306:3306 --name mysql-5.7.5 \
-v /opt/docker/mysql/data:/var/lib/mysql \
-v /opt/docker/mysql/conf:/etc/mysql/conf.d \
-v /opt/docker/mysql/logs:/logs \
-v /usr/share/zoneinfo/Asia/Shanghai:/etc/localtime \
-e MYSQL_ROOT_PASSWORD=yourpassword \
daocloud.io/library/mysql:5.7.5
```
其中，

- -v：宿主机和容器的目录映射关系；
- ":"前为宿主机目录，之后为容器目录；
- 这里分别将MySQL的数据、配置和日志目录挂载到了宿主机。



#### 2. 修改配置
MySQL的配置文件`my.cnf`会读取容器内目录/etc/mysql/conf.d下的配置文件
```
# 新建配置文件
$ vim /opt/docker/mysql/conf/docker.cnf
```

docker.cnf文件如下：
```
[mysqld]
skip-host-cache
skip-name-resolve
lower_case_table_names=1
character-set-server=utf8 

[client]
default-character-set=utf8 
[mysql]
default-character-set=utf8
```
其中`lower_case_table_names=1`表示：表名存储在磁盘是小写的，但是比较的时候是不区分大小写<br />
<br />重启MySQL容器，使配置生效
```
$ docker restart mysql-5.7.5
```


## Redis
Redis 是一个开源，基于内存的高性能 key-value 数据库。
#### 1. 运行Redis容器
```
$ docker run -d -p 6379:6379 --name redis-6379 \
-v /opt/docker/redis/data:/data \
-v /usr/share/zoneinfo/Asia/Shanghai:/etc/localtime \
daocloud.io/library/redis:5.0 \
redis-server --appendonly yes --requirepass yourpassword
```
这里使用了Redis AOF持久化，同时将`appendonly.aof`文件挂载到了`/opt/docker/redis/data`目录下。<br />

#### 2. 重写aof文件
当用的时间长了，文件`appendonly.aof`会越来越大，使用`BGREWRITEAOF`命令可以对redis的AOF进行重写
```
# 伪终端以交互式的方式进入容器
$ docker exec -it redis-6379 bash
root@1ff606aa6ad6:/data# redis-cli
127.0.0.1:6379> auth yourpassword
OK
127.0.0.1:6379> BGREWRITEAOF
Background append only file rewriting started
```


## Nginx
Nginx 是一款轻量级的 Web 服务器、反向代理服务器、及电子邮件（IMAP/POP3）代理服务器。一般使用本地安装的方式安装nginx，但我比较喜欢用docker运行，真香！
#### 1. 运行Nginx容器
```
$ docker run --name nginx-1.17.9 -d -p 80:80 \
-v /opt/docker/nginx/html:/etc/nginx/html \
-v /opt/docker/nginx/log:/var/log/nginx:rw \
-v /opt/docker/nginx/config/conf.d:/etc/nginx/conf.d \
-v /usr/share/zoneinfo/Asia/Shanghai:/etc/localtime \
daocloud.io/library/nginx:1.17.9
```


#### 2. 配置
新建nginx配置文件
```
$ vim /opt/docker/nginx/config/conf.d/dudiao.conf
```

`dudiao.conf`文件如下：
```nginx
server {
    # nginx服务器端口号
    listen       80;
    server_name  192.168.199.210;

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
      root   html;
    }
}
```
重启nginx，使配置生效
```
# 重启nginx
$ docker restart nginx-1.17.9

$ cd /opt/docker/nginx/html
$ touch index.html && echo 'hello world!' >> index.html
```

<br />访问`ip:80`即可看到刚刚输入的 hello world!<br />

## RabbitMQ
RabbitMQ 是开源的消息队列系统（或称消息中间件）<br />

#### 运行RabbitMQ容器
```
$ docker run -d --name rabbitmq-3.7.24 --hostname my-rabbit \
-p 5672:5672 -p 15672:15672 \
-v /usr/share/zoneinfo/Asia/Shanghai:/etc/localtime \
daocloud.io/library/rabbitmq:3.7.24-management
```
RabbitMQ 已经有一些自带管理插件的镜像。用这些镜像创建的容器实例可以直接使用默认的 15672 端口访问，默认账号密码是`guest/guest`<br />

## Neuxs
Sonatype Nexus是一个Maven私服软件
```
# 拉取镜像，时间会有点长，642M
$ docker pull sonatype/nexus3:3.22.0

$ mkdir -p /opt/docker/nexus/data && chown -R 200 /opt/docker/nexus/data

# 这里运行nexus3的时区还有点问题（会相差8个多小时）
$ docker run -d -p 8081:8081 --name nexus-3.22.0 \
-v /opt/docker/nexus/data:/nexus-data \
sonatype/nexus3:3.22.0

# 查看日志
$ docker logs -f nexus-8081

...
-------------------------------------------------

Started Sonatype Nexus OSS 3.22.0-02

-------------------------------------------------
```

<br />访问`ip:8081`，点击右上角 Sign in 按钮，进行登录，账号为：admin，密码在 /opt/docker/nexus/data/admin.password中。<br />
<br />关于Nexus3的使用，有机会开一章单独说下。

## Gitlab
Gitlab是一个用于仓库管理系统的开源项目，使用Git作为代码管理工具，并在此基础上搭建起来的web服务。官方推荐需要至少4G的内存。<br />

#### 1. 运行Gitlab容器
```
# 时间会比较久，共1.89GB
$ docker pull gitlab/gitlab-ce:12.8.8-ce.0

$ docker run --detach \
  --publish 10443:443 --publish 10080:80 --publish 10022:22 \
  --name gitlab-ce-12.8.8-ce.0 \
  --volume /opt/docker/gitlab/config:/etc/gitlab \
  --volume /opt/docker/gitlab/logs:/var/log/gitlab \
  --volume /opt/docker/gitlab/data:/var/opt/gitlab \
  -v /usr/share/zoneinfo/Asia/Shanghai:/etc/localtime \
  gitlab/gitlab-ce:12.8.8-ce.0
 
 # 查看日志，启动需要几分钟
 $ docker logs -f gitlab-ce-12.8.8-ce.0
```
耐心等待一段时间后，访问`ip:10080/gitlab`可以测试是否启动成功。<br />![docker-gitlab-1.png](https://cdn.nlark.com/yuque/0/2020/png/726269/1585670426116-c16ff9ff-8efc-4a14-8b27-1bf995ce044c.png#align=left&display=inline&height=566&name=docker-gitlab-1.png&originHeight=659&originWidth=1342&size=53283&status=done&style=none&width=1152)<br />出现该页面时，则启动成功，先别着急填写密码，咱们需要做一些配置。<br />

#### 2. 修改配置文件
配置文件路径为：
/opt/docker/gitlab/config/gitlab.rb

- 修改邮箱配置（可选）
```
### Email Settings
gitlab_rails['smtp_enable'] = true
gitlab_rails['smtp_address'] = "smtp.exmail.qq.com"
gitlab_rails['smtp_port'] = 465
gitlab_rails['smtp_user_name'] = "admin@gits.xyz"
gitlab_rails['smtp_password'] = "your_smtp_password"
gitlab_rails['smtp_domain'] = "exmail.qq.com"
gitlab_rails['smtp_authentication'] = "login"
gitlab_rails['smtp_enable_starttls_auto'] = true
gitlab_rails['smtp_tls'] = true
# If your SMTP server does not like the default 'From: gitlab@localhost' you
# can change the 'From' with this setting.
gitlab_rails['gitlab_email_from'] = 'admin@gits.xyz'
gitlab_rails['gitlab_email_reply_to'] = 'admin@gits.xyz'
```


- 修改external_url（可选）

如果你想用域名访问gitlab服务，可以更改external_url的值
```
## GitLab URL
##! URL on which GitLab will be reachable.
##! For more details on configuring external_url see:
##! https://docs.gitlab.com/omnibus/settings/configuration.html#configuring-the-external-url-for-gitlab
external_url 'http://gits.xyz'
```


- 修改ssh端口
```
### GitLab Shell settings for GitLab
gitlab_rails['gitlab_shell_ssh_port'] = 10022
```


#### 3. 刷新配置
使用`docker restart`这样的方式来重启容器达到配置生效的效果，这样做非常不安全，配置文件可能只有部分生效而且很可能造成gitlab的完全崩溃，无法启动。

使用`gitlab-ctl reconfigure`命令刷新配置
```
$ docker exec -it gitlab-ce-12.8.8-ce.0 /bin/bash

root@0d8135776b45:/# gitlab-ctl reconfigure
# 出现如下界面，则重载配置成功
gitlab Reconfigured!
root@0d8135776b45:/# exit
```

<br />然后重启docker就可以了
```
$ docker restart gitlab-ce-12.8.8-ce.0
```


#### 4. 测试

- 测试邮箱（可选）
```
$ docker exec -it gitlab-ce-12.8.8-ce.0 /bin/bash
root@0d8135776b45:/# gitlab-rails console
--------------------------------------------------------------------------------
 GitLab:       12.8.8 (6ea04b16a40) FOSS
 GitLab Shell: 11.0.0
 PostgreSQL:   10.12
--------------------------------------------------------------------------------
Loading production environment (Rails 6.0.2)
irb(main):001:0> Notify.test_email('idudiao@163.com', '测试邮件标题', '测试邮件正文').deliver_now
```
![微信图片_20200401002634.jpg](https://cdn.nlark.com/yuque/0/2020/jpeg/726269/1585672025962-70ab9fd6-e8a3-4dcc-96f1-689b283a250d.jpeg#align=left&display=inline&height=498&name=%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20200401002634.jpg&originHeight=902&originWidth=1242&size=129439&status=done&style=none&width=686)<br />收到邮件，Get!<br />

- 初次登陆时，需要访问`ip:10080` 修改root密码

然后创建个项目试试<br />![gitlab-project.png](https://cdn.nlark.com/yuque/0/2020/png/726269/1585672237949-2d27e75b-62a3-4278-9660-7e347289493d.png#align=left&display=inline&height=618&name=gitlab-project.png&originHeight=618&originWidth=973&size=63497&status=done&style=none&width=973)可以看到咱之前设置的external_url和ssh端口已经生效，Get！

## 总结
每个镜像如何使用，一般可以在`https://hub.docker.com/`会有详细的描述。比如MySQL<br />![image.png](https://cdn.nlark.com/yuque/0/2020/png/726269/1585716468053-3faeccbf-24b6-44dc-9035-33b79da38380.png#align=left&display=inline&height=471&name=image.png&originHeight=471&originWidth=821&size=43637&status=done&style=none&width=821)<br />
<br />官方会给出详细介绍<br />![image.png](https://cdn.nlark.com/yuque/0/2020/png/726269/1585716522914-b0d20bbf-48f1-4bab-9edf-69c6bcff0b40.png#align=left&display=inline&height=679&name=image.png&originHeight=679&originWidth=1210&size=99296&status=done&style=none&width=1210)
当你使用docker镜像遇到困惑时，可以试着查看这些文档。<br />
<br />下面总结下上面运行的几个镜像的关键点：

- 暴露端口：`-p`
- 挂载文件（数据、配置、日志）：`-v`
- 设置参数：`-e`
- 修改配置文件

而这些关键点和如何构建docker镜像又有着关联，感兴趣的可以了解下`docker build`命令，构建一个自己的镜像。


![长按关注【读钓的YY】](https://imgkr.cn-bj.ufileos.com/ba7d6b7a-eec8-4382-82a4-506398dd94db.png)

博客不定时更新，关注我的公众号，可以及时获取最新文章。