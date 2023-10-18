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

