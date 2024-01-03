# RocketMQ  安装踩坑日志

# 1 数据存储服务的挂载点

* **文件挂载出错**

  ```txt
  docker: Error response from daemon: error while creating mount source path
  ```

  * 宿主机如果文件系统是**只读的**，需要**换一个挂载点**，不要挂载到 `/usr/local` 中
  * 挂载到 `/home` 下

