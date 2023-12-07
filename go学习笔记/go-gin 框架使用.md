# go-gin 框架学习

## 1. gin 源码分析

### 1.1 Engine

#### 1.1.1 Engine 源码

**服务的主体，同时，可以作为 handler 传递给 http.server，Engine 实现了 ServeHTTP 方法，所以可以作为 Handler 接口传入**

```go
type Engine struct {
    //中间件信息就存储在这个里面
	RouterGroup
    //是否启动自动重定向。例如:配置Handler时是/foo，
    //但实际发送的请求是/foo/。在启用本选项后，将会重定向到/foo
	RedirectTrailingSlash bool

    // 是否启动请求路由修复功能。
    // 启用过后，当/../foo找不到匹配路由时，会自动删除..部分路由，然后重新匹配知道找到匹配路由，如上路由就会被匹配到/foo
	RedirectFixedPath bool

    //启用后，如果找不到当前路由匹配的HTTP方法,
    //而在其他HTTP方法中能找到，则返回405响应码。
    HandleMethodNotAllowed bool
    //是否获取真正的客户端IP，而不是代理服务器IP(nginx等)，
    // 开启后将会从"X-Real-IP和X-Forwarded-For"中解析得到客户端IP
	ForwardedByClientIP    bool

	//启用后将在头部加入"X-AppEngine"标识，以便与PaaS集成
	AppEngine bool

    //启用后，将使用原有的URL.rawPath(没有对转义字符进行处理的，
    // 如%/+等)地址来进行解析，而不是使用URL.path来解析，默认为false
	UseRawPath bool

	//如果启用，则路径中的转义字符将不会被转义
	UnescapePathValues bool

	//设置用来缓存客户端发送的文件的缓冲区大小，默认：32MB
	MaxMultipartMemory int64

	//启用后将会删除多余的分隔符"/"
	RemoveExtraSlash bool

    //用于保存tmpl文件中用于引用变量的定界符，默认是"{{}}"，
    // 调用r.Delims("{[{", "}]}")可以修改
    delims           render.Delims
    //设置防止JSON劫持，在json字符串前加的逻辑代码,
    //默认是:"while(1);"
    secureJsonPrefix string
    //html文件解析器
    HTMLRender       render.HTMLRender
    //tmpl文件的内建函数列表，可以在tmpl文件中调用函数，使用
    //router.SetFuncMap(template.FuncMap{
    //      "formatAsDate": formatAsDate,
    //})可设置
    FuncMap          template.FuncMap
    // HandlersChain就是func(*Context)数组
    // 以下四个调用链中保存的就是在不同情况下回调的处理函数
    // 找不到匹配路由(404)
    allNoRoute       HandlersChain
    //返回405状态时会回调
    allNoMethod      HandlersChain
    //没有配置路由时回调，主要是代码测试时候使用的
    noRoute          HandlersChain
    //没有配置映射方法时回调，主要是代码测试时候使用的
    noMethod         HandlersChain
    //连接池用于保存与客户端的连接上下文(Context)
    pool             sync.Pool
    //路径搜索树，代码中配置的路由信息都以树节点的形式组织起来
    // 下面会详细介绍
	trees            methodTrees
}
```

#### 1.1.2 Engine 逻辑

**与 `RouterGroup` 嵌套依赖**

* **路由分组（RouterGroup）**（gon-zero 的 group：@server）
* 维护**路由树（methodTree）**
  * 对每个请求方法维护一个路由树，每次来请求时，现根据方法去找对应的路由树
  * 路由树通过前缀进行构建，有自己的一个结构体
* 一些基本设置
* `HandlersChain`，处理路由的一些列中间件，通过 `addRoute function` 被组织到路由树中

#### 1.1.3 使用逻辑

* 为 `Engine` 增加路由（router）
  * `router`：路由对应的路径，`Engine` 会将这个 `router` 加入到 `methodTrees` 中
  * `handler`：处理路由对应请求的逻辑，需要一个传入参数为 `*gin.Context` 的 `function` 就行，这个会被加入 `handlersChains` 中
* 为 `Engine` 增加中间件（middleware）
  * 自定义中间件方法，传入参数为 `HandlerFunc` 即 入参为 `*gin.Context` 的函数 
  * 中间件处理请求（但是不能读取 `Body`，不然下次读取 `Body` 会发生 **EOF** 错误），通过传入的 `*gin.Context` 进行操作上下文，通过 `Context.Next()` 方法调用下一个 `handler`，通过在该方法后面处理上下文，可以获取**请求返回的数据**
  * 处理流程
    * 处理求情之前的逻辑
    * 处理请求`c.Next()`
    * 处理请求之后的逻辑
* 为 `Engine` 增加路由分组（RouterGroup）
  * 专门为分组增加中间件
  * 为分组的 `Engine`（两者嵌套以来关系）增加 `Router`

1.1.4 注意事项

* 直接使用 `Use` 来增加中间件，那么，当前 `Engine` 下的所有路由（路由分组也包括）都会先通过这些中间件，才会进行其余的操作，所以初始的 `Engine` 只能设置公共的一些中间件（所有路由都会使用的，如日志等）
* 如果想要不同的路由使用不同的中间件，就需要创建路由分组，然后对分组设置中间件，进行嵌套
* 结构
  * Engine
    * RouterGroup：新的分组，要区分使用不同的中间件
      * Engine：当前分组的路由
        * RouterGroup：其下的使用更多的 middleware
          * Engine：当前分组的路由
            * ...
        * RouterGroup：使用不同的 middleware
          * Engine：当前分组的路由
            * ...
        * ...
    * RouterGroup：不同分组，不同中间件规则
    * ...

### 1.2 RouterGroup

#### 1.2.1 RouterGroup 源码

**Engine 嵌套关系，为分组重新创建一个 Engine，然后通过分组，访问这个分组下的 Router，区分不同的规则，以及中间件**

```go
type RouterGroup struct {
	Handlers HandlersChain
    basePath string
    //注意这里存在交叉依赖
	engine   *Engine
	root     bool
}
```

**RouterGroup 实现了 GET 等方法，每个方法包括 Use 都返回一个 IRoutes 接口**

```go
func (group *RouterGroup) Use(middleware ...HandlerFunc) IRoutes {
	group.Handlers = append(group.Handlers, middleware...)
	return group.returnObj()
}

func (group *RouterGroup) POST(relativePath string, handlers ...HandlerFunc) IRoutes {
	return group.handle(http.MethodPost, relativePath, handlers)
}

func (group *RouterGroup) handle(httpMethod, relativePath string, handlers HandlersChain) IRoutes {
    //计算最简洁的相对路径，去除多余符号
    absolutePath := group.calculateAbsolutePath(relativePath)
    //合并处理函数，就是讲我们自己编写的处理函数与中间件函数连接成一个处理链，
    //这样在路由匹配时不仅会调用我们编写的函数，也会调用中间件函数
    handlers = group.combineHandlers(handlers)
    //像向engine对象添加路由信息
    group.engine.addRoute(httpMethod, absolutePath, handlers)
    
	return group.returnObj()
}

type IRoutes interface {
	Use(...HandlerFunc) IRoutes

	Handle(string, string, ...HandlerFunc) IRoutes
	Any(string, ...HandlerFunc) IRoutes
	GET(string, ...HandlerFunc) IRoutes
	POST(string, ...HandlerFunc) IRoutes
	DELETE(string, ...HandlerFunc) IRoutes
	PATCH(string, ...HandlerFunc) IRoutes
	PUT(string, ...HandlerFunc) IRoutes
	OPTIONS(string, ...HandlerFunc) IRoutes
	HEAD(string, ...HandlerFunc) IRoutes

	StaticFile(string, string) IRoutes
	Static(string, string) IRoutes
	StaticFS(string, http.FileSystem) IRoutes
}

// Engine、RouterGroup 两个都是实现了 GET 等方法的结构体，所以可以作为 IRoutes 的返回值
func (group *RouterGroup) returnObj() IRoutes {
	if group.root {
		// 如果是根节点，加入的时候直接使用最上层的 Engine
		return group.engine  
	}
    // 不是跟节点，就需要继续对 RouterGroup 进行操作
	return group
}
```



#### 1.2.2 RouterGroup 逻辑

* 绑定中间件，与 `Engine` 方法相同
* 增加新的分组
* 绑定路由，直接使用 `GET` 方法，因为底层代码会返回 `IRoutes` 接口，这个接口规定了方法，这些方法，Engine、RouterGroup 都实现了
* `Use` 的逻辑不同，`Engine` 的 `Use` 会调用 `RouterGroup` 的 `Use`，因为中间件，说到底，就是为路由分组增加的，最上级的 `Engine` 增加中间件，也是给默认 `'/'` 路由分组增加的

### 1.3 methodTree

#### 1.3.1 methodTree  源码

**存储方法对应的路由，给个方法对应一个树**

```go
type methodTrees []methodTree

type methodTree struct {
    // HTTP方法名
    method string
    //真正的根节点指针
	root   *node
}

type node struct {
    //当前节点所达路径
    path      string        
    //用于记录子节点的与当前节点的最长公共前缀的后一个字符
    //例如:当前节点为/(根节点)，有两个子节点，全路径分别为:
    // /user、/form_table
    //那么当前节点的indices就是"uf"，用于快速索引到子节点的
    // 如果当前节点
    indices   string
    //子节点指针数组
    children  []*node
    //当前节点的处理链
    handlers  HandlersChain
    //匹配优先级，一般按照最长路径匹配原则设置
    priority  uint32
    //节点类型
    nType     nodeType
    //当前节点与子节点中所有参数的个数
    // 参数指的是REST参数，而不是GET/POST中提交的参数
    maxParams uint8
    //子节点是否为通配符节点
    wildChild bool
    //达到当前节点的完整路径
	fullPath  string
}

type nodeType uint8

const (
    //静态路由信息节点，默认值
    static nodeType = iota
    //根节点
    root
    //参数节点
    param
    //表示当前节点已经包含所有的REST参数了,节点出现 catchAll 之后，其子节点路由就不会在被捕获了：*name 的状态就是这个
	catchAll
)
```

**路由树的构建**

```go
func (n *node) addRoute(path string, handlers HandlersChain) {
    //记录完整的路径
    fullPath := path
    //随着匹配路径的增长优先级逐渐增大
    n.priority++
    //计算当前传入路径中有多少参数
	numParams := countParams(path)

    //如果当前节点为空，则生成一个新的根节点，并以此更新当前空节点
	// Empty tree
	if len(n.path) == 0 && len(n.children) == 0 {
		n.insertChild(numParams, path, fullPath, handlers)
		n.nType = root
		return
	}
    //初始化父节点的路径长度
    // 主要是看传入的路径和当前节点是否有公共前缀
	parentFullPathIndex := 0

walk:
	for {
        //如果当前路径中的参数个数大于父节点中记录的参数数目则更新为大值
        // 因为父节点记录的是自己和子节点中所有参数的最大个数
		if numParams > n.maxParams {
			n.maxParams = numParams
		}

		//寻找公共前缀的下标索引
		i := longestCommonPrefix(path, n.path)

        //如果当前节点的路径长度大于公共前缀的下标索引
        // 则说明当前节点路径和新加入的路径不存在包含关系
        // (/user、/user/:name这样就有包含关系)
        // 需要分裂成两个子节点，例如:
        // /user /form_table
        // 最初当前节点路径为/user,传入的新路径为/form_table
        // 则可计算得公共前缀的下标索引为1
        // 则当前节点的路径更新为"/"
        // 并分裂成两个子节点,路径分别为"user"、"form_table"

        // 这第一步就是将当前节点分裂成一个子节点
		if i < len(n.path) {
			child := node{
				path:      n.path[i:],
				wildChild: n.wildChild,
				indices:   n.indices,
				children:  n.children,
                handlers:  n.handlers,
                //由于分裂并不会增长匹配路径所以优先级不会增加
                //这里减一主要是在这个函数的开始部分默认就会+1
                //所以需要减掉
				priority:  n.priority - 1,
				fullPath:  n.fullPath,
			}

			//更新当前节点的最大参数值
			for _, v := range child.children {
				if v.maxParams > child.maxParams {
					child.maxParams = v.maxParams
				}
			}

			n.children = []*node{&child}
			//这里就更新成了公共前缀的后一个字符了
			n.indices = string([]byte{n.path[i]})
			n.path = path[:i]
			n.handlers = nil
			n.wildChild = false
			n.fullPath = fullPath[:parentFullPathIndex+i]
		}

        //这一步就是分裂成两个子节点中的第二步
        //将新加入的路径生成相应子节点，加入刚才被分裂的父节点当中去
		if i < len(path) {
			path = path[i:]
            //判断当前节点是否是通配符节点，如:*name、:name
			if n.wildChild {
				parentFullPathIndex += len(n.path)
				n = n.children[0]
				n.priority++

				// Update maxParams of the child node
				if numParams > n.maxParams {
					n.maxParams = numParams
				}
				numParams--

				// 检查当前路径是否还未遍历完
				if len(path) >= len(n.path) && n.path == path[:len(n.path)] {
                    //如果发现还有子路可以遍历则递归
					if len(n.path) >= len(path) || path[len(n.path)] == '/' {
						continue walk
					}
				}

				pathSeg := path
				if n.nType != catchAll {
					pathSeg = strings.SplitN(path, "/", 2)[0]
				}
				prefix := fullPath[:strings.Index(fullPath, pathSeg)] + n.path
				panic("'" + pathSeg +
					"' in new path '" + fullPath +
					"' conflicts with existing wildcard '" + n.path +
					"' in existing prefix '" + prefix +
					"'")
			}
            
			c := path[0]

			//如果当前节点是参数节点，且有一个子节点则递归遍历
			if n.nType == param && c == '/' && len(n.children) == 1 {
				parentFullPathIndex += len(n.path)
				n = n.children[0]
				n.priority++
				continue walk
			}
            
            //这个是查看有没有现存的子节点与传入路径相匹配，
            // 有则进入该子节点进行递归
			for i, max := 0, len(n.indices); i < max; i++ {
				if c == n.indices[i] {
					parentFullPathIndex += len(n.path)
					i = n.incrementChildPrio(i)
					n = n.children[i]
					continue walk
				}
			}

            //如果传入路径非":"、"*"开头则说明是普通静态节点
            // 直接构造后插入，并添加子节点索引
			if c != ':' && c != '*' {
				n.indices += string([]byte{c})
				child := &node{
					maxParams: numParams,
					fullPath:  fullPath,
				}
				n.children = append(n.children, child)
				n.incrementChildPrio(len(n.indices) - 1)
				n = child
			}
			n.insertChild(numParams, path, fullPath, handlers)
			return
		}

		//如果当前节点已经有处理函数，则说明之前已经有注册过这个路由了，发出警告，并更新处理函数
		if n.handlers != nil {
			panic("handlers are already registered for path '" + fullPath + "'")
		}
		n.handlers = handlers
		return
	}
}
```

#### 1.3.2 methodTree 构造逻辑



#### 1.3.3 methodTree 注意事项

* `.name` 的路由，会使当前路由的 `nType` 变成 **catchAll**，导致其子路由都不能被捕获









[参考博客]([Golang之Gin框架源码解读——第一章_go gin源码解析-CSDN博客](https://blog.csdn.net/qq_41115702/article/details/106008260))



## 2. gin 框架使用

**gin 虽然能只能作为服务器，但是更像一个路由配置的框架，http 服务器的配置，还是需要 http.Server 来做**

### 2.1 gin 功能划分

* 路由
  * 路由
  * 路由组
* 处理请求
  * 请求解析
  * 请求内容绑定（相当于请求解析）
  * 请求处理
  * 发送响应
* 中间件
* 网页渲染
* 资源发送
* 多服务
* 服务器配置

### 2.2 路由

#### 2.2.1 路由

* 根路由，使用默认的路由组 `"/"`

  ```go
  func main(){
      r := gin.Default()
      r.GET("/get", func(c *gin.Context){
          return
      })
      r.Run(":8080")
  }
  ```

* 对初始化获取的 `Engine`，路由都是放在默认路由组中 `"/"`

* 路由添加格式 `METHOD("router", func)`

  * **METHOD**：请求方法名
  * **router**：路由的路径
  * **func**：处理请求的 `handler`，传入的参数为上下文指针（*gin.Context），请求的所有内容都被解析到这个上下文中，同时，这个上下文因为是指针，所以其中的内容，在其他的函数中处理，这里都能接收到改变，中间件可以用来获取**返回的响应**

#### 2.2.2 路由组

* 有 `go-zero` 中的 `@server(group: )` 相同，为当前路由划分组，通过对**分组**增加**共享的中间件**，以及处理逻辑，以便于管理

* 使用 `Engine` 的 `Group` 函数，来设置路由组，该函数返回值为 `RouterGroup` ，可以通过这个返回值操作这个路由组

* 路由组下可以继续设置 `Engine`（不同设置，初始化会有），来进行嵌套（直接嵌套 `Group`），同时路由组的 `function`，与 `Engine` 基本相似，因为这两个都要作为接口（IRoutes）传递

  ```go
  func AuthRequired() func(*gin.Context) {
  	return func(ctx *gin.Context) {
  		ctx.Writer.Write([]byte("AuthRequired\n"))
  	}
  }
  
  func main() {
  	r := gin.New()
  
  	r.Use(gin.Logger())
  	r.Use(gin.Recovery())
  
  	r.GET("/benchmark", func(c *gin.Context) {
  		c.String(http.StatusOK, "/benchmark/%s", "back")
  	})
  	auth := r.Group("/")
  	auth.Use(AuthRequired())
  	{
  		auth.POST("/login", func(c *gin.Context) {
  			c.String(http.StatusOK, "login")
  		})
  		testing := auth.Group("testing")
  		testing.GET("/analytics", func(c *gin.Context) {
  			c.String(http.StatusOK, "analytics")
  		})
  	}
  	r.Run(":8888")
  }
  ```

#### 2.2.3 路由参数

* 路由匹配

  * `:`模糊匹配一个
  * `*` 模糊匹配零个以上

  ```go
  func main() {
  	router := gin.Default()
  
  	// 此 handler 将匹配 /user/john 但不会匹配 /user/ 或者 /user
  	router.GET("/user/:name", func(c *gin.Context) {
  		name := c.Param("name")
  		c.String(http.StatusOK, "Hello %s", name)
  	})
  
  	// 此 handler 将匹配 /user/john/ 和 /user/john/send
  	// 如果没有其他路由匹配 /user/john，它将重定向到 /user/john/
  	router.GET("/user/:name/*action", func(c *gin.Context) {
  		name := c.Param("name")
  		action := c.Param("action")
  		message := name + " is " + action
  		c.String(http.StatusOK, message)
  	})
  
  	router.Run(":8080")
  }
  ```

### 2.3 请求处理

#### 2.3.1 请求解析

##### 2.3.1.1 gin.Context

* `gin.Context` 会对请求进行解析，结果以键值对的形式存储，可以通过 `gin.Context` 获取 Header、Body等内容

##### 2.3.1.2 url 参数、表单内容获取

* 请求内容

  ```http
  POST /post?id=1234&page=1 HTTP/1.1
  Content-Type: application/x-www-form-urlencoded
  
  name=manu&message=this_is_great
  ```

  * `Body` 信息编码使用：`x-www-form-urlencoded`
  *  同时附带 `url` 的 `Param`

* 获取参数

  ```go
  func main() {
  	router := gin.Default()
  
  	router.POST("/post", func(c *gin.Context) {
  
  		id := c.Query("id")
  		page := c.DefaultQuery("page", "0")  // 后面是默认的返回结果
  		name := c.PostForm("name")
  		message := c.PostForm("message")
  
  		fmt.Printf("id: %s; page: %s; name: %s; message: %s", id, page, name, message)
  	})
  	router.Run(":8080")
  }
  ```

  * `url` 参数通过 `Query` 进行查找
  * `x-www-form-urlencoded` 参数通过 `PostForm` 进行查找

##### 2.3.1.3 获取路由参数

* 获取路由的信息

  ```go
  func main() {
  	router := gin.Default()
  
  	// 此 handler 将匹配 /user/john 但不会匹配 /user/ 或者 /user
  	router.GET("/user/:name", func(c *gin.Context) {
  		name := c.Param("name")
  		c.String(http.StatusOK, "Hello %s", name)
  	})
  
  	// 此 handler 将匹配 /user/john/ 和 /user/john/send
  	// 如果没有其他路由匹配 /user/john，它将重定向到 /user/john/
  	router.GET("/user/:name/*action", func(c *gin.Context) {
  		name := c.Param("name")
  		action := c.Param("action")
  		message := name + " is " + action
  		c.String(http.StatusOK, message)
  	})
  
  	router.Run(":8080")
  }
  ```

#### 2.3.2 请求内容绑定

**将请求中的内容，绑定到预设的结构体中**

##### 2.3.2.1 只绑定 url 参数（忽略 post 数据）

* 使用 `ShouldBindQuery` 函数，绑定 `url` 参数

  ```go
  package main
  
  import (
  	"log"
  
  	"github.com/gin-gonic/gin"
  )
  
  type Person struct {
  	Name    string `form:"name"`
  	Address string `form:"address"`
  }
  
  func main() {
  	route := gin.Default()
  	route.Any("/testing", startPage)
  	route.Run(":8085")
  }
  
  func startPage(c *gin.Context) {
  	var person Person
  	if c.ShouldBindQuery(&person) == nil {
  		log.Println("====== Only Bind By Query String ======")
  		log.Println(person.Name)
  		log.Println(person.Address)
  	}
  	c.String(200, "Success")
  }
  ```

  * 首先需要构建绑定数据的结构体
  * 然后使用函数进行绑定

##### 2.3.2.2 重复绑定 body

* `body` 被读取一次后，再次读取就会发生 `EOF` 错误，所以想要重复绑定，就需要使用 `ShouldBindBodyWith`

  ```go
  type formA struct {
    Foo string `json:"foo" xml:"foo" binding:"required"`
  }
  
  type formB struct {
    Bar string `json:"bar" xml:"bar" binding:"required"`
  }
  
  func SomeHandler(c *gin.Context) {
    objA := formA{}
    objB := formB{}
    // 读取 c.Request.Body 并将结果存入上下文。
    if errA := c.ShouldBindBodyWith(&objA, binding.JSON); errA == nil {
      c.String(http.StatusOK, `the body should be formA`)
    // 这时, 复用存储在上下文中的 body。
    } else if errB := c.ShouldBindBodyWith(&objB, binding.JSON); errB == nil {
      c.String(http.StatusOK, `the body should be formB JSON`)
    // 可以接受其他格式
    } else if errB2 := c.ShouldBindBodyWith(&objB, binding.XML); errB2 == nil {
      c.String(http.StatusOK, `the body should be formB XML`)
    } else {
      ...
    }
  }
  ```

##### 2.3.2.3 映射查询字符串或表单参数

**多个结果，以 map[string]string 存储其中的多个结果**

* 请求

  ```http
  POST /post?ids[a]=1234&ids[b]=hello HTTP/1.1
  Content-Type: application/x-www-form-urlencoded
  
  names[first]=thinkerou&names[second]=tianou
  ```

* 使用 `QueryMap，PostFormMap` 进行获取结果

  ```go
  func main() {
  	router := gin.Default()
  
  	router.POST("/post", func(c *gin.Context) {
  
  		ids := c.QueryMap("ids")
  		names := c.PostFormMap("names")
  
  		fmt.Printf("ids: %v; names: %v", ids, names)
  	})
  	router.Run(":8080")
  }
  ```

* 结果

  ```go
  ids: map[b:hello a:1234], names: map[second:tianou first:thinkerou]
  ```

##### 2.3.2.4 模型的绑定（验证）

**Gin 提供了两种绑定方法**

* Type - Must bind
  - **Methods** - `Bind`, `BindJSON`, `BindXML`, `BindQuery`, `BindYAML`
  - **Behavior** - 这些方法属于 `MustBindWith` 的具体调用。 如果发生绑定错误，则请求终止，并触发 `c.AbortWithError(400, err).SetType(ErrorTypeBind)`。响应状态码被设置为 400 并且 `Content-Type` 被设置为 `text/plain; charset=utf-8`。 如果您在此之后尝试设置响应状态码，Gin会输出日志 `[GIN-debug] [WARNING] Headers were already written. Wanted to override status code 400 with 422`。 如果您希望更好地控制绑定，考虑使用 `ShouldBind` 等效方法。
* Type - Should bind
  - **Methods** - `ShouldBind`, `ShouldBindJSON`, `ShouldBindXML`, `ShouldBindQuery`, `ShouldBindYAML`
  - **Behavior** - 这些方法属于 `ShouldBindWith` 的具体调用。 如果发生绑定错误，Gin 会返回错误并由开发者处理错误和请求。
* `Must bind` 发生错误直接返固定的 `Bad Request`；`Should bind` 可以由开发者处理错误请求

* 将不同的请求体格式（json，xml 等）绑定到预定义的结构体中

  ```go
  // 绑定 JSON
  type Login struct {
  	User     string `form:"user" json:"user" xml:"user"  binding:"required"`
  	Password string `form:"password" json:"password" xml:"password" binding:"required"`
  }
  
  func main() {
  	router := gin.Default()
  
  	// 绑定 JSON ({"user": "manu", "password": "123"})
  	router.POST("/loginJSON", func(c *gin.Context) {
  		var json Login
  		if err := c.ShouldBindJSON(&json); err != nil {
  			c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
  			return
  		}
  		
  		if json.User != "manu" || json.Password != "123" {
  			c.JSON(http.StatusUnauthorized, gin.H{"status": "unauthorized"})
  			return
  		} 
  		
  		c.JSON(http.StatusOK, gin.H{"status": "you are logged in"})
  	})
  
  	// 绑定 XML (
  	//	<?xml version="1.0" encoding="UTF-8"?>
  	//	<root>
  	//		<user>manu</user>
  	//		<password>123</password>
  	//	</root>)
  	router.POST("/loginXML", func(c *gin.Context) {
  		var xml Login
  		if err := c.ShouldBindXML(&xml); err != nil {
  			c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
  			return
  		}
  		
  		if xml.User != "manu" || xml.Password != "123" {
  			c.JSON(http.StatusUnauthorized, gin.H{"status": "unauthorized"})
  			return
  		} 
  		
  		c.JSON(http.StatusOK, gin.H{"status": "you are logged in"})
  	})
  
  	// 绑定 HTML 表单 (user=manu&password=123)
  	router.POST("/loginForm", func(c *gin.Context) {
  		var form Login
  		// 根据 Content-Type Header 推断使用哪个绑定器。
  		if err := c.ShouldBind(&form); err != nil {
  			c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
  			return
  		}
  		
  		if form.User != "manu" || form.Password != "123" {
  			c.JSON(http.StatusUnauthorized, gin.H{"status": "unauthorized"})
  			return
  		} 
  		
  		c.JSON(http.StatusOK, gin.H{"status": "you are logged in"})
  	})
  
  	// 监听并在 0.0.0.0:8080 上启动服务
  	router.Run(":8080")
  }
  ```

##### 2.3.2.5 绑定 HTML 复选框

* 根据复选框的 `name` 进行绑定，获取复选框的值

  ```go
  type myForm struct {
  	Colors []string `form:"colors[]"`
  }
  
  func main() {
  	router := gin.Default()
  
  	router.LoadHTMLGlob("../html/*")
  	router.GET("/", func(c *gin.Context) {
  		c.HTML(200, "form.html", nil)
  	})
  	router.POST("/colors", func(c *gin.Context) {
  		var fakeForm myForm
  		c.ShouldBind(&fakeForm)
  		c.JSON(http.StatusOK, gin.H{"color": fakeForm.Colors})
  	})
  
  	router.Run(":8080")
  }
  ```

* HTML 

  ```html
  <form action="/" method="POST">
      <p>Check some colors</p>
      <label for="red">Red</label>
      <input type="checkbox" name="colors[]" value="red" id="red" />
      <label for="green">Green</label>
      <input type="checkbox" name="colors[]" value="green" id="green" />
      <label for="blue">Blue</label>
      <input type="checkbox" name="colors[]" value="blue" id="blue" />
      <input type="submit" />
  </form>
  ```

* 结果

  ```go
  {"color":["red","green","blue"]}
  ```

##### 2.3.2.6 绑定 Url

* Uri ：URI（Uniform Resource Identifier）是一种用于标识和定位资源的字符串格式。它是 Web 上标准的资源标识方式，用于唯一地标识和定位资源，例如网页、图像、视频、API 端点等。

  ```go
  package main
  
  import "github.com/gin-gonic/gin"
  
  type Person struct {
  	ID   string `uri:"id" binding:"required,uuid"`
  	Name string `uri:"name" binding:"required"`
  }
  
  func main() {
  	route := gin.Default()
  	route.GET("/:name/:id", func(c *gin.Context) {
  		var person Person
  		if err := c.ShouldBindUri(&person); err != nil {
  			c.JSON(400, gin.H{"msg": err.Error()})
  			return
  		}
  		c.JSON(200, gin.H{"name": person.Name, "uuid": person.ID})
  	})
  	route.Run(":8088")
  }
  ```

* 测试 url

  ```shell
  $ curl -v localhost:8088/thinkerou/987fbc97-4bed-5078-9f07-9141ba07c9f3
  $ curl -v localhost:8088/thinkerou/not-uuid
  ```

##### 2.3.2.7 绑定查询字符串或表单数据

* 首先编写绑定的结构体，`ShouldBind` 会根据结构体内容进行绑定

  ```go
  package main
  
  import (
  	"log"
  	"time"
  
  	"github.com/gin-gonic/gin"
  )
  
  type Person struct {
  	Name     string    `form:"name"`
  	Address  string    `form:"address"`
  	Birthday time.Time `form:"birthday" time_format:"2006-01-02" time_utc:"1"`
  }
  
  func main() {
  	route := gin.Default()
  	route.GET("/testing", startPage)
  	route.Run(":8085")
  }
  
  func startPage(c *gin.Context) {
  	var person Person
  	// 如果是 `GET` 请求，只使用 `Form` 绑定引擎（`query`）。
  	// 如果是 `POST` 请求，首先检查 `content-type` 是否为 `JSON` 或 `XML`，然后再使用 `Form`（`form-data`）。
  	// 查看更多：https://github.com/gin-gonic/gin/blob/master/binding/binding.go#L88
  	if c.ShouldBind(&person) == nil {
  		log.Println(person.Name)
  		log.Println(person.Address)
  		log.Println(person.Birthday)
  	}
  
  	c.String(200, "Success")
  }
  ```

* 测试

  ```go
  curl -X GET "localhost:8085/testing?name=appleboy&address=xyz&birthday=1992-03-15"
  ```

##### 2.3.2.8 绑定表单数据至自定义结构体

* 结构体的每个字段都需要设置 `tag`，已经设置过了的，就不能在设置了

  ```go
  type StructA struct {
      FieldA string `form:"field_a"`
  }
  
  type StructB struct {
      NestedStruct StructA
      FieldB string `form:"field_b"`
  }
  
  type StructC struct {
      NestedStructPointer *StructA
      FieldC string `form:"field_c"`
  }
  
  type StructD struct {
      NestedAnonyStruct struct {
          FieldX string `form:"field_x"`
      }
      FieldD string `form:"field_d"`
  }
  
  func GetDataB(c *gin.Context) {
      var b StructB
      c.Bind(&b)
      c.JSON(200, gin.H{
          "a": b.NestedStruct,
          "b": b.FieldB,
      })
  }
  
  func GetDataC(c *gin.Context) {
      var b StructC
      c.Bind(&b)
      c.JSON(200, gin.H{
          "a": b.NestedStructPointer,
          "c": b.FieldC,
      })
  }
  
  func GetDataD(c *gin.Context) {
      var b StructD
      c.Bind(&b)
      c.JSON(200, gin.H{
          "x": b.NestedAnonyStruct,
          "d": b.FieldD,
      })
  }
  
  func main() {
      r := gin.Default()
      r.GET("/getb", GetDataB)
      r.GET("/getc", GetDataC)
      r.GET("/getd", GetDataD)
  
      r.Run()
  }
  ```

* 不支持的格式

  ```go
  type StructX struct {
      X struct {} `form:"name_x"` // 有 form
  }
  
  type StructY struct {
      Y StructX `form:"name_y"` // 有 form
  }
  
  type StructZ struct {
      Z *StructZ `form:"name_z"` // 有 form
  }
  ```

##### 2.3.2.9 绑定总结

**总是，绑定可以显示绑定固定结构，也可以模糊绑定，让服务器自己判断（ShouldBind），然后最重要的，就是要写绑定内容的结构体**

##### 2.3.2.10 各类参数查询

* 字符串参数：url 后面的键值对
  * 使用 Query
* 路径参数：url 中的变量部分
  * 使用 Param
* 表单参数：POST 请求
  * 使用 PostForm
  * 或者 PostFormMap
* JSON 参数：Body 体
  * 使用 ShouldBinding 之类的函数



#### 2.3.3 请求处理

##### 2.3.3.1 获取 cookie

* 使用 **c.Cookie(key)** 来获取 `cookie`

##### 2.3.3.2 从 reader 读取数据（Body）

* 从其他响应体中获取数据，然后发送（用于从`io.Reader`接口中读取数据，并将其作为响应的数据发送给客户端）

  ```go
  func main() {
  	router := gin.Default()
  	router.GET("/someDataFromReader", func(c *gin.Context) {
  		response, err := http.Get("https://raw.githubusercontent.com/gin-gonic/logo/master/color.png")
  		if err != nil || response.StatusCode != http.StatusOK {
  			c.Status(http.StatusServiceUnavailable)
  			return
  		}
  
  		reader := response.Body
  		contentLength := response.ContentLength
  		contentType := response.Header.Get("Content-Type")
  
  		extraHeaders := map[string]string{
  			"Content-Disposition": `attachment; filename="gopher.png"`,
  		}
  		
          // 
  		c.DataFromReader(http.StatusOK, contentLength, contentType, reader, extraHeaders)
  	})
  	router.Run(":8080")
  }
  ```

##### 2.3.3.3 模型绑定和验证（验证器）

**对 request 增加增加 tag，用来进行反序列化时使用，通过对 request 添加函数（字段和tag的绑定：错误的映射），来验证错误**

* 绑定返回错误的本质是：字段.Tag（name.require）
* 验证器做法：请求设置一个方法，用来获取绑定返回错误的 `key(字段.Tag)`
* 验证器流程： 遍历错误（字段.Tag），去 request.Method 返回的 map 中查找（v.Field() + "." + v.Tag()） 看看是否存在

Gin使用 [**go-playground/validator/v10**](https://github.com/go-playground/validator) 进行验证。 查看标签用法的全部[文档](https://pkg.go.dev/github.com/go-playground/validator/v10#hdr-Baked_In_Validators_and_Tags).

```go
package main

import (
	"net/http"
	"reflect"
	"time"

	"github.com/gin-gonic/gin"
	"github.com/gin-gonic/gin/binding"
	"github.com/go-playground/validator/v10"
)

// Booking 包含绑定和验证的数据。
type Booking struct {
	CheckIn  time.Time `form:"check_in" binding:"required,bookabledate" time_format:"2006-01-02"`
	CheckOut time.Time `form:"check_out" binding:"required,gtfield=CheckIn,bookabledate" time_format:"2006-01-02"`
}

var bookableDate validator.Func = func(fl validator.FieldLevel) bool {
	date, ok := fl.Field().Interface().(time.Time)
	if ok {
		today := time.Now()
		if today.After(date) {
			return false
		}
	}
	return true
}

func main() {
	route := gin.Default()

	if v, ok := binding.Validator.Engine().(*validator.Validate); ok {
		v.RegisterValidation("bookabledate", bookableDate)
	}

	route.GET("/bookable", getBookable)
	route.Run(":8085")
}

func getBookable(c *gin.Context) {
	var b Booking
	if err := c.ShouldBindWith(&b, binding.Query); err == nil {
		c.JSON(http.StatusOK, gin.H{"message": "Booking dates are valid!"})
	} else {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
	}
}
```

#### 2.3.4 响应请求

##### 2.3.4.1 HTML 渲染，发送 HTML 响应

* 使用 `LoadHTMLGlob()` 或者 `LoadHTMLFiles()` 来设置 `html` 资源位置，使用 `c.HTML` 发送模板

  ```go
  func main() {
  	router := gin.Default()
  	router.LoadHTMLGlob("templates/*")
  	//router.LoadHTMLFiles("templates/template1.html", "templates/template2.html")
  	router.GET("/index", func(c *gin.Context) {
  		c.HTML(http.StatusOK, "index.tmpl", gin.H{
  			"title": "Main website",   // 将待匹配的内容进行替换
  		})
  	})
  	router.Run(":8080")
  }
  ```

  模板内容

  ```go
  <html>
  	<h1>
  		{{ .title }}    // .title 待匹配的内容
  	</h1>
  </html>
  ```

* 不同目录下名称相同的模板：要用的目录匹配通配符

  * `**`：匹配零个或多个目录级别，表示可以递归匹配任意深度的子目录。
  * `*`：匹配当前目录级别下的任意文件或目录。

  ```go
  func main() {
  	router := gin.Default()
  	router.LoadHTMLGlob("templates/**/*")
  	router.GET("/posts/index", func(c *gin.Context) {
  		c.HTML(http.StatusOK, "posts/index.tmpl", gin.H{
  			"title": "Posts",
  		})
  	})
  	router.GET("/users/index", func(c *gin.Context) {
  		c.HTML(http.StatusOK, "users/index.tmpl", gin.H{
  			"title": "Users",
  		})
  	})
  	router.Run(":8080")
  }
  ```

* 自定义模板渲染器

  ```go
  import "html/template"
  
  func main() {
  	router := gin.Default()
  	html := template.Must(template.ParseFiles("file1", "file2"))
  	router.SetHTMLTemplate(html)
  	router.Run(":8080")
  }
  ```

* 自定义分隔符

  ```go
  	r := gin.Default()
  	r.Delims("{[{", "}]}")
  	r.LoadHTMLGlob("/path/to/templates")
  ```

* 自定义模板功能

  ```go
  import (
      "fmt"
      "html/template"
      "net/http"
      "time"
  
      "github.com/gin-gonic/gin"
  )
  
  func formatAsDate(t time.Time) string {
      year, month, day := t.Date()
      return fmt.Sprintf("%d/%02d/%02d", year, month, day)
  }
  
  func main() {
      router := gin.Default()
      router.Delims("{[{", "}]}")
      router.SetFuncMap(template.FuncMap{
          "formatAsDate": formatAsDate,
      })
      router.LoadHTMLFiles("./testdata/template/raw.tmpl")
  
      router.GET("/raw", func(c *gin.Context) {
          c.HTML(http.StatusOK, "raw.tmpl", map[string]interface{}{
              "now": time.Date(2017, 07, 01, 0, 0, 0, 0, time.UTC),
          })
      })
  
      router.Run(":8080")
  }
  ```

  * raw.tmpl

    ```go
    Date: {[{.now | formatAsDate}]}
    ```

  * 结果

    ```go
    Date: 2017/07/01
    ```

##### 2.3.4.2 HTTP 2 server 推送

> HTTP/2 Server Push 是一种HTTP/2协议中的功能，它允许服务器在客户端请求之前主动推送相关资源，以提高页面加载性能和用户体验。
>
> 在传统的HTTP/1.1协议中，客户端需要发送多个请求来获取一个页面所需的所有资源，例如HTML、CSS、JavaScript和图像文件等。这会导致额外的延迟和资源请求次数，影响页面加载速度。
>
> 而HTTP/2通过引入Server Push机制，可以在服务器端主动推送与请求的主资源相关的其他资源，以减少客户端的请求次数和等待时间。服务器可以根据页面的需要，提前推送页面所需的资源，使得这些资源在客户端请求之前就已经被缓存，从而加快页面加载速度。
>
> 下面是HTTP/2 Server Push的基本工作流程：
>
> 1. 客户端发起一个请求，例如请求HTML页面。
> 2. 服务器接收到该请求，并解析请求的URL以确定所需的资源。
> 3. 服务器根据请求的URL和已知的页面依赖关系，判断需要推送哪些资源给客户端。
> 4. 服务器将相关资源主动推送给客户端，不需要客户端发送额外的请求。
> 5. 客户端收到服务器推送的资源，并缓存这些资源。
> 6. 客户端继续处理原始请求的响应，例如处理HTML页面。
> 7. 客户端在处理HTML页面时，如果遇到已经缓存的推送资源的URL，可以直接从缓存中加载，而无需再次请求服务器。
>
> 通过使用HTTP/2 Server Push，服务器可以在客户端请求之前预测和主动推送相关资源，减少了额外的往返延迟和请求次数，从而提高了页面加载速度和性能。

* 提前发送客户端需要的资源

  ```go
  package main
  
  import (
  	"html/template"
  	"log"
  
  	"github.com/gin-gonic/gin"
  )
  
  var html = template.Must(template.New("https").Parse(`
  <html>
  <head>
    <title>Https Test</title>
    <script src="/assets/app.js"></script>
  </head>
  <body>
    <h1 style="color:red;">Welcome, Ginner!</h1>
  </body>
  </html>
  `))
  
  func main() {
  	r := gin.Default()
  	r.Static("/assets", "./assets")
  	r.SetHTMLTemplate(html)
  
  	r.GET("/", func(c *gin.Context) {
  		if pusher := c.Writer.Pusher(); pusher != nil {
  			// 使用 pusher.Push() 做服务器推送
  			if err := pusher.Push("/assets/app.js", nil); err != nil {
  				log.Printf("Failed to push: %v", err)
  			}
  		}
  		c.HTML(200, "https", gin.H{
  			"status": "success",
  		})
  	})
  
  	// 监听并在 https://127.0.0.1:8080 上启动服务
  	r.RunTLS(":8080", "./testdata/server.pem", "./testdata/server.key")
  }
  ```


##### 2.3.4.3 跨域 cross

* 使用 JSONP 进行跨域处理，跨域属于前端发生，后端操作

  * 还需要检查有没有回调函数

  ```go
  package main
  
  import (
  	"github.com/gin-gonic/gin"
  )
  
  func main() {
  	router := gin.Default()
  
  	router.GET("/data", func(c *gin.Context) {
  		data := map[string]interface{}{
  			"message": "Hello, JSONP!",
  		}
  		
          // 检查回调函数
  		callback := c.Query("callback")
  		if callback != "" {
  			c.JSONP(200, gin.H{
  				"callback": callback,
  				"data":     data,
  			})
  		} else {
  			c.JSON(200, data)
  		}
  	})
  
  	router.Run(":8080")
  }
  ```

  ```http
  GET /data
  # 无回调响应
  {
      "message": "Hello, JSONP!"
  }
  
  
  GET /data?callback=myCallback
  # 回调响应
  myCallback({
      "callback": "myCallback",
      "data": {
          "message": "Hello, JSONP!"
      }
  });
  ```

##### 2.3.4.4 进行字面编码 PureJSON

* 通常，`JSON` 使用 `unicode` 替换特殊 `HTML` 字符，例如 `<` 变为 `\ u003c`。如果要按字面对这些字符进行编码，则可以使用 `PureJSON`
* 使用 `c.PureJSON` 发送

##### 2.3.4.5 SecureJSON 防止 json 劫持

* 使用 `SecureJSON` 防止 `json` 劫持。如果给定的结构是数组值，则默认预置 `"while(1),"` 到响应体

##### 2.3.4.6 XML/JSON/YAML/ProtoBuf 渲染，响应不同的请求

```go
func main() {
	r := gin.Default()

	// gin.H 是 map[string]interface{} 的一种快捷方式
	r.GET("/someJSON", func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{"message": "hey", "status": http.StatusOK})
	})

	r.GET("/moreJSON", func(c *gin.Context) {
		// 你也可以使用一个结构体
		var msg struct {
			Name    string `json:"user"`
			Message string
			Number  int
		}
		msg.Name = "Lena"
		msg.Message = "hey"
		msg.Number = 123
		// 注意 msg.Name 在 JSON 中变成了 "user"
		// 将输出：{"user": "Lena", "Message": "hey", "Number": 123}
		c.JSON(http.StatusOK, msg)
	})

	r.GET("/someXML", func(c *gin.Context) {
		c.XML(http.StatusOK, gin.H{"message": "hey", "status": http.StatusOK})
	})

	r.GET("/someYAML", func(c *gin.Context) {
		c.YAML(http.StatusOK, gin.H{"message": "hey", "status": http.StatusOK})
	})

	r.GET("/someProtoBuf", func(c *gin.Context) {
		reps := []int64{int64(1), int64(2)}
		label := "test"
		// protobuf 的具体定义写在 testdata/protoexample 文件中。
		data := &protoexample.Test{
			Label: &label,
			Reps:  reps,
		}
		// 请注意，数据在响应中变为二进制数据
		// 将输出被 protoexample.Test protobuf 序列化了的数据
		c.ProtoBuf(http.StatusOK, data)
	})

	// 监听并在 0.0.0.0:8080 上启动服务
	r.Run(":8080")
}
```

##### 2.3.4.7 文件上传

* 单文件

  ```go
  func main() {
  	router := gin.Default()
  	// 为 multipart forms 设置较低的内存限制 (默认是 32 MiB)
  	router.MaxMultipartMemory = 8 << 20  // 8 MiB
  	router.POST("/upload", func(c *gin.Context) {
  		// 单文件
  		file, _ := c.FormFile("file")
  		log.Println(file.Filename)
  
  		dst := "./" + file.Filename
  		// 上传文件至指定的完整文件路径
  		c.SaveUploadedFile(file, dst)
  
  		c.String(http.StatusOK, fmt.Sprintf("'%s' uploaded!", file.Filename))
  	})
  	router.Run(":8080")
  }
  ```

  ```shell
  curl -X POST http://localhost:8080/upload \
    -F "file=@/Users/appleboy/test.zip" \
    -H "Content-Type: multipart/form-data"
  ```

* 多文件

  ```go
  func main() {
  	router := gin.Default()
  	// 为 multipart forms 设置较低的内存限制 (默认是 32 MiB)
  	router.MaxMultipartMemory = 8 << 20  // 8 MiB
  	router.POST("/upload", func(c *gin.Context) {
  		// Multipart form
  		form, _ := c.MultipartForm()
  		files := form.File["upload[]"]
  
  		for _, file := range files {
  			log.Println(file.Filename)
  
  			// 上传文件至指定目录
  			c.SaveUploadedFile(file, dst)
  		}
  		c.String(http.StatusOK, fmt.Sprintf("%d files uploaded!", len(files)))
  	})
  	router.Run(":8080")
  }
  ```

  ```shell
  curl -X POST http://localhost:8080/upload \
    -F "upload[]=@/Users/appleboy/test1.zip" \
    -F "upload[]=@/Users/appleboy/test2.zip" \
    -H "Content-Type: multipart/form-data"
  ```

##### 2.3.4.8 重定向

* 重定向

  ```go
  r.GET("/test", func(c *gin.Context) {
  	c.Redirect(http.StatusMovedPermanently, "http://www.google.com/")
  })
  
  r.POST("/test", func(c *gin.Context) {
  	c.Redirect(http.StatusFound, "/foo")
  })
  ```

* 路由重定向，使用 `HandleContext`

  ```go
  r.GET("/test", func(c *gin.Context) {
      c.Request.URL.Path = "/test2"
      r.HandleContext(c)
  })
  r.GET("/test2", func(c *gin.Context) {
      c.JSON(200, gin.H{"hello": "world"})
  })
  ```

### 2.4 中间件

**中间件大部分需要自己实现**

#### 2.4.1 默认中间件

* 不使用默认

  ```go
  r := gin.New()
  ```

* 使用默认

  ```go
  r := gin.Default()
  ```

#### 2.4.2 BasicAuth 中间件

```go
// 模拟一些私人数据
var secrets = gin.H{
	"foo":    gin.H{"email": "foo@bar.com", "phone": "123433"},
	"austin": gin.H{"email": "austin@example.com", "phone": "666"},
	"lena":   gin.H{"email": "lena@guapa.com", "phone": "523443"},
}

func main() {
	r := gin.Default()

	// 路由组使用 gin.BasicAuth() 中间件
	// gin.Accounts 是 map[string]string 的一种快捷方式
	authorized := r.Group("/admin", gin.BasicAuth(gin.Accounts{
		"foo":    "bar",
		"austin": "1234",
		"lena":   "hello2",
		"manu":   "4321",
	}))

	// /admin/secrets 端点
	// 触发 "localhost:8080/admin/secrets
	authorized.GET("/secrets", func(c *gin.Context) {
		// 获取用户，它是由 BasicAuth 中间件设置的
		user := c.MustGet(gin.AuthUserKey).(string)
		if secret, ok := secrets[user]; ok {
			c.JSON(http.StatusOK, gin.H{"user": user, "secret": secret})
		} else {
			c.JSON(http.StatusOK, gin.H{"user": user, "secret": "NO SECRET :("})
		}
	})

	// 监听并在 0.0.0.0:8080 上启动服务
	r.Run(":8080")
}
```

#### 2.4.3 使用中间件

* 使用 `Router.Use()`

#### 2.4.4 在中间件中使用 Goroutine

* 当在中间件或 handler 中启动新的 Goroutine 时，**不能**使用原始的上下文，必须使用只读副本

  * 保证并发安全性
  * 隔离性，goroutine 使用自己的上下文
  * 防止意外修改

  ```go
  func main() {
  	r := gin.Default()
  
  	r.GET("/long_async", func(c *gin.Context) {
  		// 创建在 goroutine 中使用的副本
  		cCp := c.Copy()
  		go func() {
  			// 用 time.Sleep() 模拟一个长任务。
  			time.Sleep(5 * time.Second)
  
  			// 请注意您使用的是复制的上下文 "cCp"，这一点很重要
  			log.Println("Done! in path " + cCp.Request.URL.Path)
  		}()
  	})
  
  	r.GET("/long_sync", func(c *gin.Context) {
  		// 用 time.Sleep() 模拟一个长任务。
  		time.Sleep(5 * time.Second)
  
  		// 因为没有使用 goroutine，不需要拷贝上下文
  		log.Println("Done! in path " + c.Request.URL.Path)
  	})
  
  	// 监听并在 0.0.0.0:8080 上启动服务
  	r.Run(":8080")
  }
  ```

#### 2.4.5 自定义中间件

* 中间件作为函数进行传递，函数格式：gin.HandlerFunc

  ```go
  func MiddleWare() gin.HandlerFunc {
      return func(c *gin.Context) {
          
      }
  }
  
  # 中间件使用
  r.Use(MiddleWare())
  ```

  

### 2.5 日志相关

**一般使用日志中间件进行记录日志**

#### 2.5.1 记录日志

* 单日志默认输出到控制台

* 多日志输出使用 `gin.DefaultWriter = io.MultiWriter(f)`， `f` 为创建的日志文件

  `gin.DefaultWriter = io.MultiWriter(f, os.Stdout)`

#### 2.5.2 定义路由日志的格式

* 可以使用 `gin.DebugPrintRouteFunc` 指定格式

  ```go
  import (
  	"log"
  	"net/http"
  
  	"github.com/gin-gonic/gin"
  )
  
  func main() {
  	r := gin.Default()
  	gin.DebugPrintRouteFunc = func(httpMethod, absolutePath, handlerName string, nuHandlers int) {
  		log.Printf("endpoint %v %v %v %v\n", httpMethod, absolutePath, handlerName, nuHandlers)
  	}
  
  	r.POST("/foo", func(c *gin.Context) {
  		c.JSON(http.StatusOK, "foo")
  	})
  
  	r.GET("/bar", func(c *gin.Context) {
  		c.JSON(http.StatusOK, "bar")
  	})
  
  	r.GET("/status", func(c *gin.Context) {
  		c.JSON(http.StatusOK, "ok")
  	})
  
  	// 监听并在 0.0.0.0:8080 上启动服务
  	r.Run()
  }
  ```

#### 2.5.3 自定义日志文件

```go
func main() {
	router := gin.New()
	// LoggerWithFormatter 中间件会写入日志到 gin.DefaultWriter
	// 默认 gin.DefaultWriter = os.Stdout
	router.Use(gin.LoggerWithFormatter(func(param gin.LogFormatterParams) string {
		// 你的自定义格式
		return fmt.Sprintf("%s - [%s] \"%s %s %s %d %s \"%s\" %s\"\n",
				param.ClientIP,
				param.TimeStamp.Format(time.RFC1123),
				param.Method,
				param.Path,
				param.Request.Proto,
				param.StatusCode,
				param.Latency,
				param.Request.UserAgent(),
				param.ErrorMessage,
		)
	}))
	router.Use(gin.Recovery())
	router.GET("/ping", func(c *gin.Context) {
		c.String(200, "pong")
	})
	router.Run(":8080")
}
```



### 2.6 HTTP 配置

#### 2.6.1 配置服务

**将 Router 作为 Handler，传给 http.Server**

#### 2.6.2 启动多个服务

* 使用 `errgroup.Group`

```go
package main

import (
	"log"
	"net/http"
	"time"

	"github.com/gin-gonic/gin"
	"golang.org/x/sync/errgroup"
)

var (
	g errgroup.Group
)

func router01() http.Handler {
	e := gin.New()
	e.Use(gin.Recovery())
	e.GET("/", func(c *gin.Context) {
		c.JSON(
			http.StatusOK,
			gin.H{
				"code":  http.StatusOK,
				"error": "Welcome server 01",
			},
		)
	})

	return e
}

func router02() http.Handler {
	e := gin.New()
	e.Use(gin.Recovery())
	e.GET("/", func(c *gin.Context) {
		c.JSON(
			http.StatusOK,
			gin.H{
				"code":  http.StatusOK,
				"error": "Welcome server 02",
			},
		)
	})

	return e
}

func main() {
	server01 := &http.Server{
		Addr:         ":8080",
		Handler:      router01(),
		ReadTimeout:  5 * time.Second,
		WriteTimeout: 10 * time.Second,
	}

	server02 := &http.Server{
		Addr:         ":8081",
		Handler:      router02(),
		ReadTimeout:  5 * time.Second,
		WriteTimeout: 10 * time.Second,
	}

	g.Go(func() error {
		return server01.ListenAndServe()
	})

	g.Go(func() error {
		return server02.ListenAndServe()
	})

	if err := g.Wait(); err != nil {
		log.Fatal(err)
	}
}
```

### 2.7 资源相关

#### 2.7.1 静态文件服务

* 加载静态文件

  ```go
  func main() {
  	router := gin.Default()
  	router.Static("/assets", "./assets")
  	router.StaticFS("/more_static", http.Dir("my_file_system"))
  	router.StaticFile("/favicon.ico", "./resources/favicon.ico")
  
  	// 监听并在 0.0.0.0:8080 上启动服务
  	router.Run(":8080")
  }
  ```

#### 2.7.2 静态资源嵌入

* 你可以使用 [go-assets](https://github.com/jessevdk/go-assets) 将静态资源打包到可执行文件中。

  ```go
  func main() {
  	r := gin.New()
  
  	t, err := loadTemplate()
  	if err != nil {
  		panic(err)
  	}
  	r.SetHTMLTemplate(t)
  
  	r.GET("/", func(c *gin.Context) {
  		c.HTML(http.StatusOK, "/html/index.tmpl", nil)
  	})
  	r.Run(":8080")
  }
  
  // loadTemplate 加载由 go-assets-builder 嵌入的模板
  func loadTemplate() (*template.Template, error) {
  	t := template.New("")
  	for name, file := range Assets.Files {
  		if file.IsDir() || !strings.HasSuffix(name, ".tmpl") {
  			continue
  		}
  		h, err := ioutil.ReadAll(file)
  		if err != nil {
  			return nil, err
  		}
  		t, err = t.New(name).Parse(string(h))
  		if err != nil {
  			return nil, err
  		}
  	}
  	return t, nil
  }
  ```

  
