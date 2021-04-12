---
title: docker 安装及常用命令
date: 2020-05-23 22:06:07
categories : docker
tags: 
	- docker
---

## 环境安装（Linux）

### 移除已有docker

已经安装过docker的，需要进行卸载，卸载的命令官网截图如下：相关的命令内容也会贴出来，我本机刚刚安装的虚拟机，所以跳过这步

<!-- more -->

```shell
$ sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-selinux \
                  docker-engine-selinux \
                  docker-engine

```

###  安装条件

也就是内核版本，必须是3.10及以上，可以通过uname -r命令检查内核版本

### 安装命令

第一步

```shell
yum install -y yum-utils device-mapper-persistent-data lvm2
```

第二步

建议使用阿里云的地址，国外的地址，下载比较慢，而且很容易链接超时什么的，两个地址，我都贴出来了

```shell
官网地址
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
##阿里云地址
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

```

第三步

正式安装

```shell
yum install docker-ce
```

第四步

启动docker 以及测试

```shell
systemctl start docker
 
docker run hello-world
```

### 配置阿里云加速器

CentOs:

修改配置文件：

```shell
vi usr/lib/systemd/system/docker.service
```

添加加速器地址 : --registry-mirror=https://as31ovo6.mirror.aliyuncs.com

```shell
ExecStart=/usr/bin/dockerd --registry-mirror=your accelerate address
```

```shell
重新加载配制：$ systemctl daemon-reload
重新启动服务：$ systemctl restart docker
```

查看是否配置成功

```shell
ps -ef|grep docker
```



## Docker-Compose

```shell
# 下载安装
curl -L --fail https://github.com/docker/compose/releases/download/1.25.4/run.sh -o /usr/local/bin/docker-compose
# 授权
chmod +x /usr/local/bin/docker-compose
# 验证是否安装成功
docker-compose version


# 卸载
rm /usr/local/bin/docker-compose
```



## docker 常用命令

+ 列出所有的容器 ID

```shell
docker ps -aq
```

+ 查看所有运行或者不运行容器

```shell
docker ps -a
```

+ 停止所有的容器

```shell
docker stop $(docker ps -a -q) 或者 docker stop $(docker ps -aq) 
```

+ 删除所有的容器

```shell
docker rm $(docker ps -aq)
```

+ 强制终止并删除一个运行的容器

```shell
docker rm -f 容器ID
```

+ 删除images（镜像），通过image的id来指定删除谁

```shell
docker rmi <image id>
```

+ 删除所有的镜像

```shell
docker rmi $(docker images -q)
```

+ 强制删除全部镜像

```shell
docker rmi -f $(docker images -q)
```

+ 从容器到宿主机复制

```shell
docker cp tomcat：/webapps/js/text.js /home/admin
docker  cp 容器名:  容器路径       宿主机路径    
```

+ 从宿主机到容器复制

```shell
docker cp /home/admin/text.js tomcat：/webapps/js
docker cp 宿主路径中文件      容器名  容器路径   
```

