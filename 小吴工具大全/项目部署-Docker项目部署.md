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

  * 创建两个环境，一个用来编译，一个用来作为编译完成的可执行文件的承载。将文件复制进 image 中，进行 build，但是需要 go mod tidy 来获取依赖




## 部署具体流程

### 1.代码文件的编写

#### 1.1 需要生成的代码

* api 文件
* model 文件：通过 goctl model mysql datasource -url "root:wutianyu@tcp(127.0.0.1:8306)/ums" -table "commodities" -dir ./model/prodcutmodel 进行编写
* rpc 文件的编写以及生成

#### 1.2 组织结构

* api
  * doc
    * 子 api 文件夹
      * 同服务的不同功能的 api 文件
    * 子 api 文件夹 ...
    * 总体 api 文件，使用 import 导入子 api 文件夹中的 api 文件
  * etc
    * 服务器配置文件
  * internal
    * 代码逻辑等生成文件
  * Dockerfile 文件，用来承载 api 程序的环境
  * 生成的 api 服务主程序文件，用来 build

* rpc
  * 子服务 rpc 文件夹
    * 生成文件及文件夹
    * proto 文件，用来生成 rpc 代码
    * Dockerfile 文件，用来承载 rpc 服务的环境
  * 子服务 rpc 文件夹 ...
  * model
    * 不同服务使用到表的文件夹（通过 goctl model mysql 生成的）
    * 不同服务使用到表的文件夹...

### 2.docker file 的编写

#### 2.1 api 文件的 dockerfile 编写

```dockerfile
# build 镜像，通过这个镜像，将本地的文件拷贝进去，然后build成可执行文件
FROM golang:alpine AS builder

LABEL stage=gobuilder

ENV CGO_ENABLED 0
ENV GOOS linux
ENV GOPROXY https://goproxy.cn,direct

WORKDIR /build/zero
COPY . .
RUN sh -c "[ -f go.mod ]" || exit
COPY etc /app/etc
RUN go mod tidy

RUN go build -ldflags="-s -w" -o /app/ums ums.go

# 可执行文件的承载镜像，可执行文件在这个镜像中运行
FROM alpine

RUN echo -e http://mirrors.ustc.edu.cn/alpine/v3.13/main/ > /etc/apk/repositories
RUN cat /etc/apk/repositories
RUN apk update --no-cache
#RUN apk add --no-cache ca-certificates
RUN apk add --no-cache tzdata
ENV TZ Asia/Shanghai

WORKDIR /app
COPY --from=builder /app/ums /app/ums
COPY --from=builder /app/etc /app/etc

CMD ["./ums", "-f", "etc/ums_docker.yaml"]
```

* 出现 `apk` 不能执行的问题，在 `docker build` 的时候加入 `--network host`，是镜像访问网络

#### 2.2 mysql 的 dockerfile 编写

``` dockerfile
FROM mysql:latest

# 需要在 makefile cd到当前路径，才能使用 ums.sql 不然会报错,需要修改一下相对路径
COPY ums.sql /docker-entrypoint-initdb.d/ums.sql
ENV MYSQL_ROOT_PASSWORD=wutianyu
# 确定数据库名称
ENV MYSQL_DATABASE=ums

# 镜像在运行的时候，会初始化数据库，但是必须确定database名称，才能将表创建进去
CMD ["--init-file", "/docker-entrypoint-initdb.d/ums.sql"]
EXPOSE 3306
```

#### 2.3 rpc 的 dockerfile 编写（与 api 相同）

```dockerfile
FROM golang:alpine AS builder

LABEL stage=gobuilder

ENV CGO_ENABLED 0
ENV GOOS linux
ENV GOPROXY https://goproxy.cn,direct

WORKDIR /build/zero
COPY . .
RUN sh -c "[ -f go.mod ]" || exit
COPY rpc/cms/etc /app/etc
RUN go build -ldflags="-s -w" -o /app/cms rpc/cms/cms.go


FROM alpine

RUN echo -e http://mirrors.ustc.edu.cn/alpine/v3.13/main/ > /etc/apk/repositories
RUN cat /etc/apk/repositories
RUN apk update --no-cache
RUN apk add --no-cache ca-certificates
RUN apk add --no-cache tzdata
ENV TZ Asia/Shanghai

WORKDIR /app
COPY --from=builder /app/cms /app/cms
COPY --from=builder /app/etc /app/etc

CMD ["./cms", "-f", "etc/cms.yaml"]

```

### 3. makefile 的编写

#### 3.1 makefile 的作用

* 通过 makefile 自动化编译或者组织编译过程等操作

#### 3.2 makefile 代码

```makefile
#.PHONY:	build complie start push
#.PHONY: compile ums mysql compose

#VERSION_TAG = 1.1.0
#MILESTONE_TAG = Oct.2023
#NAME = UMS
#
#COMMIT_ID := $(shell git rev-parse HEAD)
#
#BUILD_TS := $(shell data +'%Y%m%d%H%M%S')
#
#BRANCH_TAG := $(shell git rev-parse --abbrev-ref HEAD)
#
#VERSION := $(VERSION_TAG)-build-$(BRANCH_TAG)-$(BUILD_TS)
#
#BUILD := $(VERSION_TAG)-$(MILESTONE_TAG)-build-$(BUILD_TS)-$(BRANCH_TAG)-$(COMMIT_ID)

#export GO111MODULE = on

#compile:
#	go build -p 8 -o demo
#
#create_dir:
#	mkdir -p /dev/null
#
#build: build-version
#
#build-version:
#	docker build -t $(NAME):$(VERSION) . >/dev/null
#
#tag-latest:
#	docker tag $(NAME):$(VERSION) $(NAME):latest >dev/null
#
#push:
#	docker push $(NAME):$(VERSION); docker push $(NAME):latest
#
#start:
#	docker run -it --rm $(NAME):$(VERSION) /bin/bash

#compile:
#	go build -p 8 -o demo
#
#ums:
#	cd dev/ums && docker build -t ums:v1.0 .
#	docker run -it -p 8080:8080 --name umscontainer ums:v1.0
#
#mysql:
#	cd dev/mysql && docker build -t mysql:v1.0 .
#	docker run -d -p 3306:3306 --name mysql mysql:v1.0
#
#compose:
#	docker-compose up -d

#.PHONY: build_mysql build_api build_rpc
#
#build_mysql:
#	goctl model mysql datasource -url "root:wutianyu@tcp(127.0.0.1:8306)/ums" -table "commodities" -dir ./model/prodcutmodel
#	goctl model mysql datasource -url "root:wutianyu@tcp(127.0.0.1:8306)/ums" -table "menu*" -dir ./model/menumodel
#	goctl model mysql datasource -url "root:wutianyu@tcp(127.0.0.1:8306)/ums" -table "order*" -dir ./model/ordermodel
#	goctl model mysql datasource -url "root:wutianyu@tcp(127.0.0.1:8306)/ums" -table "role*" -dir ./model/rolemodel
#	goctl model mysql datasource -url "root:wutianyu@tcp(127.0.0.1:8306)/ums" -table "user*" -dir ./model/usermodel
#
#build_api:
#	goctl api go -api ./ums.api -dir .
#
#build_rpc:


.PHONY:all rm build run compose

all: rm build run compose


rm:
	docker stop ums
	docker stop db
	docker rm ums
	docker rm db
	docker rmi ums:v1
	docker rmi mysql:v1

build:
	docker build -t ums:v1 -f Dockerfile .
	cd database/mysql && docker build -t mysql:v1 -f Dockerfile .

#run:
## windows 中不支持 -itd 命令
##	run -itd --net=host --name=ums ums:v1
##	run -itd --net=host --name=db mysql:v1
#	docker run -itd --net=host --name=ums ums:v1
#	docker run -itd --net=host --name=db mysql:v1
# run 可以使用 compose 替代
compose:
	docker-compose up -d
```



### 4. docker-compose.yaml 文件的编写

#### 4.1 docker-compose.yaml 文件的作用

* 创建并组织容器，通过已存在的镜像，进行创建容器

#### 4.2 docker-compose 容器之间访问的方法（数据库）

* 通过共享网络，然后修改使用服务以及端口进行访问
  * 将所有容器放在同一个网络下
  * 在 mysql 的 datasource 中修改数据的访问的 host，修改成服务名

#### 4.3 docker-compose.yaml 代码

```yaml
version: '3.9'
services:
  ums:
    container_name: ums
    image: ums:v1
    ports:
      - 8888:8888
    depends_on:
      - mysql
    # 设置网络
    networks:
      - network
  # 服务名
  mysql:
    container_name: db
    image: mysql:v1
    # 设置网络
    networks:
      - network
    environment:
      - MYSQL_ROOT_PASSWORD=wutianyu
# 共享网络
networks:
  network:
```

