# GORM 学习笔记

# 1 基础知识

## 1.1 模型定义

**模型-即数据库表属性对应的字段，用来存储结果**

* 标准的 struct，添加 tag 用来帮助序列化和反序列化的

**约定**

* GORM 倾向于约定优于配置 默认情况下，GORM 使用 `ID` 作为主键，使用结构体名的 `蛇形复数` 作为表名，字段名的 `蛇形` 作为列名，并使用 `CreatedAt`、`UpdatedAt` 字段追踪创建、更新时间

**自定义默认 Model**

* `gorm` 自带的默认 `model`

* ```go
  // gorm.Model 的定义
  type Model struct {
    ID        uint           `gorm:"primaryKey"`
    CreatedAt time.Time
    UpdatedAt time.Time
    DeletedAt gorm.DeletedAt `gorm:"index"`
  }
  ```

  * **tag**：`gorm:"primarykey"` 标记数据库的属性，同时可以用来创建数据库表

**高级选项**

* 字段级权限控制

  ```go
  type User struct {
    Name string `gorm:"<-:create"` // 允许读和创建
    Name string `gorm:"<-:update"` // 允许读和更新
    Name string `gorm:"<-"`        // 允许读和写（创建和更新）
    Name string `gorm:"<-:false"`  // 允许读，禁止写
    Name string `gorm:"->"`        // 只读（除非有自定义配置，否则禁止写）
    Name string `gorm:"->;<-:create"` // 允许读和写
    Name string `gorm:"->:false;<-:create"` // 仅创建（禁止从 db 读）
    Name string `gorm:"-"`  // 通过 struct 读写会忽略该字段
    Name string `gorm:"-:all"`        // 通过 struct 读写、迁移会忽略该字段
    Name string `gorm:"-:migration"`  // 通过 struct 迁移会忽略该字段
  }
  ```

  * `<-` 表示写入 ， `:create` 表示写入的形式
  * `->` 表示读取，`:false` 表示禁止读取
  * `:` 是对控制的描述
  * `-` 表示读写忽略字段， `:all` 表示读写、迁移都忽略，`:migration` 表示

* 创建\更新时间追踪（在创建的时候自动填充当前时间）

  ```go
  type User struct {
    CreatedAt time.Time // 在创建时，如果该字段值为零值，则使用当前时间填充
    UpdatedAt int       // 在创建时该字段值为零值或者在更新时，使用当前时间戳秒数填充
    Updated   int64 `gorm:"autoUpdateTime:nano"` // 使用时间戳纳秒数填充更新时间
    Updated   int64 `gorm:"autoUpdateTime:milli"` // 使用时间戳毫秒数填充更新时间
    Created   int64 `gorm:"autoCreateTime"`      // 使用时间戳秒数填充创建时间
  }
  ```

  * 要保存时间戳，要将 `time.Time` 改成 `int`

* 结构体嵌入（使用 embedded tag 进行标记）

  ```go
  type Author struct {
      Name  string
      Email string
  }
  
  type Blog struct {
    ID      int
    Author  Author `gorm:"embedded"`
    Upvotes int32
  }
  // 等效于
  type Blog struct {
    ID    int64
    Name  string
    Email string
    Upvotes  int32
  }
  ```

  加入前缀

  ```go
  type Blog struct {
    ID      int
    Author  Author `gorm:"embedded;embeddedPrefix:author_"`
    Upvotes int32
  }
  // 等效于
  type Blog struct {
    ID          int64
    AuthorName string
    AuthorEmail string
    Upvotes     int32
  }
  ```

**字段标签**

| 标签名                 | 说明                                                         |
| :--------------------- | :----------------------------------------------------------- |
| column                 | 指定 db 列名                                                 |
| type                   | 列数据类型，推荐使用兼容性好的通用类型，例如：所有数据库都支持 bool、int、uint、float、string、time、bytes 并且可以和其他标签一起使用，例如：`not null`、`size`, `autoIncrement`… 像 `varbinary(8)` 这样指定数据库数据类型也是支持的。在使用指定数据库数据类型时，它需要是完整的数据库数据类型，如：`MEDIUMINT UNSIGNED not NULL AUTO_INCREMENT` |
| serializer             | 指定将数据序列化或反序列化到数据库中的序列化器, 例如: `serializer:json/gob/unixtime` |
| size                   | 定义列数据类型的大小或长度，例如 `size: 256`                 |
| primaryKey             | 将列定义为主键                                               |
| unique                 | 将列定义为唯一键                                             |
| default                | 定义列的默认值                                               |
| precision              | 指定列的精度                                                 |
| scale                  | 指定列大小                                                   |
| not null               | 指定列为 NOT NULL                                            |
| autoIncrement          | 指定列为自动增长                                             |
| autoIncrementIncrement | 自动步长，控制连续记录之间的间隔                             |
| embedded               | 嵌套字段                                                     |
| embeddedPrefix         | 嵌入字段的列名前缀                                           |
| autoCreateTime         | 创建时追踪当前时间，对于 `int` 字段，它会追踪时间戳秒数，您可以使用 `nano`/`milli` 来追踪纳秒、毫秒时间戳，例如：`autoCreateTime:nano` |
| autoUpdateTime         | 创建/更新时追踪当前时间，对于 `int` 字段，它会追踪时间戳秒数，您可以使用 `nano`/`milli` 来追踪纳秒、毫秒时间戳，例如：`autoUpdateTime:milli` |
| index                  | 根据参数创建索引，多个字段使用相同的名称则创建复合索引，查看 [索引](https://gorm.io/zh_CN/docs/indexes.html) 获取详情 |
| uniqueIndex            | 与 `index` 相同，但创建的是唯一索引                          |
| check                  | 创建检查约束，例如 `check:age > 13`，查看 [约束](https://gorm.io/zh_CN/docs/constraints.html) 获取详情 |
| <-                     | 设置字段写入的权限， `<-:create` 只创建、`<-:update` 只更新、`<-:false` 无写入权限、`<-` 创建和更新权限 |
| ->                     | 设置字段读的权限，`->:false` 无读权限                        |
| -                      | 忽略该字段，`-` 表示无读写，`-:migration` 表示无迁移权限，`-:all` 表示无读写迁移权限 |
| comment                | 迁移时为字段添加注释                                         |

## 1.2 连接数据库

