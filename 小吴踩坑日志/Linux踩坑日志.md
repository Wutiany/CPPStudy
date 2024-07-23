# Linux 踩坑日志

## 网卡消失问题

**配置文件还在（/etc/sysconfig/network-scripts/），但是 ifconfig 不显示网卡信息**

将 NetworkManager 服务从系统启动时的运行级别中移除，以防止它在系统启动时自动运行，手动配置网络连接

```shell
$ chkconfig NetworkManager off
```

将 network 服务在系统启动时自动运行

```shell
$ chkconfig network on
```

停止正在运行的 NetworkManager 服务

```shell
$ service NetworkManager stop
```

启动 network 服务

```shell
$ service network start
```

[参考链接]([centos7 虚拟网卡丢失问题（已解决）_centos7网卡不见了-CSDN博客](https://blog.csdn.net/m0_59252626/article/details/139355670#:~:text=按顺序执行以下命令 systemctl stop NetworkManager %23停止网络守护进程 systemctl disable NetworkManager,network.service %23启动network.service服务 最后重启网卡 重启网卡 service network restart 查看网卡))



