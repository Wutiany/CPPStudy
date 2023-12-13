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

### 1.3.5 选择特定字段

* **Select 语句**，同时可以使用 **SQL 的函数**

  ```go
  db.Select("name", "age").Find(&users)
  // SELECT name, age FROM users;
  
  db.Select([]string{"name", "age"}).Find(&users)
  // SELECT name, age FROM users;
  
  db.Table("users").Select("COALESCE(age,?)", 42).Rows()
  // SELECT COALESCE(age,'42') FROM users;
  ```

### 1.3.6 排序（Order by）

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

### 1.3.7 Limit & Offset

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

### 1.3.8 Group By & Having

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

### 1.3.9 Distinct

* 结果去重

  ```go
  db.Distinct("name", "age").Order("name, age desc").Find(&results)
  ```

`Distinct`也可以与`Pluck`和`Count`一起使用

### 1.3.10 Joins

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

* Joins

  ```go
  db.Joins("Company").Find(&users)
  // SELECT `users`.`id`,`users`.`name`,`users`.`age`,`Company`.`id` AS `Company__id`,`Company`.`name` AS `Company__name` FROM `users` LEFT JOIN `companies` AS `Company` ON `users`.`company_id` = `Company`.`id`;
  
  // inner join
  db.InnerJoins("Company").Find(&users)
  // SELECT `users`.`id`,`users`.`name`,`users`.`age`,`Company`.`id` AS `Company__id`,`Company`.`name` AS `Company__name` FROM `users` INNER JOIN `companies` AS `Company` ON `users`.`company_id` = `Company`.`id`;
  ```

* 

### 1.3.* 总结

* 这些操作都是在 `tx` 的基础上进行的，一个事务逻辑上等同一个数据库连接
* 如果使用列表，那默认就是主键的条件（IN）
* 同一个 `tx`，可以在一个语句中多次查询多次修改条件
* 可以带条件的，都会有**占位符**，多参数输入后续参数是 `Any`



## 1.4 数据库设置

### 1.4.1 指定数据库模型

* `db.Moedel(&User{})`：通过这个指定数据库模型， 返回一个 `tx *DB`（事务）

### 1.4.2 指定表

* `db.Table("users")`：通过这个指定表，同样返回一个事务

