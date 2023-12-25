# etcd 笔记

# 1 基本配置

## 1.1 命令

* 引入包

  ```go
  go get github.com/coreos/etcd/clientv3
  ```

# 2 操作

## 2.1 操作逻辑

**etcd 相当于一个分布式 kv 数据库，通过类似 zookeeper 的结构（文件树）来标识 key（只是用来分辨 key，不是真正的文件树），同样存在 key 过期的设置**

* 想要操作 `etcd` 就需要获取一个**client（客户端）**（客户端内部会**自动重连**）

* 从客户端获取一个 `kv` **键值库**，用来操作存储 `kv`（一个操作 kv 键值库的**接口**）

  **每个 kv 的属性**

  | 属性                                     | 描述                                                         |
  | ---------------------------------------- | ------------------------------------------------------------ |
  | Key（键）                                | 键是一个唯一的字符串，用于在 etcd 中标识一个特定的值         |
  | Value（值）                              | 值是与键关联的数据。它可以是任意字节序列，没有特定的数据类型限制 |
  | Create Revision（创建版本）              | 创建版本是指键值对在 etcd 中被创建时的修改索引。每个键值对都有一个唯一的创建版本号 |
  | Mod Revision（修改版本）                 | 修改版本是指键值对最后一次被修改时的修改索引。每次对键值对进行更新操作时，修改版本号都会增加 |
  | Version（版本）                          | 版本是指键值对的修改次数。每次更新键值对时，版本号都会递增   |
  | Lease（租约）                            | 租约是 etcd 中的一种机制，用于设置键值对的生命周期。键值对可以与一个租约进行关联，租约到期后，键值对会被自动删除 |
  | Lease Time-to-Live (TTL)（租约到期时间） | 租约到期时间是指与键值对关联的租约的剩余时间。这表示键值对将在多长时间后过期 |
  | TTL Refresh（租约刷新）                  | 租约刷新是指通过更新键值对的租约，延长租约的过期时间         |
  | Deleted（删除标志）                      | 删除标志表示键值对是否已被删除。如果删除标志为 true，则表示该键值对已被删除 |

  **每个属性 client 会提供对应的方法进行获取结果**

* 从**接口提供的方法**中进行**操作** kv 键值库
* **操作数据库**的方法：
  * **Put**：`put` 一个 `kv`，返回一个 `PutResponse`（不同的 `kv` 对应着不同的 `response`）
  * **Get**：获取给定的 `kv`， 也可以给 `kv` 的获取增加**限定条件**（前缀查找等）
    * 通过最后一个参数 `opts ...OpOption`来获取，包提供方法（OpOption）
  * **Lease**：租约对象，授权一个显示访问的对象
  * **Op**：一个抽象操作，`Put，Get` 也是 `Op`
    * **OpDelete**
    * **OpGet**
    * **OpPut**
    * **OpTxn**
  * **Txn**：原子执行，支持 `if ... then ... else ...` 表达式（链式）
    * `if` 条件中可以使用 `clientv3.Compare` 方法进行比较 **版本、值**等
    * 最后使用 `Commit` 提交
  * **Watch**：监听某个键的变化，配置监控

**创建客户端连接**

**获取键值库**

**使用键值库提供的方法操作键值库**

**其他功能**

* Lease
* Txn
* Watch

## 2.2 代码使用

### 2.2.1 连接客户端

* 创建客户端函数：

  ```go
  client, err := clientv3.New(clientv3.Config{
      Endpoints: []string{"localhost:2379"},
      DialTimeout: 5 * time.Second,
  })
  ```

* 参数

  * **Endpoints**：etcd 的多个节点服务地址
  * **DialTimeout**：首次连接超时时间，超时时间没成功返回 err，同时首次连接上之后**不需要进行重连**，内部源码会进行**重连**

* 返回的客户端结构体：提供了一些其他操作，通过这个结构体来**创建**操作 kv 数据库的接口等； **client 成员对应着的就是其功能**

  * **Cluster**：向集群添加 etcd 服务端节点
  * **KV**：kv 数据库的操作接口；**底层**通过 gPRC 的客户端进行创建的连接
  * **Lease**：租约相关操作
  * **Watcher**：订阅观察，监听`kv` 变化，这就是用 `etcd` 作为服务管理的原因、
  * **Auth**：管理 `etcd` 的用户和权限，属于管理员操作
  * **Maintenance**：维护 `etcd`，比如主动迁移 `etcd` 的 `leader` 节点

### 2.2.2 访问 KV 数据库

* KV interface 提供了访问数据库的所有方法，同时还可以使用 Do 进行 Op 方法的操作

  ```go
  type KV interface {
      Put(ctx context.Context, key string, val string, opts ...OpOption) (*PutResponse, error)
      Get(ctx context.Context, key string, opts ...OpOption) (*GetResponse, error)
      Delete(ctx context.Context, key string, opts ...OpOption) (*DeleteResponse, error)
      Compact(ctx context.Context, rev int64, opts ...CompactOption) (*CompactResponse, error) // 删除旧数据收回存储空间的方法
      Do(ctx context.Context, op Op) (OpResponse, error)
      Txn(ctx context.Context) Txn
  }
  ```

* 创建 KV 接口（clientv3 提供了方法）：`NewKV(*client)`

### 2.2.3 操作 KV 数据库

**因为使用的 gRPC 作为底层通信，所以操作都会有一个 Response 作为返回值，同时 Get 方法的 err，不能表述出 kv 不存在的错误，需要通过 Response**

* **Put**：`Put(ctx context.Context, key, val string, opts ...OpOption) (*PutResponse, error)`
  * 支持可变长参数（opts ...OpOption）：用来传递一些对 `put` 操作有影响的操作，如携带一个 `lease ID` 支持 `key` 过期
* **Get**：`Get(ctx context.Context, key string, opts ...OpOption) (*GetResponse, error)`
  * `err` **不能**反映 `key` 是否存在（只会返回异常），需要通过 `GetResponse` 判断 `key` 是否存在
  * `GetResponse` 返回 `RangeResponse`，其中 `Kvs` 是用来遍历返回结果的
  * 增加 `OpOption` **改变查找的行为**
    * `clientv3.WitPrefix()` 查找前缀
    * `clientv3.WithLimit(int)` 限定查找数据，可以结合 `More，Count` 字段作为翻页查找使用
    * 等等
    * `KV` **有序存储**，**前缀**相近的，查询结果会在一起

* **Lease**：`lease` 对象是一个 `interface`

  * 获取：`clientv3.NewLease(*client)`
  * 提供的方法：
    * Grant：分配一个租约
    * Revoke：释放一个租约
    * TimeToLive：获取剩余TTL时间
    * Leases：列举所有etcd中的租约
    * KeepAlive：自动定时的续约某个租约
    * KeepAliveOnce：为某个租约续约一次
    * Close：释放当前客户端建立的所有租约
  * 通过这些方法实现 **key 自动过期**等功能：`kv.Put(context.TODO(), "/test/vanish", "vanish in 10s", clientv3.WithLease(grantResp.ID))`；`grantResp` 授予的租约的响应
  * etcd **没有提供原子**的 `Put with Lease`

* **Op**：Do 方法会接受一个 Op，Op 也是由包提供的

  * Do 会根据 Op 的类型进行对应 `Response` 类型的返回

  * 提供的 Op

    * func OpDelete(key string, opts …OpOption) Op
    * func OpGet(key string, opts …OpOption) Op
    * func OpPut(key, val string, opts …OpOption) Op
    * func OpTxn(cmps []Cmp, thenOps []Op, elseOps []Op) Op

  * 返回类型：

    ```go
    type OpResponse struct {
        put *PutResponse
        get *GetResponse
        del *DeleteResponse
        txn *TxnResponse
    }
    ```

    什么类型的 Op，就去调用什么类型的 Response 指针

* **Txn**：事务接口，链式操作

  * Interface

    ```go
    type Txn interface {
        // If takes a list of comparison. If all comparisons passed in succeed,
        // the operations passed into Then() will be executed. Or the operations
        // passed into Else() will be executed.
        If(cs ...Cmp) Txn
    
        // Then takes a list of operations. The Ops list will be executed, if the
        // comparisons passed in If() succeed.
        Then(ops ...Op) Txn
    
        // Else takes a list of operations. The Ops list will be executed, if the
        // comparisons passed in If() fail.
        Else(ops ...Op) Txn
    
        // Commit tries to commit the transaction.
        Commit() (*TxnResponse, error)
    }
    ```

  * 支持 `If(cs ...Cmp)Then(ops .,. Op)Else(ops .,. Op)`

  * Cmp：`clientv3.Compare(cmp Cmp, result string, v interface{}) Cmp`

    * Methods on (\*Cmp):
      KeyBytes() []byte
      WithKeyBytes(key []byte)
      ValueBytes() []byte
      WithValueBytes(v []byte)
    * Methods on (Cmp):
      WithRange(end string) clientv3.Cmp
      WithPrefix() clientv3.Cmp

    > ```go
    > kv.Txn(context.TODO()).If(
    >  clientv3.Compare(clientv3.Value(k1), ">", v1),
    >  clientv3.Compare(clientv3.Version(k1), "=", 2)
    > ).Then(
    >  clientv3.OpPut(k2,v2), clentv3.OpPut(k3,v3)
    > ).Else(
    >  clientv3.OpPut(k4,v4), clientv3.OpPut(k5,v5)
    > ).Commit()
    > ```
    >
    > 其他类似 Value 的方法
    >
    > - func CreateRevision(key string) Cmp：key=xxx的创建版本必须满足…
    > - func LeaseValue(key string) Cmp：key=xxx的Lease ID必须满足…
    > - func ModRevision(key string) Cmp：key=xxx的最后修改版本必须满足…
    > - func Value(key string) Cmp：key=xxx的创建值必须满足…
    > - func Version(key string) Cmp：key=xxx的累计更新次数必须满足…

* **Watch**：监听 `kv` 变化，调用后会返回 `WatchChan`，发生变化，向 `Chan` 发送 `WatchResponse`

  * ```go
    type WatchChan <-chan WatchResponse
    
    type WatchResponse struct {
        Header pb.ResponseHeader
        Events []*Event
    
        CompactRevision int64
    
        Canceled bool
    
        Created bool
    }
    ```

  * 

* 返回类型（Response）

  ```go
  type (
     CompactResponse pb.CompactionResponse
     PutResponse     pb.PutResponse
     GetResponse     pb.RangeResponse
     DeleteResponse  pb.DeleteRangeResponse
     TxnResponse     pb.TxnResponse
  )
  ```

  * **PutResponse**:

    ```go
    type PutResponse struct {
        // 保存了本次更新的 revision 信息
        Header               *ResponseHeader  `protobuf:"bytes,1,opt,name=header,proto3" json:"header,omitempty"`
        // PrevKv 返回 Put 覆盖之前的 Value
        PrevKv               *mvccpb.KeyValue `protobuf:"bytes,2,opt,name=prev_kv,json=prevKv,proto3" json:"prev_kv,omitempty"`
        XXX_NoUnkeyedLiteral struct{}         `json:"-"`
        XXX_unrecognized     []byte           `json:"-"`
        XXX_sizecache        int32            `json:"-"`
    }
    
    // function 
    Reset()
    String() string
    ProtoMessage()
    Descriptor() ([]byte, []int)
    XXX_Unmarshal(b []byte) error
    XXX_Marshal(b []byte, deterministic bool) ([]byte, error)
    XXX_Merge(src proto.Message)
    XXX_Size() int
    XXX_DiscardUnknown()
    GetHeader() *etcdserverpb.ResponseHeader
    GetPrevKv() *mvccpb.KeyValue
    Marshal() (dAtA []byte, err error)
    MarshalTo(dAtA []byte) (int, error)
    MarshalToSizedBuffer(dAtA []byte) (int, error)
    Size() (n int)
    Unmarshal(dAtA []byte) error
    ```

  * **RangeResponse**：Get 获取的结果存储的结构体

    ```go
    type RangeResponse struct {
        Header               *ResponseHeader    `protobuf:"bytes,1,opt,name=header,proto3" json:"header,omitempty"`
        Kvs                  []*mvccpb.KeyValue `protobuf:"bytes,2,rep,name=kvs,proto3" json:"kvs,omitempty"`
        More                 bool               `protobuf:"varint,3,opt,name=more,proto3" json:"more,omitempty"`
        Count                int64              `protobuf:"varint,4,opt,name=count,proto3" json:"count,omitempty"`
        XXX_NoUnkeyedLiteral struct{}           `json:"-"`
        XXX_unrecognized     []byte             `json:"-"`
        XXX_sizecache        int32              `json:"-"`
    }
    
    // function
    
    Reset()
    String() string
    ProtoMessage()
    Descriptor() ([]byte, []int)
    XXX_Unmarshal(b []byte) error
    XXX_Marshal(b []byte, deterministic bool) ([]byte, error)
    XXX_Merge(src proto.Message)
    XXX_Size() int
    XXX_DiscardUnknown()
    GetHeader() *etcdserverpb.ResponseHeader
    GetKvs() []*mvccpb.KeyValue
    GetMore() bool
    GetCount() int64
    Marshal() (dAtA []byte, err error)
    MarshalTo(dAtA []byte) (int, error)
    MarshalToSizedBuffer(dAtA []byte) (int, error)
    Size() (n int)
    Unmarshal(dAtA []byte) error
    ```



# * 参考文档

[etcd 简明教程](https://zhuanlan.zhihu.com/p/89446425)

[etcd  微服务实践](https://www.cnblogs.com/jiujuan/p/13200898.html)

[官方文档](https://pkg.go.dev/github.com/coreos/etcd/clientv3)



