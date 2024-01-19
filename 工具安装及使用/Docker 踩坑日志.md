# Docker 踩坑日志

## 容器间通信

* 使用 `networks`，放在**一个网段**
* `rpc` 需要 `ip` 地址的，不仅需要放在一个**网段**，还需要设置 `ip`
  * `docker`（使用 `docker-compose`）中的**容器** `ip` 就是 `docker-compose.yaml` 中**服务的名字**