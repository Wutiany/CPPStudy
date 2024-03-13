# gin 项目各部分架构

## 目录树

### 1.目录树

```
├─api
│  └─server
│      ├─httputils
│      │  ├─request
│      │  └─response
│      ├─middleware
│      └─router
│          ├─auth
│          ├─homepage
│          └─user
├─cmd
│  └─app
│      ├─config
│      └─options
├─docs
├─etc
├─pkg
│  ├─db
│  │  └─model
│  ├─jwt
│  ├─log
│  ├─logic
│  │  ├─homepage
│  │  └─user
│  ├─mq
│  ├─oss
│  │  └─qiniu
│  ├─types
│  └─utils
│      ├─config
│      ├─debug
│      ├─encrypt
│      ├─error
│      └─os
├─storage
│  └─logs
└─templates
```

### 2.目录树功能注释

```
├─api
│  └─server
│      ├─httputils
│      │  ├─request   // 解析请求的结构体以及方法（验证器函数）
│      │  └─response  // 响应的结构体以及方法（设置 code、data、message）
│      ├─middleware  // 中间件
│      └─router   // 路由以及中间件等功能的初始化（调用函数）
│          ├─auth    // auth 路由组具体的路由功能，以及路由初始化具体的函数
│          ├─homepage // homepage 路由组具体的路由功能，以及路由初始化具体的函数
│          └─user  // user 路由组具体的路由功能，以及路由初始化具体的函数
├─cmd    // 整个服务配置和启动的文件
│  └─app    // app 的配置服务，即初始化部分
│      ├─config  // 解析配置文件字段的结构体
│      └─options  // 依赖结构体，初始化依赖（依赖注入），同时构建一些 options 结构体的函数，用来在 Run 中调用
├─docs
├─etc    // 配置文件
├─pkg   // 具体的逻辑（以接口的形式提供）
│  ├─db  // 数据库操作的接口
│  │  └─model   // gorm 的模型：操作数据库的结构体
│  ├─jwt   // jwt提供功能的接口
│  ├─log   // 日志提供功能的接口
│  ├─logic   // 路由的具体操作逻辑，提供了一个主接口，然后其他的接口分别用不同文件夹写 
│  │  ├─homepage	// homepage 操作的接口以及逻辑
│  │  └─user		// user 操作的接口以及逻辑
<<<<<<< HEAD
│  ├─mq    // mq 的逻辑
│  ├─oss   // 对象存储的逻辑
│  │  └─qiniu
│  ├─types  // 一些逻辑中用到的类型，放到这里
│  └─utils  // 工具函数，减少代码冗余
=======
│  ├─mq
│  ├─oss
│  │  └─qiniu
│  ├─types
│  └─utils
>>>>>>> master
│      ├─config    // 读取配置文件的功能结构体和函数
│      ├─debug		// debug 的 print 函数
│      ├─encrypt	// 抽出来的加密函数
│      ├─error	// 自定义错误的工具
<<<<<<< HEAD
│      └─os    // 系统操作的函数
=======
│      └─os
>>>>>>> master
├─storage
│  └─logs
└─templates
```

## 基本架构

### 1. main.go

* 服务器启动入口
* `NewServerCommand`：配置和服务的初始化入口，**初始化** cmd 文件下的**配置文件**以及**运行服务**

### 2.命令行工具（cobra）

**路径： cmd/app/server.go**

**函数组成**：`NewServerCommand，Run`

**NewServerCommand()**

* `Options` **初始化**调用位置所在，一些初始化的工作都放在这里面
* 用命令行工具来创建**命令行程序**（创建命令行参数）
* **绑定**命令行参数

**Run()**

* 服务器的**优雅启动**，将服务器的初始化拆分
  * 优雅启动就是通过一个 `chan` 接收**系统**的 `SIGINT SIGTERM` 信号，然后使用一个**超时上下文**进行 `shutdown` 服务器
  * 绑定信号通知，释放 chan 的信号（阻塞态），**接收到信号（释放掉）**之后使用**超时上下文**进行关闭
* 专门使用 `Run` 函数来对 `Options` 中的 `gin.HttpEngine` 进行**初始化**
  * 将 `Options` 的 `HttpEngine` 作为 `http.Server` 的 `Handler`
  * 初始化 `Options` 的 `HttpEngine` 的 `Router`：`InstallRouters`（所有的组件，中间件，路由，路由分组，资源加载，都在这）
  * `InstallRouters` 函数就是**初始化** `gin` 的**服务器**
* **Run 函数**只是运行 `http.Server` 初始化放在其他函数（router.InstallRouters）

**参考文档**：[万字长文——Go 语言现代命令行框架 Cobra 详解 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/627848739)

### 3.配置初始化（Options）

**路径：cmd/app/options/options.go**

**函数组成**：`NewOptions，Complete，BindFlags，register，registerDatabase，registerLog，registerJwt，registerRabbitMQ，Validate`

**结构体组成：**`Options`

**NewOptions**

* 获取一个 初始化的`Options` 结构体指针，供后续**初始化**结构体的**字段**
  * 初始化读取的配置文件路径
  * 初始化一个**默认**的 `gin` 服务器（gin.Default ）

**Complete**

* 验证配置文件
* 读取本地配置文件，载入配置文件**结构体**（工具结构体，提供初始化的功能，传入参数）
* 检查初始化的参数正确性
* 调用**依赖注入函数（register）**，进行**依赖注入**
* 初始化逻辑依赖（Logic，**路由功能**的**主要逻辑**，操作数据库等逻辑放到这里）

**BindFlags**

* **绑定配置文件**的**结构体**（命令行 cobra.Command）

**register**

* 不**对外暴漏**的**依赖注入**结构体
* **registerDatabase**
  * `options` 结构体的**数据库接口**依赖注入
* **registerLog**
  * `options` 结构体的**日志接口**依赖注入
* **registerJwt**
  * `options` 结构体的**鉴权接口**的依赖注入
* **registerRabbitMQ**
  * `options` 结构体的 RabbitMQ **消息队列接口**的依赖注入

**Validate**

* 可以进行参数验证啥的

**总结**

* `options` 结构体的 `func` 可以自定义，在 `cobra` 里面调用就行

### 4.配置文件结构体以及初始化工具

#### 4.1 配置文件结构体

**路径：cmd/app/config/config.go**

* 各种结构体，对应 `config.yaml` 中的字段 

#### 4.2 初始化工具

**路径：pkg/utils/config/config.go**

* 有一个配置结构体，通过 New 函数获得
* 使用结构体函数初始化这个结构体
  * 配置文件路径
  * 配置文件类型等
* 通过函数进行读取解析结构体中的路径

### 5.gin.Engine 初始化

**路径：api/server/router/router.go**

**函数**：` InstallRouters，install`

**InstallRouters**

* 用来注册路由使用，也就相当于根路由，一些配置功能放在这里：通过函数名列表，来逐一注册功能
  * 注册中间件
  * 注册路由

**install**

* 遍历函数列表，调用来注册路由，实际注册功能会放到函数中（NewRouter，获取 options 的 engine 来进行构造路由和路由组等）

### 6.中间件的注册

**路径：api/server/middleware.go**

* 封装了 `Options` 使用 `HttpEngine.Use()` 的方法
* 需要自定义中间件

## 路由初始化

<<<<<<< HEAD
### 1.路由组件初始化

**路径：api/server/router/user/user.go**

**结构体**：`userRouter`

* 提供路由**逻辑接口**，以及**组件**的接口

**函数**：`NewRouter，initRoutes`

**NewRouter**

* 初始化 `userRouter` 结构体，同时调用结构体的方法 `initRoutes`，初始化**路由**

**initRoutes**

* `userRouter` 的方法，用来初始化路由，同时调用 `userRouter` 定义的方法来作为 `HandlerFunc`

### 2. HanderFunc

**路径：api/server/router/user/user_router.go**

* 这里面是路由的 `HandlerFunc` 实现，因为是 `userRouter` 的方法（该结构体的**字段**包含需要用到的组件和逻辑接口），所以可以**调用**服务器的**组件**和**服务**
* 在这里面所用的到的绑定的结构体是**请求结构体**，**绑定**的是 `request` 的内容

## 逻辑功能接口

### 1.数据库功能接口

#### 1.1 数据库模型（操作数据库表的结构体）

**路径：pkg/db/model**

* 这个路径下**是数据库表**的模型（结构体）

#### 1.2 数据库接口（封装了各个表的接口）

**路径：pkg/db/db.go**

**结构体**：`shareDaoFactory`

* 最外层的接口，封装了各个数据库表的接口
* 结构体内是 `*gorm.DB`，用来初始化**表接口结构体**的 `*gorm.DB`，来**实际访问**数据库表

#### 1.3 数据库表接口

**路径：pkg/db**

* 数据库表操作的接口，提供各种**操作表**的方法

### 2.日志等以来的逻辑

**路径：pkg/db/***

* 所有**依赖**的**逻辑结构**放到这里

## 中间件

### 1.中间件初始化

**路径：api/server/middleware/middleware.go**

* 初始化中间件组件

### 2.中间件实现

**路径：api/server/middleware/*.go**

* **中间件代码**的具体**实现**

## 请求与响应

### 1.请求

**路径：api/server/httputils/request/***

* 请求要绑定的**结构体**
* `binding` 的**验证器**，用来验证**必须绑定**但**未绑定到**的内容
  * 通过 **字段 + tag** 来进行验证
  * 如果未绑定到，err 中会有字段（Field）和 tag（Tag）
* 对请求的一些**操作**

### 2.响应

**路径：api/server/httputils/response/***

* RESTful 标准，相应一般是固定格式
  * `code`：状态码，最好自定义，可以显示更加具体的错误
  * `message`：错误消息等
  * `data`：返回的数据

**路径：api/server/httputils/httputils.go**

* 主要是这个，这个用来定义**响应的内容**，提供操作**响应结构体**的一些**方法**




## 逻辑功能接口
