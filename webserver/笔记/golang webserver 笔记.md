# golang webserver 笔记

## WebServer 基本结构

![image-20231011110753116](D:\github\CPPStudy\src\photo\image-20231011110753116.png)

## net/http 包

### 注册路由的函数

* HandleFunc()：处理http请求的 func

  ```go
  func HandleFunc(pattern string, handler func(ResponseWriter, *Request))
  func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) 
  ```

* Handle()：一个处理http请求的结构体

  ```go
  func (mux *ServeMux) Handle(pattern string, handler Handler)
  ```

  对 URL 模式进行处理：

  1. 检查合法性
  2. 创建映射 `map`
  3. 根据目录和非目录存入不同的位置做不同的操作：如果最后一个字符为 `/` ， 为目录，存放如 `es` 中，如果第一个字符是 `/`， 即 是 `hosts`

* ServerMux 结构体：相当于将 `net/http` 包中的一个路由器（router），是一个 `HTTP` 请求的多路复用器，实际上是用于将请求分发给处理函数（handlers）

  ```go
  type ServeMux struct {
      mu    sync.RWMutex        //用于保证并发安全性的互斥锁
      m     map[string]muxEntry //一个映射表，将URL模式映射到对应的处理程序。在处理HTTP请求时，ServeMux将使用此映射表来查找与请求URL路径匹配的处理程序
      es    []muxEntry          //一个按长度排序的URL模式条目的切片。这个切片是用来加速ServeMux的URL匹配操作的。在处理HTTP请求时，ServeMux会按照长度递减的顺序迭代这个切片，以便找到最长的匹配URL模式
      hosts bool                //标志位，表示ServeMux是否具有任何带有主机名的URL模式。如果是，则在处理HTTP请求时，ServeMux还需要匹配主机名。如果不是，则可以忽略主机名匹配
  }
  ```

  那 `Handle()` 或者 `HandleFunc()` 的第一个形参模式，就是构建这个映射表用的，`HandleFunc(http路径， http路径对应的处理函数)`

* muxEntry 结构体：相当于记录 URL 模式和处理函数的映射

  ```go
  type muxEntry struct {
      h       Handler //一个处理程序，它是用于处理与该URL模式匹配的HTTP请求的函数
      pattern string  //与该处理程序相关联的URL模式。在ServeMux中，pattern是映射到处理程序的关键字之一。在匹配请求路径时，ServeMux将使用pattern来判断请求是否与该条目匹配。
  }
  ```

* ServerMux 的自定义：

  ```go
  package main
  
  import (
      "fmt"
      "net/http"
  )
  
  type routeIndex struct {
      content string
  }
  
  func (route *routeIndex) ServeHTTP(w http.ResponseWriter, r *http.Request) {
      fmt.Fprintf(w, route.content)
  }
  
  func htmlHandler(w http.ResponseWriter, r *http.Request) {
      w.Header().Set("Content-Type", "text/html")
      html := `<!doctype  html>  
      <META  http-equiv="Content-Type"  content="text/html"  charset="utf-8">    
      <html  lang="zhCN">  
              <head>   
                      <title>Golang</title>  
                      <meta  name="viewport"  content="width=device-width,  initial-scale=1.0,  maximum-scale=1.0,  user-scalable=0;"  />   
              </head>     
              <body>        
              <div  id="app">Hello, HandleFunc World!</div>       
              </body>   
      </html>`
      fmt.Fprintf(w, html)
  }
  
  func main() {
      //自定义serveMux
    mux := http.NewServeMux()
      mux.Handle("/Handle", &routeIndex{content: "Hello, Handle World"})
      mux.HandleFunc("/HandleFunc", htmlHandler)
      //创建服务且监听
      http.ListenAndServe(":8080", mux)
  }
  ```

### handler 函数，请求处理函数

* 接口函数只要实现了 `ServeHTTP` 的都是可以成为 `handler`，调用方法是 `http.Handler.ServeHTTP`, 这个方法是用来处理 `request` 并且构建 `response` 的核心逻辑

* 接口类型：

  ```go
  type Handler interface {
      ServeHTTP(ResponseWriter, *Request)
  }
  ```



## WebServer 函数结构

### 需要实现的函数

* 首先是需要处理请求的 `Handler`， 接口函数的实现（带有 `ServeHTTP` 的结构体）
  * `ServeHTTP` 需要实现对 `request` 的处理，和 `response` 的返回
* 其次是新建 ServeMux