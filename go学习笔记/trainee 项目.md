# trainee 项目

## 1.Log

### 1.1 日志库使用 zerolog

* 使用 `zerolog` 高性能日志库，作为日志输出，这个日志库可以使用
* 使用日志要同时输出到日志文件以及控制台
  * 使用 os.Stdout 和日志文件，同时进行输出，`io.MultiWriter` 组织这两个输出的文件

### 1.2 日志文件管理：lumberjack

* 管理日志文件

## 2. 接口可视化：swagger

### 2.1 安装

* 项目中进行安装 

  ```shell
  go get github.com/swaggo/swag/cmd/swag
  ```

* 没有 `swag.exe` 解决

  * 先看看 `GOPATH` 下的 `bin` 文件夹中有没有，有的话添加以下环境变量
  * 没有的话，去 `C:\Users\[user]\go\pkg\mod\github.com\swaggo\swag@v1.16.2\cmd\swag` 文件夹下，里面有 `mian.go`，使用 `go install`，安装一下

* 初始化

  ```shell
  $ swag init
  ```

  * 这个初始化会出现找不到 `main.go` 的错误，需要限定路径

  * 如果接口文件不出现，出现 `No operations defined in spec` 的错误，需要在项目根目录下使用

    ```shell
    $ swag init -g .\cmd\main.go
    ```

    才可以，不然扫描不到项目的其他文件中的注释

* 安装 `gin-swagger`

  ```shell
  go get -u github.com/swaggo/gin-swagger
  ```

### 2.2 编写文档

* 首先：在 `main.go` 中加入 `BasePath` 的注解，标记路径，`swagger` 会从 `main.go` 进入，然后找其后的注解

  ```go
  // @title 标题
  // @version 1.0 版本
  // @description 描述
  // @termsOfService http://swagger.io/terms/
  // @contact.name 联系人
  // @contact.url http://www.swagger.io/support
  // @contact.email 
  // @license.name Apache 2.0
  // @license.url http://www.apache.org/licenses/LICENSE-2.0.html
  // @host 
  // @BasePath /api
  // @query.collection.format multi
  func main() {
  	r := Router()
  	r.Run(":80")
  }
  ```

* 然后：在 `Router HandlerFunc` 的位置加入后续的路由的文档

  ```go
  // Test godoc
  // @Summary 摘要
  // @Schemes
  // @Description 描述
  // @Tags 标签
  // @Param Id query int true "参数描述" 分别表示 ：@Param 参数名 位置（query或path或body） 类型 是否必需 注释
  // @Param object query models.ParamList false "查询参数"
  // @Accept json
  // @Produce json
  // @Success 200 {object} Model._ResponseList 返回结果 200 类型（object就是结构体） 类型 注释
  // @Router /user [get]
  func Test(c *gin.Context)
  {
      
  }
  ```

* 参数类型，必须是实际存在的，写完之后会变成白色注解

* 如果使用了外部依赖需要加入参数

  ```shell
  $ swag init --parseDependency --parseInternal
  ```

### 2.3 加入路由

* 首先在根 Router 文件中导入包

  ```go
  import (
      "cmd/docs"   // 使用 swag init 生成的文档位置
      swaggerfiles "github.com/swaggo/files"
  	ginSwagger "github.com/swaggo/gin-swagger"
  )
  ```

* 增加 `swagger` 路由

  ```go
  // 启动 swagger api 路由
  docs.SwaggerInfo.BasePath = "/api/v1/users"
  httpEngine.GET("/swagger/*any", ginSwagger.WrapHandler(swaggerFiles.Handler))
  ```

### 2.4 更新路由的文档

```go
$ swag init -g .\cmd\main.go
```

