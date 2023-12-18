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

### 1.2.1 数据库驱动

**连接数据库需要使用对应的数据库驱动**

* 默认驱动：直接使用 `DSN` 打开（mysql.Open）

* 自定义驱动：通过 `mysql.Config("gorm.io/driver/mysql")` 中的 `DriverName` 字段自定义驱动

  ```go
  import (
    _ "example.com/my_mysql_driver"
    "gorm.io/driver/mysql"
    "gorm.io/gorm"
  )
  
  db, err := gorm.Open(mysql.New(mysql.Config{
    DriverName: "my_mysql_driver",
    DSN: "gorm:gorm@tcp(localhost:9910)/gorm?charset=utf8&parseTime=True&loc=Local", // data source name, 详情参考：https://github.com/go-sql-driver/mysql#dsn-data-source-name
  }), &gorm.Config{})
  ```

* 使用现有驱动同时，使用高级配置（使用 mysql.Config 进行初始化）

  ```go
  db, err := gorm.Open(mysql.New(mysql.Config{
    DSN: "gorm:gorm@tcp(127.0.0.1:3306)/gorm?charset=utf8&parseTime=True&loc=Local", // DSN data source name
    DefaultStringSize: 256, // string 类型字段的默认长度
    DisableDatetimePrecision: true, // 禁用 datetime 精度，MySQL 5.6 之前的数据库不支持
    DontSupportRenameIndex: true, // 重命名索引时采用删除并新建的方式，MySQL 5.7 之前的数据库和 MariaDB 不支持重命名索引
    DontSupportRenameColumn: true, // 用 `change` 重命名列，MySQL 8 之前的数据库和 MariaDB 不支持重命名列
    SkipInitializeWithVersion: false, // 根据当前 MySQL 版本自动配置
  }), &gorm.Config{})
  ```

* 使用现有连接创建（使用 mysql.Config 进行初始化）

  ```go
  import (
    "database/sql"
    "gorm.io/driver/mysql"
    "gorm.io/gorm"
  )
  
  sqlDB, err := sql.Open("mysql", "mydb_dsn")
  gormDB, err := gorm.Open(mysql.New(mysql.Config{
    Conn: sqlDB,
  }), &gorm.Config{})
  ```

### 1.2.2 数据库类型及DSN

* mysql
  * DSN："gorm:gorm@tcp(localhost:9910)/gorm?charset=utf8&parseTime=True&loc=Local"
  * 默认驱动：mysql.Open()
  * 自定义配置及驱动：mysql.New   mysql.Config

* postgreSQL
  * DSN："host=localhost user=gorm password=gorm dbname=gorm port=9920 sslmode=disable TimeZone=Asia/Shanghai"
  * 默认驱动：postgres.Open()
  * 自定义配置及驱动：postgres.New   postgres.Config
* SQLite
  * 默认驱动：sqlite.Open("gorm.db")
* SQL Server
  * DSN："sqlserver://gorm:LoremIpsum86@localhost:9930?database=gorm"
  * 默认驱动：sqlserver.Open()
* TiDB
  * 兼容 mysql 协议，使用 mysql.Open 创建即可

[dsn 参考](https://github.com/go-sql-driver/mysql#dsn-data-source-name)

### 1.2.3 数据库连接池及其他属性

**通过 database/sql 返回的 *DB 指针来进行维护**

```go
// 获取通用数据库对象 sql.DB，然后使用其提供的功能
sqlDB, err := db.DB()

// Ping
sqlDB.Ping()

// Close
sqlDB.Close()

// 返回数据库统计信息
sqlDB.Stats()

// SetMaxIdleConns 用于设置连接池中空闲连接的最大数量。
sqlDB.SetMaxIdleConns(10)

// SetMaxOpenConns 设置打开数据库连接的最大数量。
sqlDB.SetMaxOpenConns(100)

// SetConnMaxLifetime 设置了连接可复用的最大时间。
sqlDB.SetConnMaxLifetime(time.Hour)
```

### 1.2.4 ClickHouse

**用于数据分析和数据处理的开源列式数据库管理系统，在 GORM 中，ClickHouse 被作为其中一个数据库驱动的支持**

```go
  // 自动迁移 (这是GORM自动创建表的一种方式--译者注)
  db.AutoMigrate(&User{})
  // 设置表选项
  db.Set("gorm:table_options", "ENGINE=Distributed(cluster, default, hits)").AutoMigrate(&User{})
```

## 1.3 数据库操作

### 1.3.1 创建表

* 迁移（AutoMigrate），自定义模型（表的属性结构体）

  ```go
  type Product struct {
    ID    uint `gorm:"primaryKey;default:auto_random()"`
    Code  string
    Price uint
  }
  
  // 自动迁移 (这是GORM自动创建表的一种方式--译者注)
  db.AutoMigrate(&Product{})
  ```

  * 会自动创建结构体名对应的一个表，同时 `tag` 会作为表属性的约束等

### 1.3.2 创建记录

* 使用 Create 创建记录（需要传结构体指针，结构体大，降低结构体复制的开销，同时返回插入记录的 ID）

  * 单记录：传递结构体指针，会返回插入后的记录 ID
  * 多记录：传递列表，列表中为**结构体指针**或者**结构体列表的指针**
  * result：返回 Error，RowsAffected（插入记录的条数）

* 使用指定字段创建记录：等同于 `INSERT INTO 'users' ('name','age','created_at') VALUES ("jinzhu", 18, "2020-07-04 11:05:21.775")`

  * 为指定字段赋值（Select 选取指定字段）

    ```go
    db.Select("Name", "Age", "CreatedAt").Create(&user)
    ```

  * 忽略特定字段的值（Omit 与 Select 相反）：`INSERT INTO 'users' ('birthday','updated_at') VALUES ("2020-01-01 00:00:00.000", "2020-07-04 11:05:21.775")`

    ```go
    db.Omit("Name", "Age", "CreatedAt").Create(&user)
    ```

* 批量插入（高效处理，传入结构体列表切片）

  * 创建 `Batch`

    ```go
    var users = []User{{Name: "jinzhu_1"}, ...., {Name: "jinzhu_10000"}}
    
    // batch size 100
    db.CreateInBatches(users, 100)
    ```

  * 设置 Session 属性

    * 在数据库打开的时候，通过 gorm.Config 设置全局的属性

      ```go
      db, err := gorm.Open(sqlite.Open("gorm.db"), &gorm.Config{
        CreateBatchSize: 1000,
      })
      ```

    * 创建会话的时候设置会话属性，通过 gorm.Session 设置

      ```go
      db := db.Session(&gorm.Session{CreateBatchSize: 1000})
      ```

* 使用 `Map` 创建

  ```go
  db.Model(&User{}).Create(map[string]interface{}{
    "Name": "jinzhu", "Age": 18,
  })
  
  // batch insert from `[]map[string]interface{}{}`
  db.Model(&User{}).Create([]map[string]interface{}{
    {"Name": "jinzhu_1", "Age": 18},
    {"Name": "jinzhu_2", "Age": 20},
  })
  ```
  
* 使用 SQL 表达式创建。Context Valuer 创建记录

  * SQL 表达式是通过 `clause.Expr` 来创建的
  * `clause.Expr` 结构体具有两个字段：`SQL` 和 `Values`。`SQL` 字段用于存储要执行的原始 SQL 表达式，而 `Values` 字段用于存储与表达式相关的参数值。

  ```go
  // Create from map
  db.Model(User{}).Create(map[string]interface{}{
    "Name": "jinzhu",
    "Location": clause.Expr{SQL: "ST_PointFromText(?)", Vars: []interface{}{"POINT(100 100)"}},
  })
  // INSERT INTO `users` (`name`,`location`) VALUES ("jinzhu",ST_PointFromText("POINT(100 100)"));
  
  // Create from customized data type
  type Location struct {
      X, Y int
  }
  
  // Scan implements the sql.Scanner interface
  func (loc *Location) Scan(v interface{}) error {
    // Scan a value into struct from database driver
  }
  
  func (loc Location) GormDataType() string {
    return "geometry"
  }
  
  func (loc Location) GormValue(ctx context.Context, db *gorm.DB) clause.Expr {
    return clause.Expr{
      SQL:  "ST_PointFromText(?)",
      Vars: []interface{}{fmt.Sprintf("POINT(%d %d)", loc.X, loc.Y)},
    }
  }
  
  type User struct {
    Name     string
    Location Location
  }
  
  db.Create(&User{
    Name:     "jinzhu",
    Location: Location{X: 100, Y: 100},
  })
  // INSERT INTO `users` (`name`,`location`) VALUES ("jinzhu",ST_PointFromText("POINT(100 100)"))
  ```

* 关联创建

  * 模型嵌套，同时创建两个

    ```go
    type CreditCard struct {
      gorm.Model
      Number   string
      UserID   uint
    }
    
    type User struct {
      gorm.Model
      Name       string
      CreditCard CreditCard
    }
    
    db.Create(&User{
      Name: "jinzhu",
      CreditCard: CreditCard{Number: "411111111111"}
    })
    // INSERT INTO `users` ...
    // INSERT INTO `credit_cards` ...
    ```

  * 跳过空值和只创建某个值

    ```go
    db.Omit("CreditCard").Create(&user)
    
    // skip all associations
    db.Omit(clause.Associations).Create(&user)
    ```

* 创建的时候附带默认值：使用 Tag

  ```go
  type User struct {
    ID   int64
    Name string `gorm:"default:galeone"`
    Age  int64  `gorm:"default:18"`
  }
  ```

  * 迁移时跳过默认值：`default:(-)`

    ```go
    type User struct { 
      ID         string  `gorm:"default:uuid_generate_v3()"`  // db func
       FirstName string
       LastName   string
       Age        uint8
       FullName   string  `gorm:"->;type:GENERATED ALWAYS AS (concat(firstname,' ',lastname) ));默认值:(-);"`
     }
    ```

* Upsert 及冲突

  * `db.Clauses`  查询定制

  * `clause.OnConflict{}` 冲突解决

  * 五种冲突解决的方法

    * 不做任何事：`clause.OnConflict{DoNothing: true}` 
    * 更新 `key` 对应的 `value`：`DoUpdates: clause.Assignments(map[string]interface{}{"role": "user"})`
    * 使用 `sql` 表达式更新：`DoUpdates: clause.Assignments(map[string]interface{}{"count": gorm.Expr("GREATEST(count, VALUES(count))")})`
    * 更新指定的列：`DoUpdates: clause.AssignmentColumns([]string{"name", "age"})`
    * 更新所有： `UpdateAll: true`

    ```go
    import "gorm.io/gorm/clause"
    
    // Do nothing on conflict
    db.Clauses(clause.OnConflict{DoNothing: true}).Create(&user)
    
    // Update columns to default value on `id` conflict
    db.Clauses(clause.OnConflict{
    // 默认列发生冲突
      Columns:   []clause.Column{{Name: "id"}},
    // 
      DoUpdates: clause.Assignments(map[string]interface{}{"role": "user"}),
    }).Create(&users)
    // MERGE INTO "users" USING *** WHEN NOT MATCHED THEN INSERT *** WHEN MATCHED THEN UPDATE SET ***; SQL Server
    // INSERT INTO `users` *** ON DUPLICATE KEY UPDATE ***; MySQL
    
    // Use SQL expression
    db.Clauses(clause.OnConflict{
      Columns:   []clause.Column{{Name: "id"}},
      DoUpdates: clause.Assignments(map[string]interface{}{"count": gorm.Expr("GREATEST(count, VALUES(count))")}),
    }).Create(&users)
    // INSERT INTO `users` *** ON DUPLICATE KEY UPDATE `count`=GREATEST(count, VALUES(count));
    
    // Update columns to new value on `id` conflict
    db.Clauses(clause.OnConflict{
      Columns:   []clause.Column{{Name: "id"}},
      DoUpdates: clause.AssignmentColumns([]string{"name", "age"}),
    }).Create(&users)
    // MERGE INTO "users" USING *** WHEN NOT MATCHED THEN INSERT *** WHEN MATCHED THEN UPDATE SET "name"="excluded"."name"; SQL Server
    // INSERT INTO "users" *** ON CONFLICT ("id") DO UPDATE SET "name"="excluded"."name", "age"="excluded"."age"; PostgreSQL
    // INSERT INTO `users` *** ON DUPLICATE KEY UPDATE `name`=VALUES(name),`age`=VALUES(age); MySQL
    
    // Update all columns to new value on conflict except primary keys and those columns having default values from sql func
    db.Clauses(clause.OnConflict{
      UpdateAll: true,
    }).Create(&users)
    // INSERT INTO "users" *** ON CONFLICT ("id") DO UPDATE SET "name"="excluded"."name", "age"="excluded"."age", ...;
    // INSERT INTO `users` *** ON DUPLICATE KEY UPDATE `name`=VALUES(name),`age`=VALUES(age), ...; MySQL
    
    ```

    


### 1.3.* 创建 Hook

**在创建和保存之前，使用 Hook 提前做判断，这个 Hook 是表结构体的方法，GORM 会先调用操作的表结构体的方法**

* 规范：`BeforeSave`、`AfterSave`、`BeforeCreate`、`AfterCreate`

  ```go
  func (u *User) BeforeCreate(tx *gorm.DB) (err error) {
    u.UUID = uuid.New()
  
      if u.Role == "admin" {
          return errors.New("invalid role")
      }
      return
  }
  ```

* 跳过 `Hook`：使用 `gorm.Session` 修改会话的设置

  ```go
  DB.Session(&gorm.Session{SkipHooks: true}).Create(&users)
  
  DB.Session(&gorm.Session{SkipHooks: true}).CreateInBatches(users, 100)
  ```
  
* 查询 `Hook`：**AfterFind**，同样是结构体的方法

* 更新 `Hook`：**BeforeUpdate**

### 1.3.3 查询

#### 1.3.3.1 查询单个对象

**不使用 Find，性能不稳定，会查询整张表只返回第一个对象**

* `First`：主键升序
* `Take`：没有排序字段，`LIMIT 1` 有随机性
* `Last`： 获取最后一条记录，主键降序

**错误检查**

* `errors.Is(result.Error, gorm.ErrRecordNotFound)`

**First 和 Last 生效场景**

* 目标 `struct` 是指针，或者通过 `db.Model()` 指定 `model` 的时候方法才有效
* 两种组合
  * 使用模型指针，不使用 `Model`
  * 使用 `map`，不使用模型指针，使用 `Model`

#### 1.3.3.2 检索全部对象

* **Find 传递列表指针**

  ```go
  // Get all records
  var users []User
  result := db.Find(&users)
  // SELECT * FROM users;
  
  result.RowsAffected // returns found records count, equals `len(users)`
  result.Error  
  ```

#### 1.3.3.3 内联条件查询

**主键是数字类型**

* 内嵌条件：**where 默认主键等于或者 IN**

  ```go
  db.First(&user, 10)
  // SELECT * FROM users WHERE id = 10;
  
  db.First(&user, "10")
  // SELECT * FROM users WHERE id = 10;
  
  db.Find(&users, []int{1,2,3})
  // SELECT * FROM users WHERE id IN (1,2,3);
  
  ```

**主键是字符串型**

* 内嵌条件：**where 等于的条件**

  ```go
  db.First(&user, "id = ?", "1b74413f-f3b8-409f-ac47-e8c062e3472a")
  // SELECT * FROM users WHERE id = "1b74413f-f3b8-409f-ac47-e8c062e3472a";
  ```

**模型带有字段（等同于条件）**

* 模型嵌入条件：**where 的条件在模型中**

  ```go
  var user = User{ID: 10}
  db.First(&user)
  // SELECT * FROM users WHERE id = 10;
  
  var result User
  db.Model(User{ID: 10}).First(&result)
  // SELECT * FROM users WHERE id = 10;
  ```

**特定字段类型，产生 AND 条件查询**

* 如 `gorm.DeletedAt`

  ```go
  type User struct {
    ID           string `gorm:"primarykey;size:16"`
    Name         string `gorm:"size:24"`
    DeletedAt    gorm.DeletedAt `gorm:"index"`
  }
  
  var user = User{ID: 15}
  db.First(&user)
  //  SELECT * FROM `users` WHERE `users`.`id` = '15' AND `users`.`deleted_at` IS NULL ORDER BY `users`.`id` LIMIT 1
  ```

### 1.3.4 条件查询

#### 1.3.4.1 String 条件

* 各种条件查询的使用

  ```go
  // Get first matched record
  db.Where("name = ?", "jinzhu").First(&user)
  // SELECT * FROM users WHERE name = 'jinzhu' ORDER BY id LIMIT 1;
  
  // Get all matched records
  db.Where("name <> ?", "jinzhu").Find(&users)
  // SELECT * FROM users WHERE name <> 'jinzhu';
  
  // IN
  db.Where("name IN ?", []string{"jinzhu", "jinzhu 2"}).Find(&users)
  // SELECT * FROM users WHERE name IN ('jinzhu','jinzhu 2');
  
  // LIKE
  db.Where("name LIKE ?", "%jin%").Find(&users)
  // SELECT * FROM users WHERE name LIKE '%jin%';
  
  // AND
  db.Where("name = ? AND age >= ?", "jinzhu", "22").Find(&users)
  // SELECT * FROM users WHERE name = 'jinzhu' AND age >= 22;
  
  // Time
  db.Where("updated_at > ?", lastWeek).Find(&users)
  // SELECT * FROM users WHERE updated_at > '2000-01-01 00:00:00';
  
  // BETWEEN
  db.Where("created_at BETWEEN ? AND ?", lastWeek, today).Find(&users)
  // SELECT * FROM users WHERE created_at BETWEEN '2000-01-01 00:00:00' AND '2000-01-08 00:00:00';
  ```

* 如果条件中设置的**主键**，会用 **AND 进行合并查询**

  ```go
  var user = User{ID: 10}
  db.Where("id = ?", 20).First(&user)
  // SELECT * FROM users WHERE id = 10 and id = 20 ORDER BY id ASC LIMIT 1
  ```

#### 1.3.4.2 Struct & Map 条件

* 条件中使用 **struct，map 或者列表**，但是多个**条件会用 AND 来并列**

  ```go
  // Struct
  db.Where(&User{Name: "jinzhu", Age: 20}).First(&user)
  // SELECT * FROM users WHERE name = "jinzhu" AND age = 20 ORDER BY id LIMIT 1;
  
  // Map
  db.Where(map[string]interface{}{"name": "jinzhu", "age": 20}).Find(&users)
  // SELECT * FROM users WHERE name = "jinzhu" AND age = 20;
  
  // Slice of primary keys
  db.Where([]int64{20, 21, 22}).Find(&users)
  // SELECT * FROM users WHERE id IN (20, 21, 22);
  ```

* **结构体**查询**只查询 non-zero 字段**，`0，'', false` 这些不会被放入条件中

* 想要 **non-zero 字段**查询，就需要使用 **map 条件查询**

* 使用结构体查询，**额外的查询条件（string）**，不赋值就是用默认 **non-zero**

  ```go
  db.Where(&User{Name: "jinzhu"}, "name", "Age").Find(&users)
  // SELECT * FROM users WHERE name = "jinzhu" AND age = 0;
  
  db.Where(&User{Name: "jinzhu"}, "Age").Find(&users)
  // SELECT * FROM users WHERE age = 0;
  ```

#### 1.3.4.3 内联条件

* 在 Find 和 First 中**内嵌的条件**

  * 同样可以使用 map 作为条件，同时**第一个是条件字符串**，后面的是**占位符对应的值**

  ```go
  // Get by primary key if it were a non-integer type
  db.First(&user, "id = ?", "string_primary_key")
  // SELECT * FROM users WHERE id = 'string_primary_key';
  
  // Plain SQL
  db.Find(&user, "name = ?", "jinzhu")
  // SELECT * FROM users WHERE name = "jinzhu";
  
  db.Find(&users, "name <> ? AND age > ?", "jinzhu", 20)
  // SELECT * FROM users WHERE name <> "jinzhu" AND age > 20;
  
  // Struct
  db.Find(&users, User{Age: 20})
  // SELECT * FROM users WHERE age = 20;
  
  // Map
  db.Find(&users, map[string]interface{}{"age": 20})
  // SELECT * FROM users WHERE age = 20;
  ```

#### 1.3.4.4 Not 条件

* 与 where 相似

  ```go
  db.Not("name = ?", "jinzhu").First(&user)
  // SELECT * FROM users WHERE NOT name = "jinzhu" ORDER BY id LIMIT 1;
  
  // Not In
  db.Not(map[string]interface{}{"name": []string{"jinzhu", "jinzhu 2"}}).Find(&users)
  // SELECT * FROM users WHERE name NOT IN ("jinzhu", "jinzhu 2");
  
  // Struct
  db.Not(User{Name: "jinzhu", Age: 18}).First(&user)
  // SELECT * FROM users WHERE name <> "jinzhu" AND age <> 18 ORDER BY id LIMIT 1;
  
  // Not In slice of primary keys
  db.Not([]int64{1,2,3}).First(&user)
  // SELECT * FROM users WHERE id NOT IN (1,2,3) ORDER BY id LIMIT 1;
  ```

#### 1.3.4.5 Or 条件

* 要接在 Where 后面

  ```go
  db.Where("role = ?", "admin").Or("role = ?", "super_admin").Find(&users)
  // SELECT * FROM users WHERE role = 'admin' OR role = 'super_admin';
  
  // Struct
  db.Where("name = 'jinzhu'").Or(User{Name: "jinzhu 2", Age: 18}).Find(&users)
  // SELECT * FROM users WHERE name = 'jinzhu' OR (name = 'jinzhu 2' AND age = 18);
  
  // Map
  db.Where("name = 'jinzhu'").Or(map[string]interface{}{"name": "jinzhu 2", "age": 18}).Find(&users)
  // SELECT * FROM users WHERE name = 'jinzhu' OR (name = 'jinzhu 2' AND age = 18);
  ```

### 1.3.5 常用操作

#### 1.3.5.1 选择特定字段（select）

* **Select 语句**，同时可以使用 **SQL 的函数**

  ```go
  db.Select("name", "age").Find(&users)
  // SELECT name, age FROM users;
  
  db.Select([]string{"name", "age"}).Find(&users)
  // SELECT name, age FROM users;
  
  db.Table("users").Select("COALESCE(age,?)", 42).Rows()
  // SELECT COALESCE(age,'42') FROM users;
  ```

#### 1.3.5.2 排序（Order by）

* string 表示，或者多个，或者按照字段 id 来

  ```go
  db.Order("age desc, name").Find(&users)
  // SELECT * FROM users ORDER BY age desc, name;
  
  // Multiple orders
  db.Order("age desc").Order("name").Find(&users)
  // SELECT * FROM users ORDER BY age desc, name;
  
  db.Clauses(clause.OrderBy{
    Expression: clause.Expr{SQL: "FIELD(id,?)", Vars: []interface{}{[]int{1, 2, 3}}, WithoutParentheses: true},
  }).Find(&User{})
  // SELECT * FROM users ORDER BY FIELD(id,1,2,3)
  ```

  * ORDER BY FIELD(id, 1, 2, 3) 是用于指定按照特定顺序对结果进行排序。在这个例子中，id 是用于排序的列名

#### 1.3.5.3 Limit & Offset

* `Offset`是一个用于分页查询的参数，在查询结果中指定要跳过的记录数量，以便从指定位置开始返回记录。通过结合使用`Limit`和`Offset`，可以实现有效的分页功能，以便在大型数据集中按需获取数据。

  ```go
  db.Limit(3).Find(&users)
  // SELECT * FROM users LIMIT 3;
  
  // Cancel limit condition with -1
  db.Limit(10).Find(&users1).Limit(-1).Find(&users2)
  // SELECT * FROM users LIMIT 10; (users1)
  // SELECT * FROM users; (users2)
  
  db.Offset(3).Find(&users)
  // SELECT * FROM users OFFSET 3;
  
  db.Limit(10).Offset(5).Find(&users)
  // SELECT * FROM users OFFSET 5 LIMIT 10;
  
  // Cancel offset condition with -1
  db.Offset(10).Find(&users1).Offset(-1).Find(&users2)
  // SELECT * FROM users OFFSET 10; (users1)
  // SELECT * FROM users; (users2)
  ```

  * `Offset(-1), Limit(-1)` 为取消
  * `Offset` 不能超过查询出来的所有记录的数量

#### 1.3.5.4 Group By & Having

* `Group By` 和 `Having` 相当于 `GORM` 自己添加这个关键字，然后附带上函数中的参数作为条件

  ```go
  type result struct {
    Date  time.Time
    Total int
  }
  
  db.Model(&User{}).Select("name, sum(age) as total").Where("name LIKE ?", "group%").Group("name").First(&result)
  // SELECT name, sum(age) as total FROM `users` WHERE name LIKE "group%" GROUP BY `name` LIMIT 1
  
  
  db.Model(&User{}).Select("name, sum(age) as total").Group("name").Having("name = ?", "group").Find(&result)
  // SELECT name, sum(age) as total FROM `users` GROUP BY `name` HAVING name = "group"
  
  rows, err := db.Table("orders").Select("date(created_at) as date, sum(amount) as total").Group("date(created_at)").Rows()
  defer rows.Close()
  for rows.Next() {
    ...
  }
  
  rows, err := db.Table("orders").Select("date(created_at) as date, sum(amount) as total").Group("date(created_at)").Having("sum(amount) > ?", 100).Rows()
  defer rows.Close()
  for rows.Next() {
    ...
  }
  
  type Result struct {
    Date  time.Time
    Total int64
  }
  db.Table("orders").Select("date(created_at) as date, sum(amount) as total").Group("date(created_at)").Having("sum(amount) > ?", 100).Scan(&results)
  ```

* 注意点：

  * `rows` 需要 `close`，不然占用数据库连接池的连接
  * `rows` 必须先 `Next`，移动游标才行

#### 1.3.5.5 Distinct

* 结果去重

  ```go
  db.Distinct("name", "age").Order("name, age desc").Find(&results)
  ```

`Distinct`也可以与`Pluck`和`Count`一起使用

#### 1.3.5.6 Joins

**Joins**

* 多个 `JOIN` 需要在条件 `string` 中的开头加入 `JOIN`

  ```go
  type result struct {
    Name  string
    Email string
  }
  
  db.Model(&User{}).Select("users.name, emails.email").Joins("left join emails on emails.user_id = users.id").Scan(&result{})
  // SELECT users.name, emails.email FROM `users` left join emails on emails.user_id = users.id
  
  rows, err := db.Table("users").Select("users.name, emails.email").Joins("left join emails on emails.user_id = users.id").Rows()
  for rows.Next() {
    ...
  }
  
  db.Table("users").Select("users.name, emails.email").Joins("left join emails on emails.user_id = users.id").Scan(&results)
  
  // multiple joins with parameter
  db.Joins("JOIN emails ON emails.user_id = users.id AND emails.email = ?", "jinzhu@example.org").Joins("JOIN credit_cards ON credit_cards.user_id = users.id").Where("credit_cards.number = ?", "411111111111").Find(&user)
  ```

**Joins 预加载**

* Joins：一次性加载关联数据

  ```go
  db.Joins("Company").Find(&users)
  // SELECT `users`.`id`,`users`.`name`,`users`.`age`,`Company`.`id` AS `Company__id`,`Company`.`name` AS `Company__name` FROM `users` LEFT JOIN `companies` AS `Company` ON `users`.`company_id` = `Company`.`id`;
  
  // inner join
  db.InnerJoins("Company").Find(&users)
  // SELECT `users`.`id`,`users`.`name`,`users`.`age`,`Company`.`id` AS `Company__id`,`Company`.`name` AS `Company__name` FROM `users` INNER JOIN `companies` AS `Company` ON `users`.`company_id` = `Company`.`id`;
  ```

* 附带条件

  ```go
  db.Joins("Company", db.Where(&Company{Alive: true})).Find(&users)
  // SELECT `users`.`id`,`users`.`name`,`users`.`age`,`Company`.`id` AS `Company__id`,`Company`.`name` AS `Company__name` FROM `users` LEFT JOIN `companies` AS `Company` ON `users`.`company_id` = `Company`.`id` AND `Company`.`alive` = true;
  ```

**Joins 一个衍生表**

```go
type User struct {
    Id  int
    Age int
}

type Order struct {
    UserId     int
    FinishedAt *time.Time
}

query := db.Table("order").Select("MAX(order.finished_at) as latest").Joins("left join user user on order.user_id = user.id").Where("user.age > ?", 18).Group("order.user_id")
db.Model(&Order{}).Joins("join (?) q on order.finished_at = q.latest", query).Scan(&results)
// SELECT `order`.`user_id`,`order`.`finished_at` FROM `order` join (SELECT MAX(order.finished_at) as latest FROM `order` left join user user on order.user_id = user.id WHERE user.age > 18 GROUP BY `order`.`user_id`) q on order.finished_at = q.latest
```

#### 1.3.5.7 Scan

* Scan 用来将结果存入结构体中，需要传入**模型的结构体指针**

  ```go
  type Result struct {
    Name string
    Age  int
  }
  
  var result Result
  db.Table("users").Select("name", "age").Where("name = ?", "Antonio").Scan(&result)
  
  // Raw SQL
  db.Raw("SELECT name, age FROM users WHERE name = ?", "Antonio").Scan(&result)
  ```

  * `Raw` 等于 `sql` 的 `Query`，直接执行查询语句

### 1.3.6 高级查询

#### 1.3.6.1 智能选取字段

**通过创建带有特定字段的结构体，来限制 select 查询的字段**

```go
type User struct {
  ID     uint
  Name   string
  Age    int
  Gender string
  // 假设后面还有几百个字段...
}

type APIUser struct {
  ID   uint
  Name string
}

// 查询时会自动选择 `id`, `name` 字段
db.Model(&User{}).Limit(10).Find(&APIUser{})
// SELECT `id`, `name` FROM `users` LIMIT 10
```

* 失效情况：在 `gorm` 配置成 **QueryFields** 的时候，会查询**全部字段**，两种设置

  * 全局设置

    ```go
    db, err := gorm.Open(sqlite.Open("gorm.db"), &gorm.Config{
      QueryFields: true,
    })
    ```

  * 单个 `session` 设置

    ```go
    db.Session(&gorm.Session{QueryFields: true}).Find(&user)
    ```

#### 1.3.6.2 Locking 多种锁（For Update）

* 使用 Clauses（子句）来设置：相当于在 sql 语句中加不同的锁，通过 `clause.Locking`

  ```go
  db.Clauses(clause.Locking{Strength: "UPDATE"}).Find(&users)
  // SELECT * FROM `users` FOR UPDATE
  
  db.Clauses(clause.Locking{
    Strength: "SHARE",
    Table: clause.Table{Name: clause.CurrentTable},
  }).Find(&users)
  // SELECT * FROM `users` FOR SHARE OF `users`
  
  db.Clauses(clause.Locking{
    Strength: "UPDATE",
    Options: "NOWAIT",
  }).Find(&users)
  // SELECT * FROM `users` FOR UPDATE NOWAIT
  ```

  * `Locking` 结构体用来设置子查询的锁设置

#### 1.3.6.3 子查询

**支持子查询嵌套，通过占位符来捕获 *gorm.DB 的子查询**

* **嵌套子查询**

  ```go
  db.Where("amount > (?)", db.Table("orders").Select("AVG(amount)")).Find(&orders)
  // SELECT * FROM "orders" WHERE amount > (SELECT AVG(amount) FROM "orders");
  
  subQuery := db.Select("AVG(age)").Where("name LIKE ?", "name%").Table("users")
  db.Select("AVG(age) as avgage").Group("name").Having("AVG(age) > (?)", subQuery).Find(&results)
  // SELECT AVG(age) as avgage FROM `users` GROUP BY `name` HAVING AVG(age) > (SELECT AVG(age) FROM `users` WHERE name LIKE "name%")	
  ```

* **`From` 子查询**：`Table` 方法支持 `FROM` 子查询，通过占位符来获取

  ```go
  db.Table("(?) as u", db.Model(&User{}).Select("name", "age")).Where("age = ?", 18).Find(&User{})
  // SELECT * FROM (SELECT `name`,`age` FROM `users`) as u WHERE `age` = 18
  
  subQuery1 := db.Model(&User{}).Select("name")
  subQuery2 := db.Model(&Pet{}).Select("name")
  db.Table("(?) as u, (?) as p", subQuery1, subQuery2).Find(&User{})
  // SELECT * FROM (SELECT `name` FROM `users`) as u, (SELECT `name` FROM `pets`) as p
  ```

#### 1.3.6.4 Group 条件

* 将 Where 的条件进行分组，相当于**加括号**

```go
db.Where(
    db.Where("pizza = ?", "pepperoni").Where(db.Where("size = ?", "small").Or("size = ?", "medium")),
).Or(
    db.Where("pizza = ?", "hawaiian").Where("size = ?", "xlarge"),
).Find(&Pizza{}).Statement

// SELECT * FROM `pizzas` WHERE (pizza = "pepperoni" AND (size = "small" OR size = "medium")) OR (pizza = "hawaiian" AND size = "xlarge")
```

#### 1.3.6.5 带多个列的 IN

* 等于占位符所替代的是一个 `[][]interface{}{初始化}`

#### 1.3.6.6 命名参数

* 支持 `sql,NamedArg` 和 `map[string]interface{}{}` 形式的命名参数

  ```go
  db.Where("name1 = @name OR name2 = @name", sql.Named("name", "jinzhu")).Find(&user)
  // SELECT * FROM `users` WHERE name1 = "jinzhu" OR name2 = "jinzhu"
  
  db.Where("name1 = @name OR name2 = @name", map[string]interface{}{"name": "jinzhu"}).First(&user)
  // SELECT * FROM `users` WHERE name1 = "jinzhu" OR name2 = "jinzhu" ORDER BY `users`.`id` LIMIT 1
  ```

#### 1.3.6.7 Find 至 map

* 不用定义结构体，将结果映射到 `map` 中
  * 单条记录：`map[string]interface{}`
  * 多条记录：`[]map[string]interface{}`

#### 1.3.6.8 FirstOrInit 未找到根据条件初始化

* 通过结构体或者 `map` 去设置条件

  ```go
  // 未找到 user，则根据给定的条件初始化一条记录
  db.FirstOrInit(&user, User{Name: "non_existing"})
  // user -> User{Name: "non_existing"}
  
  // 找到了 `name` = `jinzhu` 的 user
  db.Where(User{Name: "jinzhu"}).FirstOrInit(&user)
  // user -> User{ID: 111, Name: "Jinzhu", Age: 18}
  
  // 找到了 `name` = `jinzhu` 的 user
  db.FirstOrInit(&user, map[string]interface{}{"name": "jinzhu"})
  // user -> User{ID: 111, Name: "Jinzhu", Age: 18}
  ```

  * **Attrs** 用更多的属性进行初始化结构体（不会生成查询 SQL）

    ```go
    // 未找到 user，则根据给定的条件以及 Attrs 初始化 user
    db.Where(User{Name: "non_existing"}).Attrs(User{Age: 20}).FirstOrInit(&user)
    // SELECT * FROM USERS WHERE name = 'non_existing' ORDER BY id LIMIT 1;
    // user -> User{Name: "non_existing", Age: 20}
    
    // 未找到 user，则根据给定的条件以及 Attrs 初始化 user
    db.Where(User{Name: "non_existing"}).Attrs("age", 20).FirstOrInit(&user)
    // SELECT * FROM USERS WHERE name = 'non_existing' ORDER BY id LIMIT 1;
    // user -> User{Name: "non_existing", Age: 20}
    
    // 找到了 `name` = `jinzhu` 的 user，则忽略 Attrs
    db.Where(User{Name: "Jinzhu"}).Attrs(User{Age: 20}).FirstOrInit(&user)
    // SELECT * FROM USERS WHERE name = jinzhu' ORDER BY id LIMIT 1;
    // user -> User{ID: 111, Name: "Jinzhu", Age: 18}
    ```

  * `Assign` **不管是否**找到记录，都会将属性赋给 `struct`

    ```go
    // 未找到 user，根据条件和 Assign 属性初始化 struct
    db.Where(User{Name: "non_existing"}).Assign(User{Age: 20}).FirstOrInit(&user)
    // user -> User{Name: "non_existing", Age: 20}
    
    // 找到 `name` = `jinzhu` 的记录，依然会更新 Assign 相关的属性
    db.Where(User{Name: "Jinzhu"}).Assign(User{Age: 20}).FirstOrInit(&user)
    // SELECT * FROM USERS WHERE name = jinzhu' ORDER BY id LIMIT 1;
    // user -> User{ID: 111, Name: "Jinzhu", Age: 20}
    ```

* 实际就是没找到的话，创建一个不存在的，同时不会对数据库进行操作（不存在的不会创建到数据库中）

#### 1.3.6.9 FirstOrCreate

* 获取匹配的第一条记录或者根据给定条件创建一条新纪录，`RowsAffected` 返回创建、更新的记录数
  * **Attrs**：没找到记录则创建但不会生成 SQL 语句，找到了则忽略
  * **Assign**：不管找没找到，都将属性赋值给 `struct` 同时**写回**数据库

#### 1.3.6.10 优化器、索引提示

* 优化器（gorm.io/hints）：优化器提示用来控制查询优化器选择某个查询执行计划

  ```go
  import "gorm.io/hints"
  
  db.Clauses(hints.New("MAX_EXECUTION_TIME(10000)")).Find(&User{})
  // SELECT * /*+ MAX_EXECUTION_TIME(10000) */ FROM `users`
  ```

* 索引提示：传递索引提示到数据库，防止查询计划器出现混乱（允许你在查询中向数据库传递一些提示信息，以帮助查询优化器生成更好的查询计划）

  ```go
  import "gorm.io/hints"
  
  db.Clauses(hints.UseIndex("idx_user_name")).Find(&User{})
  // SELECT * FROM `users` USE INDEX (`idx_user_name`)
  
  db.Clauses(hints.ForceIndex("idx_user_name", "idx_user_id").ForJoin()).Find(&User{})
  // SELECT * FROM `users` FORCE INDEX FOR JOIN (`idx_user_name`,`idx_user_id`)"
  ```

#### 1.3.6.11 迭代（火山模型）

* 返回多行数据，使用 `Rows.Next()`，和 `sql` 库相同

#### 1.3.6.12 FindInBatches 批量查询并处理记录

* 在 FindInBatches 中设置批量以及对应**处理函数**，来对查询的**记录进行处理**

  ```go
  // 每次批量处理 100 条
  result := db.Where("processed = ?", false).FindInBatches(&results, 100, func(tx *gorm.DB, batch int) error {
    for _, result := range results {
      // 批量处理找到的记录
    }
  
    tx.Save(&results)
  
    tx.RowsAffected // 本次批量操作影响的记录数
  
    batch // Batch 1, 2, 3
  
    // 如果返回错误会终止后续批量操作
    return nil
  })
  
  result.Error // returned error
  result.RowsAffected // 整个批量操作影响的记录数
  ```

#### 1.3.6.13 查询钩子

* **AfterFind**：同样是结构体的方法

#### 1.3.6.14 Pluck 查询单个列

* 使用 `Pluck` 方法，传入一个存储单列结果的 `slice`

#### 1.3.6.15 Scope 指定常用查询

* 提前定义好查询语句（封装），然后将函数作为 `Scopes` 的参数，让 `Scopes`进行调用，等于**语句复用**

  ```go
  func AmountGreaterThan1000(db *gorm.DB) *gorm.DB {
    return db.Where("amount > ?", 1000)
  }
  
  func PaidWithCreditCard(db *gorm.DB) *gorm.DB {
    return db.Where("pay_mode_sign = ?", "C")
  }
  
  func PaidWithCod(db *gorm.DB) *gorm.DB {
    return db.Where("pay_mode_sign = ?", "C")
  }
  
  func OrderStatus(status []string) func (db *gorm.DB) *gorm.DB {
    return func (db *gorm.DB) *gorm.DB {
      return db.Where("status IN (?)", status)
    }
  }
  
  db.Scopes(AmountGreaterThan1000, PaidWithCreditCard).Find(&orders)
  // 查找所有金额大于 1000 的信用卡订单
  
  db.Scopes(AmountGreaterThan1000, PaidWithCod).Find(&orders)
  // 查找所有金额大于 1000 的货到付款订单
  
  db.Scopes(AmountGreaterThan1000, OrderStatus([]string{"paid", "shipped"})).Find(&orders)
  // 查找所有金额大于 1000 且已付款或已发货的订单
  ```

#### 1.3.6.16 Count

* 用于获得获取匹配的记录数

### 1.3.4 更新

#### 1.3.4.1 保存所有字段（Save）

* `Save` 方法回保存**所有字段**，即使字段是零值

#### 1.3.4.2 更新单个列

* 指定 `Model`，然后直接使用 `update` 进行 `kv` 形式的更新

  ```go
  // Update with conditions
  db.Model(&User{}).Where("active = ?", true).Update("name", "hello")
  // UPDATE users SET name='hello', updated_at='2013-11-17 21:34:10' WHERE active=true;
  
  // User's ID is `111`:
  db.Model(&user).Update("name", "hello")
  // UPDATE users SET name='hello', updated_at='2013-11-17 21:34:10' WHERE id=111;
  
  // Update with conditions and model value
  db.Model(&user).Where("active = ?", true).Update("name", "hello")
  // UPDATE users SET name='hello', updated_at='2013-11-17 21:34:10' WHERE id=111 AND active=true;
  ```

  * 多个条件，Model 可以使用附加条件（在结构体中）

#### 1.3.4.3 更新多个列

* 使用 `Updates`，传入模型的结构体（为更新的字段显示设置值）或者使用 `map[string]interface{}`

* 更新**选定字段**和**更新多列相同**

#### 1.3.4.4 批量更新

* 带有**非主键条件（非唯一条件）**，则批量更新

#### 1.3.4.5 阻止全局更新（安全措施）

* 不设定主键条件，或者产生更新所有记录的语句，都会产生 `ErrMissingWhereClause` 的错误
* 想要进行全局更新，需要通过对 Session 设置 `AllowGlobalUpdate` 来进行操作

```go
db.Model(&User{}).Update("name", "jinzhu").Error // gorm.ErrMissingWhereClause

db.Model(&User{}).Where("1 = 1").Update("name", "jinzhu")
// UPDATE users SET `name` = "jinzhu" WHERE 1=1

db.Exec("UPDATE users SET name = ?", "jinzhu")
// UPDATE users SET name = "jinzhu"

db.Session(&gorm.Session{AllowGlobalUpdate: true}).Model(&User{}).Update("name", "jinzhu")
// UPDATE users SET `name` = "jinzhu"
```

#### 1.3.4.6 更新的记录数

* 通过 `Result.RowsAffected` 查看

#### 1.3.4.7 使用 SQL 表达式更新

* 通过表达式更新一个列（gorm.Expr），相当于将表达式嵌入到 `SQL` 语句中

  ```go
  // product's ID is `3`
  db.Model(&product).Update("price", gorm.Expr("price * ? + ?", 2, 100))
  // UPDATE "products" SET "price" = price * 2 + 100, "updated_at" = '2013-11-17 21:34:10' WHERE "id" = 3;
  
  db.Model(&product).Updates(map[string]interface{}{"price": gorm.Expr("price * ? + ?", 2, 100)})
  // UPDATE "products" SET "price" = price * 2 + 100, "updated_at" = '2013-11-17 21:34:10' WHERE "id" = 3;
  
  db.Model(&product).UpdateColumn("quantity", gorm.Expr("quantity - ?", 1))
  // UPDATE "products" SET "quantity" = quantity - 1 WHERE "id" = 3;
  
  db.Model(&product).Where("quantity > 1").UpdateColumn("quantity", gorm.Expr("quantity - ?", 1))
  // UPDATE "products" SET "quantity" = quantity - 1 WHERE "id" = 3 AND quantity > 1;
  ```

* 通过表达式和上下文值来创建（对特定的字段调用表达式：clause.Expr），字段结构体自带的 GormValue 方法，来获取表达式

  ```go
  // Create from customized data type
  type Location struct {
      X, Y int
  }
  
  func (loc Location) GormValue(ctx context.Context, db *gorm.DB) clause.Expr {
    return clause.Expr{
      SQL:  "ST_PointFromText(?)",
      Vars: []interface{}{fmt.Sprintf("POINT(%d %d)", loc.X, loc.Y)},
    }
  }
  
  db.Model(&User{ID: 1}).Updates(User{
    Name:  "jinzhu",
    Location: Location{X: 100, Y: 100},
  })
  // UPDATE `user_with_points` SET `name`="jinzhu",`location`=ST_PointFromText("POINT(100 100)") WHERE `id` = 1
  ```

* 综合就是 `Updates` 会先查找更新的结构体字段是否有 `GormValue` 方法，有的话，会将内容嵌入到 `SQL` 语句中

#### 1.3.4.8 子查询嵌套更新

* 等同于 `updates` 的赋值，是通过子查询获取的内容（即子查询会返回一个结果）

  ```go
  db.Model(&user).Update("company_name", db.Model(&Company{}).Select("name").Where("companies.id = users.company_id"))
  // UPDATE "users" SET "company_name" = (SELECT name FROM companies WHERE companies.id = users.company_id);
  
  db.Table("users as u").Where("name = ?", "jinzhu").Update("company_name", db.Table("companies as c").Select("name").Where("c.id = u.company_id"))
  
  db.Table("users as u").Where("name = ?", "jinzhu").Updates(map[string]interface{}{"company_name": db.Table("companies as c").Select("name").Where("c.id = u.company_id")})
  ```

#### 1.3.4.9 不使用 Hook 和时间追踪

* 使用 `UpdateColumn` 和 `UpdateColumns` 方法，与 `Update` 和 `Updates` 类似

#### 1.3.4.10 返回修改行的数据

* 使用 `Clauses` 子查询，使用 `clause.Returning{}` 参数 

  ```go
  // return all columns
  var users []User
  db.Model(&users).Clauses(clause.Returning{}).Where("role = ?", "admin").Update("salary", gorm.Expr("salary * ?", 2))
  // UPDATE `users` SET `salary`=salary * 2,`updated_at`="2021-10-28 17:37:23.19" WHERE role = "admin" RETURNING *
  // users => []User{{ID: 1, Name: "jinzhu", Role: "admin", Salary: 100}, {ID: 2, Name: "jinzhu.2", Role: "admin", Salary: 1000}}
  
  // return specified columns
  db.Model(&users).Clauses(clause.Returning{Columns: []clause.Column{{Name: "name"}, {Name: "salary"}}}).Where("role = ?", "admin").Update("salary", gorm.Expr("salary * ?", 2))
  // UPDATE `users` SET `salary`=salary * 2,`updated_at`="2021-10-28 17:37:23.19" WHERE role = "admin" RETURNING `name`, `salary`
  // users => []User{{ID: 0, Name: "jinzhu", Role: "", Salary: 100}, {ID: 0, Name: "jinzhu.2", Role: "", Salary: 1000}}
  ```

  * `Returning` 不设置返回的字段，会**全部返回**。设置了，就**返回指定的**，其余的为 ""

#### 1.3.4.11 检查字段是否变更

* 使用 `Hook`，在 `Hook` 中使用 `tx.Statement.Changed` **检查字段**是否变更

  ```go
  func (u *User) BeforeUpdate(tx *gorm.DB) (err error) {
    // if Role changed
      if tx.Statement.Changed("Role") {
      return errors.New("role not allowed to change")
      }
  
    if tx.Statement.Changed("Name", "Admin") { // if Name or Role changed
      tx.Statement.SetColumn("Age", 18)
    }
  
    // if any fields changed
      if tx.Statement.Changed() {
          tx.Statement.SetColumn("RefreshedAt", time.Now())
      }
      return nil
  }
  
  db.Model(&User{ID: 1, Name: "jinzhu"}).Updates(map[string]interface{"name": "jinzhu2"})
  // Changed("Name") => true
  db.Model(&User{ID: 1, Name: "jinzhu"}).Updates(map[string]interface{"name": "jinzhu"})
  // Changed("Name") => false, `Name` not changed
  db.Model(&User{ID: 1, Name: "jinzhu"}).Select("Admin").Updates(map[string]interface{
    "name": "jinzhu2", "admin": false,
  })
  // Changed("Name") => false, `Name` not selected to update
  
  db.Model(&User{ID: 1, Name: "jinzhu"}).Updates(User{Name: "jinzhu2"})
  // Changed("Name") => true
  db.Model(&User{ID: 1, Name: "jinzhu"}).Updates(User{Name: "jinzhu"})
  // Changed("Name") => false, `Name` not changed
  db.Model(&User{ID: 1, Name: "jinzhu"}).Select("Admin").Updates(User{Name: "jinzhu2"})
  // Changed("Name") => false, `Name` not selected to update
  ```

#### 1.3.4.12 在 Update 时修改值

* 同样使用 `Hook` 在更新前判断

  ```go
  func (user *User) BeforeSave(tx *gorm.DB) (err error) {
    if pw, err := bcrypt.GenerateFromPassword(user.Password, 0); err == nil {
      tx.Statement.SetColumn("EncryptedPassword", pw)
    }
  
    if tx.Statement.Changed("Code") {
      user.Age += 20
      tx.Statement.SetColumn("Age", user.Age)
    }
  }
  
  db.Model(&user).Update("Name", "jinzhu")
  ```

### 1.3.5 删除

#### 1.3.5.1 删除一条记录

* 删除对象需要指定主键，否则会**触发批量删除**

#### 1.3.5.2 根据主键删除

* 主键，内联条件

#### 1.3.5.3 批量删除

* **不指定主属性**，使用模糊查询之类的，会进行批量删除，Delete 中就可以附带**模糊查询**的语句

* 带有主键的模型切片，可以提高删除的效率

#### 1.3.5.4 阻止全局删除

* 与 `Updates` 相同，会去阻止全局删除

#### 1.3.5.5 返回删除行的数据

* 同样与 `Updates` 相同，使用 `clause.Returning`

#### 1.3.5.6 软删除

**软删除，标记删除，但不会物理删除**

* `gorm.DeletedAt`字段，自动获得软删除能力
* `gorm` 会标记为当前时间，让查询方法无法查询到

**查找被删除的记录**

* 使用 `Unscoped()` 获取 `tx`，来进行查找

  ```go
  db.Unscoped().Where("age = 20").Find(&users)
  // SELECT * FROM users WHERE age = 20;
  ```

**永久删除**

* 同样使用 `Unscoped()` 获取 `tx`，然后进行删除

#### 1.3.5.7 删除标志

**gorm.Model 使用 *time.Time 作为 DeletedAt 的字段类型，通过软删除插件 gorm.io/plugin/soft_delete 可以指定其他数据类型**

```go
import "gorm.io/plugin/soft_delete"

type User struct {
  ID        uint
  Name      string                `gorm:"uniqueIndex:udx_name"`
  DeletedAt soft_delete.DeletedAt `gorm:"uniqueIndex:udx_name"`
}
```

**Unix 时间戳**

* 使用 `milli` 或者 `nano` 作为值，通过 `tag` 作为标记

  ```go
  type User struct {
    ID    uint
    Name  string
    DeletedAt soft_delete.DeletedAt `gorm:"softDelete:milli"`
    // DeletedAt soft_delete.DeletedAt `gorm:"softDelete:nano"`
  }
  
  // 查询
  SELECT * FROM users WHERE deleted_at = 0;
  
  // 软删除
  UPDATE users SET deleted_at = /* current unix milli second or nano second */ WHERE ID = 1;
  ```

* 使用 `1/0` 作为删除标志

  ```go
  import "gorm.io/plugin/soft_delete"
  
  type User struct {
    ID    uint
    Name  string
    IsDel soft_delete.DeletedAt `gorm:"softDelete:flag"`
  }
  
  // 查询
  SELECT * FROM users WHERE is_del = 0;
  
  // 软删除
  UPDATE users SET is_del = 1 WHERE ID = 1;
  ```

* 混合模式

  ```go
  type User struct {
    ID        uint
    Name      string
    DeletedAt time.Time
    IsDel     soft_delete.DeletedAt `gorm:"softDelete:flag,DeletedAtField:DeletedAt"` // use `1` `0`
    // IsDel     soft_delete.DeletedAt `gorm:"softDelete:,DeletedAtField:DeletedAt"` // use `unix second`
    // IsDel     soft_delete.DeletedAt `gorm:"softDelete:nano,DeletedAtField:DeletedAt"` // use `unix nano second`
  }
  
  // 查询
  SELECT * FROM users WHERE is_del = 0;
  
  // 软删除
  UPDATE users SET is_del = 1, deleted_at = /* current unix second */ WHERE ID = 1;
  ```

### 1.3.6 原生 SQL 和 SQL 生成器

* **原生 SQL**

  * 使用 `Raw` 执行原生 `SQL` 语句，`Scan` 获取结果（不需要传入模型结构体，而是对应查询显示结果的结构体即可
  * `Exec`  原生 SQL，可以使用**占位符**来**获取表达式**
  * GORM 允许缓存预编译 SQL 语句来提高性能，查看 [性能](https://gorm.io/zh_CN/docs/performance.html) 获取详情

* **命名参数**（支持 `sql.NamedArg` 和 `map[string]interface{}{}` 或 `struct` 形式的命名参数）

  * `sql.Named`: 占位符名称 + 替换参数的形式
  * `map[string]interface{}`： 同样是 占位符名称 + 替换参数的 `kv` 形式
  * `struct`：字段为占位符的名称，字段对应的值，为替换参数的值（初始化）

  ```go
  db.Where("name1 = @name OR name2 = @name", sql.Named("name", "jinzhu")).Find(&user)
  // SELECT * FROM `users` WHERE name1 = "jinzhu" OR name2 = "jinzhu"
  
  db.Where("name1 = @name OR name2 = @name", map[string]interface{}{"name": "jinzhu2"}).First(&result3)
  // SELECT * FROM `users` WHERE name1 = "jinzhu2" OR name2 = "jinzhu2" ORDER BY `users`.`id` LIMIT 1
  
  // 原生 SQL 及命名参数
  db.Raw("SELECT * FROM users WHERE name1 = @name OR name2 = @name2 OR name3 = @name",
     sql.Named("name", "jinzhu1"), sql.Named("name2", "jinzhu2")).Find(&user)
  // SELECT * FROM users WHERE name1 = "jinzhu1" OR name2 = "jinzhu2" OR name3 = "jinzhu1"
  
  db.Exec("UPDATE users SET name1 = @name, name2 = @name2, name3 = @name",
     sql.Named("name", "jinzhunew"), sql.Named("name2", "jinzhunew2"))
  // UPDATE users SET name1 = "jinzhunew", name2 = "jinzhunew2", name3 = "jinzhunew"
  
  db.Raw("SELECT * FROM users WHERE (name1 = @name AND name3 = @name) AND name2 = @name2",
     map[string]interface{}{"name": "jinzhu", "name2": "jinzhu2"}).Find(&user)
  // SELECT * FROM users WHERE (name1 = "jinzhu" AND name3 = "jinzhu") AND name2 = "jinzhu2"
  
  type NamedArgument struct {
      Name string
      Name2 string
  }
  
  db.Raw("SELECT * FROM users WHERE (name1 = @Name AND name3 = @Name) AND name2 = @Name2",
       NamedArgument{Name: "jinzhu", Name2: "jinzhu2"}).Find(&user)
  // SELECT * FROM users WHERE (name1 = "jinzhu" AND name3 = "jinzhu") AND name2 = "jinzhu2"
  ```

* **DryRun 模式**（不执行，生成 `SQL` 以及参数，用来调试）

  * 同样根据 `gorm.Session` 进行设置参数 `.Statement` 获取

  ```go
  stmt := db.Session(&gorm.Session{DryRun: true}).First(&user, 1).Statement
  stmt.SQL.String() //=> SELECT * FROM `users` WHERE `id` = $1 ORDER BY `id`
  stmt.Vars         //=> []interface{}{1}
  ```

* **ToSQL**（返回生成 `SQL` 但不执行）

  * 相当与是用来查看些的语句产生的 SQL 是什么

    ```go
    sql := db.ToSQL(func(tx *gorm.DB) *gorm.DB {
      return tx.Model(&User{}).Where("id = ?", 100).Limit(10).Order("age desc").Find(&[]User{})
    })
    ```

* **Row 和 Rows**（与 sql 相同）

  * `Row` 直接使用 `Scan`
  * `Rows` 需要先 `Next` 移动游标，然后还需要关闭 `rows.Close()` ，不然数据库连接不会释放

* **将 sql.Rows 扫描到 model 中**

  * s使用 `ScanRows(rows, &user)` 来进行扫描，同时也是需要**移动游标**

* **连接；在一个 tpc DB 连接中运行多条 SQL（不是事务）**

  * 方法中的 `SQL`，会在一个 `Connect` 中全部执行

  ```go
  db.Connection(func(tx *gorm.DB) error {
    tx.Exec("SET my.role = ?", "admin")
  
    tx.First(&User{})
  })
  ```

* **子句（Clause）**

  * GORM 内部使用 SQL builder 生成 SQL。对于每个操作，GORM 都会创建一个 `*gorm.Statement` 对象，所有的 GORM API 都是在为 `statement` 添加、修改 `子句`，最后，GORM 会根据这些子句生成 SQL

  * 当调用 `First` 进行查询时，会在 `Statement` 中添加以下子句，用来构造查询

    ```go
    var limit = 1
    clause.Select{Columns: []clause.Column{{Name: "*"}}}
    clause.From{Tables: []clause.Table{{Name: clause.CurrentTable}}}
    clause.Limit{Limit: &limit}
    clause.OrderBy{Columns: []clause.OrderByColumn{
      {
        Column: clause.Column{
          Table: clause.CurrentTable,
          Name:  clause.PrimaryKey,
        },
      },
    }}
    ```

  * 然后 GORM 在 `Query` callback 中构建最终的查询 SQL

    ```go
    Statement.Build("SELECT", "FROM", "WHERE", "GROUP BY", "ORDER BY", "LIMIT", "FOR")
    ```

  * 生成 SQL

    ```go
    SELECT * FROM `users` ORDER BY `users`.`id` LIMIT 1
    ```

  * 您可以自定义 `子句` 并与 GORM 一起使用，这需要实现 [Interface](https://pkg.go.dev/gorm.io/gorm/clause?tab=doc#Interface) 接口；[示例](https://github.com/go-gorm/gorm/tree/master/clause)

* **子句构造器**（不同数据库，子句可能生成不同的 SQL）

  * 支持 Clause 的原因是，允许数据库驱动程序通过注册 Clause Builder 来取代默认值

    ```go
    db.Offset(10).Limit(5).Find(&users)
    // SQL Server 会生成
    // SELECT * FROM "users" OFFSET 10 ROW FETCH NEXT 5 ROWS ONLY
    // MySQL 会生成
    // SELECT * FROM `users` LIMIT 5 OFFSET 10
    ```

* **子句选项**（添加自定义子句）

  ```go
  db.Clauses(clause.Insert{Modifier: "IGNORE"}).Create(&user)
  // INSERT IGNORE INTO users (name,age...) VALUES ("jinzhu",18...);
  ```

  * clause.Insert 是 GORM 提供的插入语句选项，用于指定插入操作的选项和修饰符。
  * {Modifier: "IGNORE"} 是 clause.Insert 的一个选项，通过设置 Modifier 字段为 "IGNORE"，指示数据库在插入时忽略重复键的错误。

* **StatementModifier**

  * 允许修改语句

    ```go
    import "gorm.io/hints"
    
    db.Clauses(hints.New("hint")).Find(&User{})
    // SELECT * /*+ hint */ FROM `users`
    ```

  * 在 GORM 中，StatementModifier 表示语句修改器，用于向查询中添加特定的 SQL 修饰符或选项。它可以通过 Clauses 方法和相应的 GORM 结构体选项来指定。

  * 常见的示例

    * `clause.Locking`：用于在查询中添加数据库行级别的锁定选项，如 `FOR UPDATE` 或 `LOCK IN SHARE MODE`。
    * `clause.OrderBy`：用于指定查询结果的排序方式。
    * `clause.GroupBy`：用于指定查询结果的分组方式。
    * `clause.Having`：用于指定查询结果的筛选条件。
    * `clause.Limit`：用于限制查询结果的返回数量。
    * `clause.Offset`：用于指定查询结果的起始偏移量。
    * `clause.Select`：用于指定查询结果中要选择的列。

### 1.3.* 总结

* 这些操作都是在 `tx` 的基础上进行的，一个事务逻辑上等同一个数据库连接
* 如果使用列表，那默认就是主键的条件（IN）
* 同一个 `tx`，可以在一个语句中多次查询多次修改条件
* 可以带条件的，都会有**占位符**，多参数输入后续参数是 `Any`
* 占位符不是捕获一个变量，而是可以**捕获各种内容**（多维列表，*gorm.DB 的语句等）
* Model 回同时判断结构体中**初始化的字段**来当作**条件**
* Hook 功能很多，更新前更新后，对数据库操作，对插入数据操作，验证，加密等等

## 1.4 数据库设置

### 1.4.1 指定数据库模型

* `db.Moedel(&User{})`：通过这个指定数据库模型， 返回一个 `tx *DB`（事务）

### 1.4.2 指定表

* `db.Table("users")`：通过这个指定表，同样返回一个事务

### 1.4.3 Belongs To（一对一的关系）

**相当于指定模型外键**

#### 1.4.3.1 Belongs To

`belongs to` 会与另一个模型建立了一对一的连接。 这种模型的每一个实例都“属于”另一个模型的一个实例

```go
// `User` 属于 `Company`，`CompanyID` 是外键
type User struct {
  gorm.Model
  Name      string
  CompanyID int
  Company   Company
}

type Company struct {
  ID   int
  Name string
}
```

* `CompanyID` 被隐含地用来在 `User` 和 `Company` 之间创建一个外键关系， 因此必须包含在 `User` 结构体中才能填充 `Company` 内部结构体

#### 1.4.3.2 重写外键

**使用 Tag 重新定义外键**

* 在一对一的实体的 `Tag` 中标注**外键字段**的名字

  ```go
  type User struct {
    gorm.Model
    Name         string
    CompanyRefer int
    Company      Company `gorm:"foreignKey:CompanyRefer"`
    // 使用 CompanyRefer 作为外键
  }
  
  type Company struct {
    ID   int
    Name string
  }
  ```

#### 1.4.3.3 重写引用 

* `GORM` 会默认把 `Company` 中的主键的属性值保存到 `User` 的外键中，可以通过 `references` 修改

  ```go
  type User struct {
    gorm.Model
    Name      string
    CompanyID string
    Company   Company `gorm:"references:Code"` // 使用 Code 作为引用
  }
  
  type Company struct {
    ID   int
    Code string
    Name string
  }
  ```

  * 将 `Company` 实体中的 `Code` 作为 `User` 外键初始化的值
  * 如果外键名恰好在拥有者类型中存在，GORM 通常会错误的认为它是 `has one` 关系。我们需要在 `belongs to` 关系中指定 `references`

  ```go
  type User struct {
    gorm.Model
    Name      string
    CompanyID string
    Company   Company `gorm:"references:CompanyID"` // 使用 Company.CompanyID 作为引用
  }
  
  type Company struct {
    CompanyID   int
    Code        string
    Name        string
  }
  ```

#### 1.4.3.4 外键约束

* 你可以通过 `constraint` 标签配置 `OnUpdate`、`OnDelete` 实现外键约束，在使用 GORM 进行迁移时它会被创建

  ```go
  type User struct {
    gorm.Model
    Name      string
    CompanyID int
    Company   Company `gorm:"constraint:OnUpdate:CASCADE,OnDelete:SET NULL;"`
  }
  
  type Company struct {
    ID   int
    Name string
  }
  ```

### 1.4.4 Has One（一对一关联）

**每个模型都拥有另一个完整的模型，不是外键关系，而是整体关系**

#### 1.4.4.1 Has One

* **声明**

  ```go
  // User 有一张 CreditCard，UserID 是外键
  type User struct {
    gorm.Model
    CreditCard CreditCard   // 完整的模型
  }
  
  type CreditCard struct {
    gorm.Model
    Number string
    UserID uint   // 会将 CreditCard 的主键保存到 User 中
  }
  ```

* **检索**

  ```go
  // 检索用户列表并预加载信用卡
  func GetAll(db *gorm.DB) ([]User, error) {
      var users []User
      err := db.Model(&User{}).Preload("CreditCard").Find(&users).Error
      return users, err
  }
  ```

#### 1.4.4.2 重写外键

```go
type User struct {
  gorm.Model
  CreditCard CreditCard `gorm:"foreignKey:UserName"` // 使用 UserName 作为外键
}

type CreditCard struct {
  gorm.Model
  Number   string
  UserName string
}
```

#### 1.4.4.3 重写引用

* 在 `Tag` 中标注 `foreignKey`，同时标注 `references`

  ```go
  type User struct {
    gorm.Model
    Name       string     `gorm:"index"`
    CreditCard CreditCard `gorm:"foreignKey:UserName;references:name"`
  }
  
  type CreditCard struct {
    gorm.Model
    Number   string
    UserName string
  }
  ```

#### 1.4.4.4 多态关联

* GORM 为 `has one` 和 `has many` 提供了多态关联支持，它会将拥有者实体的表名、主键值都保存到多态类型的字段中

  ```go
  type Cat struct {
    ID    int
    Name  string
    Toy   Toy `gorm:"polymorphic:Owner;"`
  }
  
  type Dog struct {
    ID   int
    Name string
    Toy  Toy `gorm:"polymorphic:Owner;"`
  }
  
  type Toy struct {
    ID        int
    Name      string
    OwnerID   int
    OwnerType string
  }
  
  db.Create(&Dog{Name: "dog1", Toy: Toy{Name: "toy1"}})
  // INSERT INTO `dogs` (`name`) VALUES ("dog1")
  // INSERT INTO `toys` (`name`,`owner_id`,`owner_type`) VALUES ("toy1","1","dogs")
  ```

* 使用标签 `polymorphicValue` 来更改多态类型的值

  ```go
  type Dog struct {
    ID   int
    Name string
    Toy  Toy `gorm:"polymorphic:Owner;polymorphicValue:master"`
  }
  
  type Toy struct {
    ID        int
    Name      string
    OwnerID   int
    OwnerType string
  }
  
  db.Create(&Dog{Name: "dog1", Toy: Toy{Name: "toy1"}})
  // INSERT INTO `dogs` (`name`) VALUES ("dog1")
  // INSERT INTO `toys` (`name`,`owner_id`,`owner_type`) VALUES ("toy1","1","master")
  ```

#### 1.4.4.5 自引用 Has One

```go
type User struct {
  gorm.Model
  Name      string
  ManagerID *uint
  Manager   *User
}
```

#### 1.4.4.6 外键约束

* 与 `Belongs to` 相同

  ```go
  type User struct {
    gorm.Model
    CreditCard CreditCard `gorm:"constraint:OnUpdate:CASCADE,OnDelete:SET NULL;"`
  }
  
  type CreditCard struct {
    gorm.Model
    Number string
    UserID uint
  }
  ```

### 1.4.5 Has Many（一对多的连接） 

**拥有者可以有零个或者多个关联模型**

#### 1.4.5.1 Has Many

* **声明**（实体列表）

  ```go
  // User 有多张 CreditCard，UserID 是外键
  type User struct {
    gorm.Model
    CreditCards []CreditCard
  }
  
  type CreditCard struct {
    gorm.Model
    Number string
    UserID uint
  }
  ```

* **检索**

  ```go
  // 检索用户列表并预加载信用卡
  func GetAll(db *gorm.DB) ([]User, error) {
      var users []User
      err := db.Model(&User{}).Preload("CreditCards").Find(&users).Error
      return users, err
  }
  ```

#### 1.4.5.2 重写外键

* 用另一个字段作为外键

  ```go
  type User struct {
    gorm.Model
    CreditCards []CreditCard `gorm:"foreignKey:UserRefer"`   // 使用 UserRefer 作为 CreditCard 的外键
  }
  
  type CreditCard struct {
    gorm.Model
    Number    string
    UserRefer uint
  }
  ```

#### 1.4.5.3 重写引用

```go
type User struct {
  gorm.Model
  MemberNumber string
  CreditCards  []CreditCard `gorm:"foreignKey:UserNumber;references:MemberNumber"`
}

type CreditCard struct {
  gorm.Model
  Number     string
  UserNumber string
}
```

#### 1.4.5.4 多态关联

* GORM 为 `has one` 和 `has many` 提供了多态关联支持，它会将拥有者实体的表名、主键都保存到多态类型的字段中

  ```go
  type Dog struct {
    ID   int
    Name string
    Toys []Toy `gorm:"polymorphic:Owner;"`
  }
  
  type Toy struct {
    ID        int
    Name      string
    OwnerID   int    // 主键
    OwnerType string   // 表名
  }
  
  db.Create(&Dog{Name: "dog1", Toys: []Toy{{Name: "toy1"}, {Name: "toy2"}}})
  // INSERT INTO `dogs` (`name`) VALUES ("dog1")
  // INSERT INTO `toys` (`name`,`owner_id`,`owner_type`) VALUES ("toy1","1","dogs"), ("toy2","1","dogs")
  ```

* 使用标签 `polymorphicValue` 来更改多态类型的值

#### 1.4.5.5 自引用 has Many

```go
type User struct {
  gorm.Model
  Name      string
  ManagerID *uint
  Team      []User `gorm:"foreignkey:ManagerID"`
}
```

#### 1.4.5.6 外键约束

* 同 `Has One`

### 1.4.6 Many To Many（多对多关系）

**多表之间互相包含**

#### 1.4.6.1 Many To Many

* 会在两个 Model 中添加一张连接表（一个模型包含多个实体，多个实体又可以对应一个模型）

* **声明**

  ```go
  // User 拥有并属于多种 language，`user_languages` 是连接表
  type User struct {
    gorm.Model
    Languages []Language `gorm:"many2many:user_languages;"`
  }
  
  type Language struct {
    gorm.Model
    Name string
  }
  ```

* 当使用 GORM 的 `AutoMigrate` 为 `User` 创建表时，GORM 会自动创建连接表

* **反向引用**

  * **声明**

    ```go
    // User 拥有并属于多种 language，`user_languages` 是连接表
    type User struct {
      gorm.Model
      Languages []*Language `gorm:"many2many:user_languages;"`
    }
    
    type Language struct {
      gorm.Model
      Name string
      Users []*User `gorm:"many2many:user_languages;"`
    }
    ```

  * **检索**

    ```go
    // 检索 User 列表并预加载 Language
    func GetAllUsers(db *gorm.DB) ([]User, error) {
        var users []User
        err := db.Model(&User{}).Preload("Languages").Find(&users).Error
        return users, err
    }
    
    // 检索 Language 列表并预加载 User
    func GetAllLanguages(db *gorm.DB) ([]Language, error) {
        var languages []Language
        err := db.Model(&Language{}).Preload("Users").Find(&languages).Error
        return languages, err
    }
    ```

#### 1.4.6.2 重写外键

* 连接表会同时拥有两个模型的外键

  ```go
  type User struct {
    gorm.Model
    Languages []Language `gorm:"many2many:user_languages;"`
  }
  
  type Language struct {
    gorm.Model
    Name string
  }
  
  // 连接表：user_languages
  //   foreign key: user_id, reference: users.id
  //   foreign key: language_id, reference: languages.id
  ```

* 使用标签 `foreignKey`、`references`、`joinforeignKey`、`joinReferences` 进行重写

  ```go
  type User struct {
      gorm.Model
      Profiles []Profile `gorm:"many2many:user_profiles;foreignKey:Refer;joinForeignKey:UserReferID;References:UserRefer;joinReferences:ProfileRefer"`
      Refer    uint      `gorm:"index:,unique"`
  }
  
  type Profile struct {
      gorm.Model
      Name      string
      UserRefer uint `gorm:"index:,unique"`
  }
  
  // 会创建连接表：user_profiles
  //   foreign key: user_refer_id, reference: users.refer
  //   foreign key: profile_refer, reference: profiles.user_refer
  ```

* 逻辑梳理

  ```go
  type User struct {
      gorm.Model
      Profiles []Profile `gorm:"many2many:user_profiles;foreignKey:Refer;joinForeignKey:UserReferID;References:UserRefer;joinReferences:ProfileRefer"`
      Refer    uint      `gorm:"index:,unique"`
  }
  
  type Profile struct {
      gorm.Model
      Name      string
      UserRefer uint `gorm:"index:,unique"`
  }
  
  // 会创建连接表：user_profiles
  //   foreign key: user_refer_id, reference: users.refer
  //   foreign key: profile_refer, reference: profiles.user_refer
  
  // many2many:user_profiles 表示的是连接表，后面跟的所有外键和引用都是这个表中的
  // foreignKey:Refer 去连接表中查，这个是 User 的外键
  // References:UserRefer 与上方对应，修改外键对应的引用（在 Profile 中）
  // joinForeignKey:UserReferID 去连接表中查，这个是 Profile 的外键
  // joinReferences:ProfileRefer 将 UserReferID 的引用修改成 ProfileRefer
  ```

* 某些数据库只允许在唯一索引字段上创建外键，如果您在迁移时会创建外键，则需要指定 `unique index` 标签

#### 1.4.6.3 自引用 Many2Many

```go
type User struct {
  gorm.Model
    Friends []*User `gorm:"many2many:user_friends"`
}

// 会创建连接表：user_friends
//   foreign key: user_id, reference: users.id
//   foreign key: friend_id, reference: users.id
```

#### 1.4.6.4 自定义连接表

* 自定义连接表要求外键是复合主键或复合唯一索引

  ```go
  type Person struct {
    ID        int
    Name      string
    Addresses []Address `gorm:"many2many:person_addressses;"`
  }
  
  type Address struct {
    ID   uint
    Name string
  }
  
  type PersonAddress struct {
    PersonID  int `gorm:"primaryKey"`
    AddressID int `gorm:"primaryKey"`
    CreatedAt time.Time
    DeletedAt gorm.DeletedAt
  }
  
  func (PersonAddress) BeforeCreate(db *gorm.DB) error {
    // ...
  }
  
  // 修改 Person 的 Addresses 字段的连接表为 PersonAddress
  // PersonAddress 必须定义好所需的外键，否则会报错
  err := db.SetupJoinTable(&Person{}, "Addresses", &PersonAddress{})
  ```

#### 1.4.6.5 外键约束

```go
type User struct {
  gorm.Model
  Languages []Language `gorm:"many2many:user_speaks;"`
}

type Language struct {
  Code string `gorm:"primarykey"`
  Name string
}

// CREATE TABLE `user_speaks` (`user_id` integer,`language_code` text,PRIMARY KEY (`user_id`,`language_code`),CONSTRAINT `fk_user_speaks_user` FOREIGN KEY (`user_id`) REFERENCES `users`(`id`) ON DELETE SET NULL ON UPDATE CASCADE,CONSTRAINT `fk_user_speaks_language` FOREIGN KEY (`language_code`) REFERENCES `languages`(`code`) ON DELETE SET NULL ON UPDATE CASCADE);
```

#### 1.4.6.6 复合外键

* 模型使用了**复合主键**，GORM 会默认启用

  ```go
  type Tag struct {
    ID     uint   `gorm:"primaryKey"`
    Locale string `gorm:"primaryKey"`
    Value  string
  }
  
  type Blog struct {
    ID         uint   `gorm:"primaryKey"`
    Locale     string `gorm:"primaryKey"`
    Subject    string
    Body       string
    Tags       []Tag `gorm:"many2many:blog_tags;"`
    LocaleTags []Tag `gorm:"many2many:locale_blog_tags;ForeignKey:id,locale;References:id"`
    SharedTags []Tag `gorm:"many2many:shared_blog_tags;ForeignKey:id;References:id"`
  }
  
  // 连接表：blog_tags
  //   foreign key: blog_id, reference: blogs.id
  //   foreign key: blog_locale, reference: blogs.locale
  //   foreign key: tag_id, reference: tags.id
  //   foreign key: tag_locale, reference: tags.locale
  
  // 连接表：locale_blog_tags
  //   foreign key: blog_id, reference: blogs.id
  //   foreign key: blog_locale, reference: blogs.locale
  //   foreign key: tag_id, reference: tags.id
  
  // 连接表：shared_blog_tags
  //   foreign key: blog_id, reference: blogs.id
  //   foreign key: tag_id, reference: tags.id
  ```

  



