# docker 安装
## window 安装 docker desktop
* 首先安装 WSL（windows子系统），详见网络教程
* 然后安装 docker desktop 详见网上安装教程
## 安装 docker 镜像
### 镜像网站
https://hub.docker.com/
* 安装 gcc 为基础的镜像，搜索 gcc 即可，找到其中的版本，然后写自己的 Dockerfile
### 通过 Dockerfile 构建镜像
* 首先写自己的 Dockerfile，直接使用梯子不需要更改镜像源
```dockerfile
FROM gcc:10.5.0

MAINTAINER ty

#RUN mv /etc/apt/sources.list /etc/apt/sources.list.bak  // 这部分是使用国内源进行更新软件
#ADD sources.list /etc/apt/         // 将本地的sources.list 移动到镜像中
RUN apt-get update
RUN apt-get install -y net-tools --fix-missing
RUN apt-get install -y vim --fix-missing
RUN apt-get install -y openssh-server --fix-missing
RUN apt-get install -y build-essential --fix-missing
RUN apt-get install -y gdb --fix-missing
RUN apt-get install -y git --fix-missing
RUN echo "root:password"|chpasswd
RUN ssh-keygen -A
RUN mkdir -p /var/run/sshd

EXPOSE 22 8080
```
这部分我在更换源仍旧执行 RUN apt-get update 一直停住，使用网上写在 Dockerfile 中的换源代码，在执行 apt-get 的时候出错
* 当前文件夹下写 sources.list
## 运行容器
```shell
docker run -it -p 22:22 -p 8080:8080 -v F:\project:/root/project --name gcc gcc
```
* 命令含义：
  * -it：表示以交互式（Interactive）方式运行容器，并分配一个伪终端（TTY）。
  * -p 22:22：表示将容器内部的22端口映射到宿主机的22端口，用于SSH连接。
  * -p 8080:8080：表示将容器内部的8080端口映射到宿主机的8080端口，用于应用程序访问。
  * -v D:\yazi\yazi-web:/root/yazi-web：表示将本地目录D:\yazi\yazi-web映射到容器内部的/root/yazi-web目录，用于在容器中访问本地文件。
  * --name gcc：表示指定容器的名称为gcc。
  * gcc：表示指定运行的镜像名称为gcc。
## 启动容器的 SSH 服务用以 VScode 进行远程连接
* 修改 sshd 配置文件，允许 root 用户登陆
```shell
vim /etc/ssh/sshd_config

// 修改参数
PermitRootLogin yes
```
* 开启 ssh 服务
```shell
/usr/sbin/sshd
```
## 配置 VScode
* 安装 ssh 插件
* 安装后在远程连接窗口增加一个连接，使用命令连接
```shell
ssh root@127.0.0.1 -A
```
* 右下方新建终端，和打开远程的文件夹
* 安装 C++ 插件