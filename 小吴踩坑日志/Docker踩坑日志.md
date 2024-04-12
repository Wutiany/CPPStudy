# Docker 踩坑日志

## 容器间通信

* 使用 `networks`，放在**一个网段**
* `rpc` 需要 `ip` 地址的，不仅需要放在一个**网段**，还需要设置 `ip`
  * `docker`（使用 `docker-compose`）中的**容器** `ip` 就是 `docker-compose.yaml` 中**服务的名字**
* 使用容器名字

## 容器访问本地数据库

* 在 127.0.0.1 ip 的地方换成：`host.docker.internal`

## no such host

* `host.docker.internal` 并未设置
  * 解决方法：在 `docker run` 中加入 `--add-host host.docker.internal:host-gateway`

参考：[如何连接到 Docker 容器中的本地主机 (linux-console.net)](https://cn.linux-console.net/?p=7713#:~:text=host.docker.internal – 这解析为外部主机。 如果您在主机上运行 MySQL 服务器，Docker 容器可以通过连接到 host.docker.internal%3A3306,引擎用户也可以通过 --add-host 标志为 docker run 启用 host.docker.internal 。)

