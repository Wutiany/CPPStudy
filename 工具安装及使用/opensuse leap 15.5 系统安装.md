# opensuse leap 15.5 系统安装

## 使用安装镜像安装系统

**按顺序进行**

* 安装位置：SanDisk
* 分区部分：使用默认分区，不进行分区
* 安装类型选择：服务器，文本类型
* 本地用户创建：不创建本地用户
* root 用户设置：
  * 密码：pintechs
* 软件安装：选择帮助文档
* ssh 服务：开启 ssh 服务，不使用已存在的 ssh 密钥
* 安装完成，重启系统

## 换源

* 官方源不可用，如果使用需配置代理

* 改用清华源：

  禁用官方软件源

  ``` shell
  sudo zypper mr -da
  ```

  添加镜像源

  ```shell	
  sudo zypper ar -cfg 'https://mirrors.tuna.tsinghua.edu.cn/opensuse/distribution/leap/$releasever/repo/oss/' mirror-oss
  sudo zypper ar -cfg 'https://mirrors.tuna.tsinghua.edu.cn/opensuse/distribution/leap/$releasever/repo/non-oss/' mirror-non-oss
  sudo zypper ar -cfg 'https://mirrors.tuna.tsinghua.edu.cn/opensuse/update/leap/$releasever/oss/' mirror-update
  sudo zypper ar -cfg 'https://mirrors.tuna.tsinghua.edu.cn/opensuse/update/leap/$releasever/non-oss/' mirror-update-non-oss
  ```

  刷新软件源

  ``` shell
  sudo zypper ref
  ```

## ssh 服务启动

* 服务器端配置 ssh 服务

  如果未安装 ssh，需安装 openssh 包

  ```shell
  sudo zypper install --no-confirm openssh
  ```

  通过 systemd 开启 ssh 服务

  ```shell
  sudo systemctl start sshd
  ```

  查看 ssh 状态

  ```shell
  sudo systemctl status sshd
  ```

  在系统启动期间启用 sshd

  ```shell
  sudo systemctl enable sshd
  ```

  为 ssh 启用防火墙规则

  ```shell
  sudo firewall-cmd --permanent --add-service=ssh
  ```

  重新加载防火墙规则

  ```shell
  sudo firewall-cmd --reload
  ```

* 客户端通过 ssh 连接服务器

  ```shell
  ssh username@serverip
  // ssh root@192.168.30.168
  ```

  首次登陆需要确认 yes

  然后输入密码：pintechs

## 服务器安装 docker

* 可以不进行的操作

  安装更新

  ```shell
  sudo zypper refresh
  sudo zypper update -y
  ```

  重启系统

  ```shell
  sudo reboot
  ```

* 使用命令安装 docker

  ```shell
  sudo zypper install -y docker
  ```

* 启动并启用 docker 服务

  ```shell
  sudo systemctl start docker
  sudo systemctl enable docker
  ```

  验证 docker 服务状态

  ```shell
  sudo systemctl status docker
  ```

  查看 docker 版本

  ```shell
  sudo docker version
  ```

  允许本地用户不使用 sudo 运行 docker 命令，将本地用户添加到 docker 组

  ```shell
  sudo usermod -aG docker $USER
  newgrp docker
  ```

* 验证 docker 安装

  ```shell
  docker run hello-world
  ```

* 卸载 docker 命令

  ```shell
  sudo zypper remove docker
  sudo rm -rf /var/lib/docker
  sudo rm -rf /var/lib/containerd
  ```

