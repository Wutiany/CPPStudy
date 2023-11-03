# zero-admin 整体架构

## 1.结构组成

### 1.1 api

**前端服务的组成，配置文件中使用了 rpc 以及 redis**

#### 1.1.1 .api 文件组成

* **admin.api 文件**：使用嵌套的方式（import "sys/config.api"） 来构建一个整体的服务
* **其他 api 文件**：
  * **sys**：系统管理服务的 `api` 文件夹
    * **user.api**：用户列表的 **api**（router：system/dept/list）
    * **role.api**：角色列表的 **api**（router：system/role/list）
    * **menu.api**：菜单列表的 **api**（router：system/menu/list）
    * **dict.api**：字典列表的 **api**（router：system/dict/list）
    * **dept.api**：结构列表的 **api**（router：system/dept/list）
    * **loginlog.api**：非系统管理列表中的组件，登陆页面的组件（router：user/login）
    * **syslog.api**：
    * **config.api**：
    * **job.api**：职位列表的 **api**
    * **upload.api**：
  * **oms**：订单管理服务的 **api** 文件夹
  * **pms**：商品管理服务的 **api** 文件夹
  * **sms**：营销管理服务的 **api** 文件夹
  * **ums**：会员管理服务的 **api** 文件夹
  * **cms**：内容服务的 **api** 文件夹

#### 1.1.2 api 文件具体内容

**例举 user.api 文件**

* **handler** 的请求以及响应的结构体
* **@server** 注解，表示当前服务的部分 handler 的信息
  * 使用了 **middleware**：`CheckUrl` 来检测 `url`
* **service** 的内容：为对应的 `router` 设置 `handler`

### 1.2 front-api

**APP 接口，处理前端业务的接口，上面的 api 是为管理业务提供的 api**

* 与 `api` 类似结构

### 1.3 rpc

**生成的 rpc 框架**

#### 1.3.1 rpc 文件组成

* 针对不同的服务（api 中的服务）创建了不同的 rpc 服务
* 每个 rpc 中，创建服务端的处理的 rpc 函数
* 每个整体的服务（sys 等）都运行在一个单独的 docker 容器中

### 1.4 总结

* 在 api 的创建中，将子 api 进行拆分到不同的 api 文件中，通过一个主体的 api 对子 api 进行带入，来创建一个服务
  * 服务 api
    * 业务 api 
* 不用服务的 rpc  生成之后，放置到不同的 docker 容器中进行使用

## 2. api 代码逻辑

### 2.1 api 的 Config 文件

* `Config` 文件中放置了对应服务的客户端 `rpc` 参数（type RpcClientConf struct ），通过 `yaml` 文件进行初始化
* `jwt` 结构体的参数（令牌，和生存时间）
* Redis 结构体参数（Address 服务器地址，Pass 连接服务器所需要的密码）

### 2.2 api 的服务器上下文文件（servicecontext）

* config
* 中间件字段（rest.Middleware）
* 客户端 rpc 服务的接口，这些接口对应着具体某个服务（sys）的一些操作，这些操作（config 的服务）又有一些对应的子操作（configAdd，等对 config 进行处理的子操作）
* 服务器上下文的初始化：
  * 通过使用配置文件的 **rpc** 配置，对 **rpc** 客户端进行初始化，通过客户端创建服务（通过客户端作为参数，获取实际的接口（`NewGrowthChangeHistoryService`），其中一些服务或者上下问，都是通过这中 **New** 开头的 `function` 进行实例化的）
  * **redis** 的创建
  * 中间件的创建（.Handle）

### 2.3 logic 文件

* 调用 `client` 的 `rpc function`，来处理，然后返回结果
  * 根据需求处理 `client rpc function` 的返沪结果，会根据需要调用多个 **rpc** 的服务
  * 根据返回结果处理请求处理的逻辑主体
  * 初始化返回结果，进行处理

### 2.4 middleware 中间件文件

* ` ` 返回一个 `http.HandlerFunc` 类型，中间件进行预处理的代码就在这部分实现
  * 实现一个中间件处理的 `HandlerFunc`，然后通过这个 function 的逻辑对请求进行预处理，如果符合结果，则调用 `next（http.HandlerFunc）` 下一个 **handler** 进行处理，可以是下一个中间件
  * 可以处理响应头，自己写一个 `response` 结构体，其中封装 `http.responseWirter`，作为参数，传给下一个 `HandlerFunc`，`HandlerFunc` 需要的 `ResponseWirter` 是一个**接口**，需要将接口的方法实现

**例子：CheckURL 中间件**

* 检查参数
* 检查 URL
* 获取用户可以访问的 URL（在 **redis** 中存储了 **userID** 对应的能访问的 **url**）
  * 判断登陆状态
  * 判断 **redis** 连接
* 检查访问权限，从 **redis** 获取的结果进行对比

### 2.5 总结

* 首先是需要编写的文件：**config**，**serviceContext**，**logic**，**middleware**，这四个部分
* **config** 文件主要是服务器的配置，如需要调用的客户端 **rpc** 配置，鉴权参数，**redis** 参数，用来初始化服务的
* **serviceContext** 文件主要是服务用到的一些接口的定义，以及组件，如各个模块的 **rpc** 服务接口，中间件，**redis**，以及接口的定义和组件的初始化
* **logic** 文件，主要是请求处理的逻辑（前提是请求能通过中间件过来），通过调用客户端 **rpc** 函数，处理请求中所涉及的服务
* **middleware** 文件，中间件，对请求进行预处理（检查 **url** 是否合法，对每个特定的请求写入日志） 
* 总体思路，先对 **config** 和 **serviceContext** 进行编写，去初始化服务的配置，和加入服务所需要用到的上下文，然后在编写中间件逻辑，因为中间件和后续的操作是解耦的，并不强关联，所以可以先编写中间件的逻辑，对请求进行处理或者一些其他的功能。最后是写 **logic** 的逻辑，来调用服务器上下文中的操作来处理请求（**rpc** 请求，数据库等）

## 3. RPC 代码逻辑

### 3.1 .proto 文件

* .proto 文件时 rpc 代码的主体，通过该文件生成 rpc 的代码框架
  *  `req message` 的结构体
  * `resp message` 的结构体
  * `rpc service`，其中包括了 **rpc function**，定义和 **handler** 的相似，api 调用的就是这个服务

* 代码生成部分
  * 会生成 message 对应的 go 格式的变量
  * 生成 rpc service 的 go 格式的接口
  * rpc 逻辑代码

### 3.2 代码构成

* **login** 的主体主要通过 logic 代码进行实现
* **ServiceContext** 存放数据库的 `Model`，通过 RPC 操作数据库
* **config** 文件中存放了 **jwt**，和 **mysql** 的 `DataSource`

### 3.3 总结

api 通过调用 rpc，使服务端处理数据库的一些操作，将对数据的操作即服务，全部放到服务器中，通过 rpc 来使服务器操作数据，这样 api 就不用获取数据库的连接，只让分布式的服务器去处理对应的服务。

## 4. 整体逻辑思路总结

* 首先 api 服务对应的是接受路由，然后通过路由进行转发操作调用 rpc 进行处理，目的是降低单一服务器的负载
* rpc 相当处理具体的业务逻辑，为 api 进行负载均衡，api 既然叫 api 只是提供了服务处理的接口，实际处理是用过 api 调用 rpc 的远程服务器来进行处理
* 实际的数据库的模型和一些不展示出来的接口，都放到服务器中，而不是使用 api 服务器进行处理， api 服务器只进行一些预处理，和对结果的处理
* api 不涉及业务逻辑，只传递请求和返回请求的处理结果，middleware 也不涉及业务，只请求做一些预处理
* rpc 才是设计到具体的业务处理内容

## 5.各模块逻辑

### 5.1 角色菜单管理

* 问题

  * roleId 和 userId 是对应的；但是我更新 roleId 的时候，**userId 对应的 redis 数据库并没有更新**

  * userId 对应的**权限 url 没有写入 redis 中的代码**

* 功能：

  * redis 在哪更新：在 **userInfo 的 logic 中更新**，rpc 通过 role，role_menu，user_role，进行 join，然后获取 userId 对应的 url
  * 这个 router 是在登陆进系统的时候会发起 **currentUser 这个请求**，用来更新当前 userId 的 url 权限

* 逻辑，将每个菜单分配一个 id，通过这个 id 来对应访问权限
  * 首先 rpc func 会将 RoleId 对应的记录删除
  * 然后为 MenusIds 中的每个 MenuId 创建一条记录
* 权限逻辑
  * 首先用户又对应的 roleId，roleId 有权限映射（int，url）
  * 用户在登陆的时候，会更新 redis 的权限，发起 currentUser 的 router 的请求
  * 然后这个路由会通过 UserInfo 去使用 join 查询 userId 对应的 url
  * 然后将结果更新到 redis，在后续访问的时候进行判断（checkUrl 默认跳过 currentUser 的 router 请求，还有其他的两个，selectAllData，queryMenuByRoleId）
  * 后续的其他访问，都会使用 checkUrl 进行检查

### 5.2 用户菜单管理

#### 5.2.1 用户搜索

* 包括用户的页面的所有内容，和单个搜索都是通过一个 **/user/list** 来进行获取的
  * 在 api 中，req 字段使用 optional，这样传入的参数就编程可选的了
  * 在 rpc 中调用 findAll 去获取
  * 在 Mysql model 中，使用 where = ‘1=1’，来进行全体查询，如果有条件，再通过判断，进行 AND 拼接查询
  * 
