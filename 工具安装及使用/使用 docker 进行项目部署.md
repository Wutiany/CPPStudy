# 使用 docker 进行项目部署

## first.部署所需要的东西

* **makefile**：为了简化部署流程，使用 `makefile` 进行部署，项目构建的命令的封装
* **dockerfile**：程序运行的环境，为二进制程序配置一个运行的容器
* **docker-compose.yaml**：一键部署容器的配置文件，用来对容器进行一键编排

## second.流程

》**makefile**：用来通过命令来构建整体的运行环境

》》**dockerfile**：`makefile` 中使用的内容，通过这个构建代码运行环境的容器

》》**docker-compose.yaml**：同样可以在`makefile` 中使用命令进行运行

## third.文件代码编写

### makefile 内容

* `makefile` 中每一行的命令都在一个独立的 shell 中运行

- 对文件夹中的项目进行编译
  - 可能需要 `go mod tiny`，对项目的以来进行获取
  - 编译成二进制文件：`go build -p 8 -o demo`（使用多线程进行编译 -p，使用 8 个线程进行编译）
- 编译 `build-image`，通过 `docker file` 编译镜像
  - 1.编译镜像（docker build -t name:version . >/dev/null）
  - 2.为镜像创建一个 tag（docker tag $(NAME):$(VERSION) $(NAME):latest），后面的是新的标签
  - 3.将docker image 推送到 docker 的镜像仓库（docker push imagename）
  - 4.启动镜像（docker run）做端口映射

### dockerfile 内容

#### 代码运行环境的 dockerfile

* 从 `docker hub` 中获取的镜像名称以及版本 `name:version`
* 配置时区
* 修改软件源
* 将本地已经 `build` 好的二进制文件传到容器中
* 修改传输完成的可执行文件的权限
* 暴露端口
* 设置容器启动时执行的命令（ENTRYPOINT ["commond", "parameter1", "parameter2"...]）

#### 数据库容器的 dockerfile

* 首先初始化数据库需要一个 `init.sql`，ddl 的数据库命令
* 从 `docker hub` 获取数据库的名称以及版本
* 将初始化的脚本复制到容器中
* 暴露端口

### docker-compose.yaml 内容

* 组织两个容器，对两个容器进行设置和管理

## 生成二进制文件进行部署

### 静态编译

* 在 windows 下生成 linux 可以使用的二进制可执行文件

  ```shell
  $ SET CGO_ENABLED=0   # 禁用对 C 语言的依赖
  $ SET GOOS=linux		# 编译为 linux 可执行文件
  $ go build -o yourapp -a -ldflags "-s"  # yourapp 为生成的二进制可执行文件的名字
  ```

* 直接构建







