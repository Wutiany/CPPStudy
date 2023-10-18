# VSCode 配置 docker 远程开发环境

## Dockerfile

```dockerfile
FROM ubuntu:22.04

RUN apt update
RUN apt install -y vim && apt install -y curl
RUN apt install -y git
RUN apt install -y net-tools
RUN apt install -y openssh-server
RUN apt install -y build-essential
RUN apt install -y gdb
RUN apt install -y g++
RUN apt install -y gcc
RUN apt install -y golang
RUN apt install -y make
RUN echo "root:password" | chpasswd
RUN ssh-keygen -A
RUN mkdir -p /var/run/sshd

EXPOSE 22 8080
```

## VScode 安装 ssh 插件，以及 C++ 相关插件

* windows 生成 ssh 的一对公钥和私钥文件

  ```shell
  $ ssh-keygen -t rsa
  ```

* 公钥私钥生成路径：C:\Users\电脑用户名\\.ssh\id_rsa.pub  `id_rsa.pub` 公钥文件

* 将公钥复制到服务器中

  * 使用命令（windows下不行）

    ```shell
    $ ssh-copy-id -i id_rsa.pub root@127.0.0.1
    ```

  * 直接复制到映射的文件夹中

* 如果系统不中不存在 `~/.ssh/authorized_keys` 路径和文件，就进行创建

* 然后将复制过去的公钥内容发送到  `authorized_keys` 文件中，**再次连接则无需使用密码**

  ```shell
  $ cat id_isa.pub > ~/.ssh/authorized_keys
  ```

* 服务器插件安装
* 产生 `c_cpp-properties.json` 文件



