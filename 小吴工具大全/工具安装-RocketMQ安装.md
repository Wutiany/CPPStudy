# RocketMQ 安装

# 1 镜像拉取

**两个镜像**

* apche/rocketmq
* styletang/rocketmq-console-ng

# 2 容器创建

## 2.1 rmqnamesrv 容器创建

* **创建 rmqnamesrv 容器的文件挂载路径**

  * `mkdir -p /home/rocketmq/data/namesrv/logs /home/rocketmq/data/namesrv/store`
  * 这里使用 `/home` 进行挂载，因为**只读文件系统**会无法挂载 `/usr/local` 文件夹，**因此报错**

* **构建 rmqnamesrv 容器**

  * ```shell
    docker run -d \
    --restart=always \
    --name rmqnamesrv \
    --privileged=true \
    -p 9876:9876 \
    -v /home/rocketmq/data/namesrv/logs:/root/logs \
    -v /home/rocketmq/data/namesrv/store:/root/store \
    -e "MAX_POSSIBLE_HEAP=100000000" \
    -e "JAVA_OPT_EXT=-Xms512M -Xmx512M -Xmn128m" \
    apache/rocketmq \
    sh mqnamesrv
    ```

## 2.2 borker 节点创建

* **创建 broker 数据挂载路径**

  * `mkdir -p /home/rocketmq/data/broker/logs /home/rocketmq/data/broker/store /home/rocketmq/conf`

* **编辑配置文件**

  * `vim /home/rocketmq/conf/broker.conf`

  * ```txt
    brokerClusterName = DefaultCluster
    brokerName = broker-a
    brokerId = 0
    deleteWhen = 04
    fileReservedTime = 48
    brokerRole = ASYNC_MASTER
    flushDiskType = ASYNC_FLUSH
    brokerIP1 = 192.168.0.1
    diskMaxUsedSpaceRatio=95
    ```

  * `brokerIP1` **修改**成自己的 IP

* **构建 broker 容器**

  * ```shell
    docker run -d \
    --restart=always \
    --name rmqbroker \
    --link rmqnamesrv:namesrv \
    --privileged=true \
    -p 10911:10911 \
    -p 10912:10912 \
    -p 10909:10909 \
    -v /home/rocketmq/data/broker/logs:/root/logs \
    -v /home/rocketmq/data/broker/store:/root/store \
    -v /home/rocketmq/conf/broker.conf:/home/rocketmq/rocketmq-5.1.4/conf/broker.conf \
    -e "NAMESRV_ADDR=namesrv:9876" \
    -e "JAVA_OPT_EXT=-Xms512M -Xmx512M -Xmn128m" \
    -e "MAX_POSSIBLE_HEAP=200000000" \
    apache/rocketmq \
    sh mqbroker -c /home/rocketmq/rocketmq-5.1.4/conf/broker.conf
    ```

  * 修改 `rocketmq` 的**版本**，需要看自己安装的是什么版本，上面命令中的文件路径带版本的要**全部替换**成自己的

## 2.3 rockermq-console 服务创建

* **构建 rockermq-console 容器**

  * ```shell
    docker run -d \
    --restart=always \
    --name rmqadmin \
    -e "JAVA_OPTS=-Drocketmq.namesrv.addr=192.168.0.1:9876 -Dcom.rocketmq.sendMessageWithVIPChannel=false" \
    -p 8081:8080 \
    --ulimit nofile=1024 \
    styletang/rocketmq-console-ng:latest
    ```

  * 修改 `JAVA_OPTS=-Drocketmq.namesrv.addr=` 后面的 IP，修改成自己本机的 IP 地址

# 3 启动服务

* **启动 rmqnamesrv 容器**
  * `docker start rmqnamesrv`
* **启动 broker 容器**
  * `docker start rmqbroker`

# 4 查看 rmq 的控制台

* rockermq-console 提供可视化平台
* 访问其中配置的地址  192.168.0.1:8081



[参考博客]([基于Docker安装RockerMQ【保姆级教程、内含图解】_docker安装apache/rocketmq-CSDN博客](https://blog.csdn.net/Acloasia/article/details/130548105))