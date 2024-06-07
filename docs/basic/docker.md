## Docker

### 镜像操作命令

搜索仓库镜像

```
docker search 镜像名
```

拉取镜像

```
docker pull 镜像名
```

查看镜像

```
docker images
```

删除镜像

```
docker rmi 镜像ID
```

### 容器操作命令

查看正在运行的容器

```
docker ps
```

查看所有容器

```
docker ps -a
```

启动（停止的）容器

```
docker start 容器ID
```

停止容器

```
docker stop 容器ID
```

重启容器

```
docker restart 容器ID
```

启动（新）容器

```
docker run -it 镜像ID /bin/bash

docker run -d --privileged --name 容器名 镜像ID
```

进入容器

```
docker exec -it 容器ID /bin/bash
# 或者（推荐使用前者）
docker attach 容器ID
```

删除容器

```
docker rm 容器ID
```

### 安装

安装

```
# 一键安装
curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun
curl -sSL https://get.daocloud.io/docker | sh

# 手动安装
yum install yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
# http://mirror.centos.org/centos/7/extras/x86_64/Packages/
yum install http://mirror.centos.org/centos/7/extras/x86_64/Packages/container-selinux-2.107-3.el7.noarch.rpm
yum install http://mirror.centos.org/centos/7/extras/x86_64/Packages/slirp4netns-0.4.3-4.el7_8.x86_64.rpm
yum install http://mirror.centos.org/centos/7/extras/x86_64/Packages/fuse-overlayfs-0.7.2-6.el7_8.x86_64.rpm
yum install docker-ce docker-ce-cli containerd.io
# yum install docker-compose-plugin
```

启动

```
systemctl start docker

docker pull hello-world
docker run hello-world
```



### 其他命令

```
docker compose -f docker-compose.yml up

docker exec -it spark-iceberg spark-sql


Tue Dec 27 07:08:41 UTC 2022

find . -name '*' -newermt "2022-12-27 07:00:00" ! -newermt "2022-12-27 07:10:00"
```

### 引用

- https://mp.weixin.qq.com/s?__biz=MjM5NTY1MjY0MQ==&mid=2650860524&idx=3&sn=02dfc31d637f70b066a6ef9842beeac5&chksm=bd017ea28a76f7b466773e68f7dab26e65ffae2918c28aa1d87c84acfc54460a7b82aa57279f&scene=27