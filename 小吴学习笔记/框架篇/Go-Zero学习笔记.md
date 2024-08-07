# go-zero 框架使用

## 1.工具安装

[golang 安装 | go-zero Documentation](https://go-zero.dev/docs/tasks)

[goctl 安装 | go-zero Documentation](https://go-zero.dev/docs/tasks/installation/goctl)

[protoc 安装 | go-zero Documentation](https://go-zero.dev/docs/tasks/installation/protoc)

[go-zero 安装 | go-zero Documentation](https://go-zero.dev/docs/tasks/installation/go-zero)

[goctl-intellij 安装 | go-zero Documentation](https://go-zero.dev/docs/tasks/installation/goctl-intellij)

[goctl-vscode 安装 | go-zero Documentation](https://go-zero.dev/docs/tasks/installation/goctl-vscode)

## 2.CTL（goctl）可以生成的框架代码

### 2.1 api 生成

**http 的 server api**

* 生成默认的一个 `webserver` 的 `api`

  ```shell
  $ mkdir -p ./workspace/api && cd ./workspace/api
  $ goctl api new demo
  ```

  生成一个默认的 demo server 框架

* 文件结构：

  * ├── demo.api
    ├── demo.go
    ├── etc
    │  └── demo-api.yaml
    ├── go.mod
    └── internal
        ├── config
        │  └── config.go
        ├── handler
        │  ├── demohandler.go
        │  └── routes.go
        ├── logic
        │  └── demologic.go
        ├── svc
        │  └── servicecontext.go
        └── types
            └── types.go
  * 其中 handler 会调用其在 `logic` 文件加下生成的 demologic.go 的请求处理逻辑代码，**只需要修改**其中的代码即可处理请求，已经生成相应的数据结构。
  * ./internal/types/types.go 文件中存放的是 .api 文件中定义的数据结构，使用 `go` 语言生成的相应的数据结构。

* 使用自己写好的 .api 文件生成代码

  ```shell
  $ goctl api go -api ./filename.api -dir ./createdir
  ```

* 如果修改 .api 文件的内容如（router），只需要使用以上命令重新生成即可，已经修改好的逻辑处理文件**不会被覆盖或者修改**

### 2.2 gRPC 生成

**gRPC 的代码框架**

* 生成一个默认的 gRPC 服务

  ```shell
  $ mkdir -p ./workspace/rpc && cd ./workspace/rpc
  $ goctl rpc new demo
  ```

* 文件结构：

  * ├── demo
    │  ├── demo.pb.go
    │  └── demo_grpc.pb.go
    ├── demo.go
    ├── demo.proto
    ├── democlient
    │  └── demo.go
    ├── etc
    │  └── demo.yaml
    ├── go.mod
    └── internal
        ├── config
        │  └── config.go
        ├── logic
        │  └── pinglogic.go
        ├── server
        │  └── demoserver.go
        └── svc
            └── servicecontext.go
  * 同样只需要修改其中 `./internal/logic/pinglogic.go` 代码即可

### 2.3 mysql 生成

**生成一个对 mysql 数据库特定表操作的代码**

* 首先需要创建 sql 文件存储到本地的 `user.sql` 文件中

  ```mysql
  CREATE TABLE user (
      id bigint AUTO_INCREMENT,
      name varchar(255) NULL COMMENT 'The username',
      password varchar(255) NOT NULL DEFAULT '' COMMENT 'The user password',
      mobile varchar(255) NOT NULL DEFAULT '' COMMENT 'The mobile phone number',
      gender char(10) NOT NULL DEFAULT 'male' COMMENT 'gender,male|female|unknown',
      nickname varchar(255) NULL DEFAULT '' COMMENT 'The nickname',
      type tinyint(1) NULL DEFAULT 0 COMMENT 'The user type, 0:normal,1:vip, for test golang keyword',
      create_at timestamp NULL,
      update_at timestamp NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
      UNIQUE mobile_index (mobile),
      UNIQUE name_index (name),
      PRIMARY KEY (id)
  ) ENGINE = InnoDB COLLATE utf8mb4_general_ci COMMENT 'user table';
  ```

* 创建工作空间和目录工程

  ```shell
  $ mkdir -p ./workspace/model/mysql
  ```

  将 `user.sql` 文件移动到 `./workspace/model/mysql` 目录下

* 生成 model 代码

  ```shell
  $ cd ./workspace/mysql
  $ goctl model mysql ddl --src user.sql --dir .
  ```

  使用 `mysql` 的 `DDL（Data Definition Language）`数据库定义语言，生成模型代码

* 文件结构：

  * ├── user.sql
    ├── usermodel.go
    ├── usermodel_gen.go
    └── vars.go
  * usermodel_gen.go 中的代码就是生成的数据库操作的代码

### 2.4 mongo 生成

**与 mysql 类似，产生一个操作 mongo 数据库的模型代码**

* 代码生成

  ```shell
  $ mkdir -p ./workspace/model/mongo && ./workspace/model/mongo
  $ goctl model mongo --type user --dir .
  ```

* 文件结构：

  * ├── error.go
    ├── usermodel.go
    ├── usermodelgen.go
    └── usertypes.go
  * usermodelgen.go 中的代码就是生成的数据库操作的代码

### 2.5 api 文件格式化

**使用 api 格式化命令，将写好的 api 文件的内容进行格式化**

* 命令

  ```shell
  $ goctl api format --dir demo.api
  ```

[api demo 代码生成 | go-zero Documentation](https://go-zero.dev/docs/tasks/cli/api-demo)

## 3. api 文件的语法

**只需要写 api 文件即可，使用 goctl 工具生成其余代码**

### 3.1 api 语法

#### 示例1.编写最简单的 ping 路由服务

```go
syntax = "v1"

// 定义 HTTP 服务
service foo {
    get /ping
}
```

foo 为最后生成服务文件的名字

#### 示例2.编写一个登录接口 api 文件

```go
syntax = "v1"

type (
    // 定义登录接口的请求体
    LoginReq {
        Username string `json:"username"`
        Password string `json:"password"`
    }
    // 定义登录接口的响应体
    LoginResp {
        Id       int64  `json:"id"`
        Name     string `json:"name"`
        Token    string `json:"token"`
        ExpireAt string `json:"expireAt"`
    }
)

// 定义 HTTP 服务
// 微服务名称为 user，生成的代码目录和配置文件将和 user 值相关
service user {
    // 定义 http.HandleFunc 转换的 go 文件名称及方法
    @handler Login
    // 定义接口
    // 请求方法为 post
    // 路由为 /user/login
    // 请求体为 LoginReq
    // 响应体为 LoginResp，响应体必须有 returns 关键字修饰
    post /user/login (LoginReq) returns (LoginResp)
}
```

#### 示例 3. 编写简单的用户服务 api 文件

```go
syntax = "v1"

type (
    // 定义登录接口的 json 请求体
    LoginReq {
        Username string `json:"username"`
        Password string `json:"password"`
    }
    // 定义登录接口的 json 响应体
    LoginResp {
        Id       int64  `json:"id"`
        Name     string `json:"name"`
        Token    string `json:"token"`
        ExpireAt string `json:"expireAt"`
    }
)

type (
    // 定义获取用户信息的 json 请求体
    GetUserInfoReq {
        Id int64 `json:"id"`
    }
    // 定义获取用户信息的 json 响应体
    GetUserInfoResp {
        Id   int64  `json:"id"`
        Name string `json:"name"`
        Desc string `json:"desc"`
    }
    // 定义更新用户信息的 json 请求体
    UpdateUserInfoReq {
        Id   int64  `json:"id"`
        Name string `json:"name"`
        Desc string `json:"desc"`
    }
)

// 定义 HTTP 服务
// @server 语法块主要用于控制对 HTTP 服务生成时 meta 信息，目前支持功能有：
// 1. 路由分组
// 2. 中间件声明
// 3. 路由前缀
// 4. 超时配置
// 5. jwt 鉴权开关
// 所有声明仅对当前 service 中的路由有效
@server (
    // 代表当前 service 代码块下的路由生成代码时都会被放到 login 目录下
    group: login
    // 定义路由前缀为 "/v1"
    prefix: /v1
)
// 微服务名称为 user，生成的代码目录和配置文件将和 user 值相关
service user {
    // 定义 http.HandleFunc 转换的 go 文件名称及方法，每个接口都会跟一个 handler
    @handler login
    // 定义接口
    // 请求方法为 post
    // 路由为 /user/login
    // 请求体为 LoginReq
    // 响应体为 LoginResp，响应体必须有 returns 关键字修饰
    post /user/login (LoginReq) returns (LoginResp)
}

// @server 语法块主要用于控制对 HTTP 服务生成时 meta 信息，目前支持功能有：
// 1. 路由分组
// 2. 中间件声明
// 3. 路由前缀
// 4. 超时配置
// 5. jwt 鉴权开关
// 所有声明仅对当前 service 中的路由有效
@server (
    // 代表当前 service 代码块下的所有路由均需要 jwt 鉴权
    // goctl 生成代码时会将当前 service 代码块下的接口
    // 信息添加上 jwt 相关代码，Auth 值为 jwt 密钥，过期
    // 等信息配置的 golang 结构体名称
    jwt: Auth
    // 代表当前 service 代码块下的路由生成代码时都会被放到 user 目录下
    group: user
    // 定义路由前缀为 "/v1"
    prefix: /v1
)
// 注意，定义多个 service 代码块时，服务名称必须一致，因此这里的服务名称必须
// 和上文的 service 名称一样，为 user 服务。
service user {
    // 定义 http.HandleFunc 转换的 go 文件名称及方法，每个接口都会跟一个 handler
    @handler getUserInfo
    // 定义接口
    // 请求方法为 post
    // 路由为 /user/info
    // 请求体为 GetUserInfoReq
    // 响应体为 GetUserInfoResp，响应体必须有 returns 关键字修饰
    post /user/info (GetUserInfoReq) returns (GetUserInfoResp)

    // 定义 http.HandleFunc 转换的 go 文件名称及方法，每个接口都会跟一个 handler
    @handler updateUserInfo
    // 定义接口
    // 请求方法为 post
    // 路由为 /user/info/update
    // 请求体为 UpdateUserInfoReq
    // 由于不需要响应体，因此可以忽略不写
    post /user/info/update (UpdateUserInfoReq)
}
```

#### 示例 4. 编写带有中间件的 api 服务

```go
syntax = "v1"

type GetUserInfoReq {
    Id int64 `json:"id"`
}

type GetUserInfoResp {
    Id   int64  `json:"id"`
    Name string `json:"name"`
    Desc string `json:"desc"`
}

// @server 语法块主要用于控制对 HTTP 服务生成时 meta 信息，目前支持功能有：
// 1. 路由分组
// 2. 中间件声明
// 3. 路由前缀
// 4. 超时配置
// 5. jwt 鉴权开关
// 所有声明仅对当前 service 中的路由有效
@server (
    // 定义一个鉴权控制的中间件，多个中间件以英文逗号,分割，如 Middleware1,Middleware2,中间件按声明顺序执行
    middleware: AuthInterceptor
)
// 定义一个名称为 user 的服务
service user {
    // 定义 http.HandleFunc 转换的 go 文件名称及方法，每个接口都会跟一个 handler
    @handler getUserInfo
    // 定义接口
    // 请求方法为 post
    // 路由为 /user/info
    // 请求体为 GetUserInfoReq
    // 响应体为 GetUserInfoResp，响应体必须有 returns 关键字修饰
    post /user/info (GetUserInfoReq) returns (GetUserInfoResp)
}
```

#### 示例 5. 编写带有超时配置的 api 服务

```go
syntax = "v1"

type GetUserInfoReq {
    Id int64 `json:"id"`
}

type GetUserInfoResp {
    Id   int64  `json:"id"`
    Name string `json:"name"`
    Desc string `json:"desc"`
}

// @server 语法块主要用于控制对 HTTP 服务生成时 meta 信息，目前支持功能有：
// 1. 路由分组
// 2. 中间件声明
// 3. 路由前缀
// 4. 超时配置
// 5. jwt 鉴权开关
// 所有声明仅对当前 service 中的路由有效
@server (
    // 定义一个超时时长为 3 秒的超时配置，这里可填写为 time.Duration 的字符串形式，详情可参考 
    // https://pkg.go.dev/time#Duration.String
    timeout: 3s
)
// 定义一个名称为 user 的服务
service user {
    // 定义 http.HandleFunc 转换的 go 文件名称及方法，每个接口都会跟一个 handler
    @handler getUserInfo
    // 定义接口
    // 请求方法为 post
    // 路由为 /user/info
    // 请求体为 GetUserInfoReq
    // 响应体为 GetUserInfoResp，响应体必须有 returns 关键字修饰
    post /user/info (GetUserInfoReq) returns (GetUserInfoResp)
}
```

#### 示例 6. 结构体引用

```go
syntax = "v1"

type Base {
    Code int    `json:"code"`
    Msg  string `json:"msg"`
}

type UserInfo {
    Id   int64  `json:"id"`
    Name string `json:"name"`
    Desc string `json:"desc"`
}

type GetUserInfoReq {
    Id int64 `json:"id"`
}

type GetUserInfoResp {
    // api 支持匿名结构体嵌套，也支持结构体引用
    Base
    Data UserInfo `json:"data"`
}

// 定义一个名称为 user 的服务
service user {
    // 定义 http.HandleFunc 转换的 go 文件名称及方法，每个接口都会跟一个 handler
    @handler getUserInfo
    // 定义接口
    // 请求方法为 post
    // 路由为 /user/info
    // 请求体为 GetUserInfoReq
    // 响应体为 GetUserInfoResp，响应体必须有 returns 关键字修饰
    post /user/info (GetUserInfoReq) returns (GetUserInfoResp)
}
```

#### 示例 7. 控制最大请求体控制的 api 服务

```go
syntax = "v1"

type GetUserInfoReq {
    Id int64 `json:"id"`
}

type GetUserInfoResp {
    Id   int64  `json:"id"`
    Name string `json:"name"`
    Desc string `json:"desc"`
}

// @server 语法块主要用于控制对 HTTP 服务生成时 meta 信息，目前支持功能有：
// 1. 路由分组
// 2. 中间件声明
// 3. 路由前缀
// 4. 超时配置
// 5. jwt 鉴权开关
// 所有声明仅对当前 service 中的路由有效
@server (
    // 定义一个请求体限制在 1MB 以内的请求，goctl >= 1.5.0 版本支持
    maxBytes: 1048576
)
// 定义一个名称为 user 的服务
service user {
    // 定义 http.HandleFunc 转换的 go 文件名称及方法，每个接口都会跟一个 handler
    @handler getUserInfo
    // 定义接口
    // 请求方法为 post
    // 路由为 /user/info
    // 请求体为 GetUserInfoReq
    // 响应体为 GetUserInfoResp，响应体必须有 returns 关键字修饰
    post /user/info (GetUserInfoReq) returns (GetUserInfoResp)
}
```

[api 语法 | go-zero Documentation](https://go-zero.dev/docs/tasks/dsl/api)

### 3.2 语法总结

.api 主要包括：指定版本、定义服务、数据类型（服务的 request 和 response 的数据定义）、服务语法块（用来控制生成下方 server 的 meta 信息）

#### 3.2.1 服务定义

* 定义服务的 `handler`（可以定义多个 handler，接口对应相应的 handler）
* 定义服务的接口：方法（POST）， router，响应体（request）returns（response）

#### 3.2.2 数据类型

* 数据类型（请求体，响应体）数据结构的定义
* 可以引用其他的结构体（相当于继承），但是请求体和结构体必须在当前文件中实现

#### 3.2.3 其他服务（服务语法块）

* 支持的功能（@server( 声明服务 )）
  * 1. 路由分组
    2. 中间件声明
    3. 路由前缀
    4. 超时配置
    5. jwt 鉴权开关

1. 路由分组
   * 参数：group: user
   * 当前代码块下的代码生成在 `user` 目录下
2. 中间件声明
   * 参数：middleware: AuthInterceptor
   * 定义一个鉴权控制的中间件，多个中间件以英文逗号,分割，如 Middleware1,Middleware2,中间件按声明顺序执行
3. 路由前缀
   * 参数：prefix: /v1
   *  定义路由前缀为 "/v1"
4. 超时配置
   * 参数：timeout: 3s
   * 定义一个超时时长为 3 秒的超时配置，这里可填写为 time.Duration 的字符串形式，详情可参考 https://pkg.go.dev/time#Duration.String
5. jwt 鉴权开关
   * 参数：jwt: Auth
   * 代表当前 service 代码块下的路由生成代码时都会被放到 user 目录下
6. 限制请求体大小
   * 参数：maxBytes: 1048576
   * 定义一个请求体限制在 1MB 以内的请求，goctl >= 1.5.0 版本支持

[API 规范 | go-zero Documentation](https://go-zero.dev/docs/tutorials)

## 4.jwt 使用

### 4.1 jwt 组成

#### 4.1.1 编码的各个结构以及方法

* HEADER（头部）
  * 其中包括了对签名的加密算法，和密钥的 id（'typ':"JWT"）
  * 使用 Base64 进行编码
* PAYLOAD（荷载）
  * 其中包括了需要加密的内容，但是不能将用户隐私信息放入进行加密（密码）
  * 依旧使用 Base64 进行编码
* SIGNATURE（签名）
  * 对头部和荷载部分进行加密生成的，使用头部中的加密算法进行加密
  * 首先将编码完成之后的头部和荷载使用 "." 进行拼接，形成一个字符串
  * 然后使用指定的算法和密钥对该字符串进行加密生成签名
  * 将生成的签名进行 Base64 编码
* 编码结果
  * 头部.荷载.签名

#### 4.1.2 传输

* 服务器传给客户端：
  * 放置在 **header** 中，格式：`Authorization: Bearer token(不带'')`（一般使用的方法）
  * 放置在表单中：json 中
* 客户端传给服务器：
  * 放置在 **header** 中，格式：`Authorization: Bearer token(不带'')`（一般使用的方法）
  * 放置在 url 中

#### 4.1.3 服务端解析 token 内容

使用 handler 结构中的 context（ctx）的 Value 方法进行解析，ctx 会将解析出来的头的内容以及 token 解析出来的内容，通过 **map** 进行保存

`authHeader := l.ctx.Value("username")`

### 4.2 jwt 开关

鉴权开关`jwt: Auth`，开启鉴权开关，然后需要编写产生 **token** 的**function**，在处理登陆请求的时候，通过产生 **token** 的 **function** 返回客户端 **token**

### 4.3 token 生成

```go
func GetJwtToken(secretKey string, iat, seconds int64, username string) (string, error) {
	claims := make(jwt.MapClaims)
	claims["exp"] = iat + seconds
	claims["iat"] = iat
	claims["username"] = username  // 需要
	token := jwt.New(jwt.SigningMethodHS256)
	token.Claims = claims
	return token.SignedString([]byte(secretKey))
}
```

### 4.4 token 解析

使用 handler 结构中的 context（ctx）的 Value 方法进行解析，ctx 会将解析出来的头的内容以及 token 解析出来的内容，通过 **map** 进行保存

`authHeader := l.ctx.Value("username")`

## 5.api 整体框架逻辑

### 5.1 文件结构

├─etc
├─internal
│  ├─config
│  ├─handler
│  │  ├─login
│  │  ├─order
│  │  ├─register
│  │  └─update
│  ├─logic
│  │  ├─login
│  │  ├─order
│  │  ├─register
│  │  └─update
│  ├─svc
│  └─types

* etc 中的 service.yaml：是服务器的配置文件，用来初始化服务器配置（api 所需要的信息都可以存这里面）
  * Name，Host，Port
  * Config 结构体中定义的变量，如 Auth 的参数
* internal
  * config 中的 config.go：是服务器的配置定义，需要通过 service.yaml 来初始化
    * 这里面配置变量的访问是通过 svcCtx 来进行访问，数据库模型可以放在这里面（l.svcCtx.Config.Auth.AccessExpire）
  * handler 存放了不同的 handler
  * logic 是 ServeHTTP 中的逻辑（handler 处理请求的逻辑）
  * svc 中 servicecontext.go 的代码中存放的是服务器的上下文（定义的上下文结构体），同时使用初始化函数（newServiceContext）进行初始化
    * 配置（Config）服务器的配置，鉴权的配置字段（密钥，持续时长）也在这
    * 数据库模型声明（ServiceContext结构体中），和初始化
  * types api 请求和响应的数据类型

### 5.2 代码逻辑结构

**server：是一个整体（servicenameLogic 就是服务）（这个就相当于 /net/http 中的 handle 结构体）**

* servicenameLogic struct（封装了服务需要的字段和方法）
  * 方法：servicename（servicenameLogic struct）自动生成，处理请求的逻辑主体
  * 字段：
    * ctx（context.Context）这个字段中包括了对客户端 request 的处理
    * svcCtx（*svc.ServiceContext）这个字段是封装的服务的一些功能，服务会调用的东西，配置文件，tablesModel

**创建逻辑**

主体的 service.go（main 函数，api 服务的主体） 会先使用 rest 中的方法将 config 文件中的 config 结构体作为参数创建一个 http server，在通过 serviceContext 创建一个服务器会使用到的上下文结构体（服务器会使用到的数据库模型等），在通过 ResigterHandler 创建 router 对应的 handler（这个是 handle，不是 func，而是结构体，相当于 Handle）然后开启服务。逻辑与 net/http 相似，不过服务器的配置信息（port、ip 等）都在 config 中了（rest.RestConf go-zero 提供的一个结构体中已经封装好了参数），也可以使用 yaml 文件进行初始化服务器参数。

结构

|main.go

|—创建服务器

|—通过 yaml 文件初始化服务器配置（config 结构体）

|——初始化 config 结构体

|—使用 config 结构体的 RestConf 字段创建服务器（等同于 http.Server 的初始化）

|—创建服务器的上下文（服务器会用到的结构体和字段，配置文件“config 结构体”，数据库模型“tableModel”）

|——服务器上下文（ServiceContext 结构体）可以通过 handler 进行访问（l.svcCtx）

|——可以将服务器的使用的东西如：数据库表的操作模型放置到这里里面作为一个字段，通过 l.svcCtx.TableModel 进行访问

|—创建服务器的 handler（mux.Handle，这里是结构体），handle 结构体，用来处理每个对应的 router

|——logic 结构体封装了服务器上下文，和上下文（上下文可以用来获取请求的一些头部信息，通过Value 来进行获取）

**主要的结构体**

* Config 结构体：用来获取服务器的配置，如增加的鉴权后的参数，logic 可以通过这个进行访问
* ServiceContext 结构体：服务器的上下文结构体，将 Config 结构体作为其中的一个字段，同时服务器的一些其他组件如数据库都可以放在这里，该结构体通过 NewServiceContext 进行获取，传入的参数为初始化好的 Config 结构体
* Logic 结构体：Logic Handle，通过将服务器上下文结构体作为其中的字段，来获取访问服务器的一些功能和参数，同时封装了日志，和将请求解析（请求头部或者鉴权的 token 解析的 payload）的上下文

## 6.mysql 使用

### 6.1 mysql 代码结构的生成

* 生成的基本函数
  * Delete
  * FindOne
  * FIndOneByUniqueKey
  * Insert
  * Update
  * tableName
* 根据设置 **unique key** 生成查询的函数
  * 默认生成主键的查询函数（FindOne）
  * 然后根据 **unique key** 生成其他的查询函数
  * unique key 会生成索引，所以会根据 `unique key` 生成查询函数

### 6.2 查询函数

* 查询函数使用的是预编译的语句，其使用了第三方库 `github.com/go-sql-driver/mysql` 来实现的

### 6.3 自定义查询等方法

#### 6.3.1 定义位置

* tabelnamemodel.go 中定义
* tabelnamemodel_gen.go 是已经产生的，不能取更改

#### 6.3.2 定义方法

* 首先是查询等方法的编写，编写成 customTablenameModel 结构体的方法
* 然后对接口进行声明方法（tablenameModel）接口，其本身存在一个接口，这个接口定义了原始的方法（defaultTablenameModel 结构的方法），只需要对新的方法进行添加就可以了
* 结构体嵌套 -- 》接口的方法也需要进行嵌套（要和结构体的结构对应）

## 7 middleware 使用

### 7.1 启用 middleware

* 在 api 文件进行定义就行，自定义 middleware 名字，多个中间件，使用逗号隔开

### 7.2 middleware 编写

* 位置 internal/middleware 下的文件
* 对其中的文件进行中间件处理逻辑的编写就行
  * 中简件的 Handle 函数是需要编写的，Handle 函数返回一个 http.HandlerFunc
  * 中间件是处理 request 请求，然后通过处理逻辑在进入下一个 handler







​	

