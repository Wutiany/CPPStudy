# windows server 部署 oracle 11g

## 1 下载 oracle 11g

### 1.1 迅雷下载链接

http://download.oracle.com/otn/nt/oracle11g/112010/win64_11gR2_database_1of2.zip

http://download.oracle.com/otn/nt/oracle11g/112010/win64_11gR2_database_2of2.zip

## 2 安装

### 2.1 解压缩放置于同一路径

### 2.2 安装过程

* 直接 setup 安装
* ![image-20231121103531935](..\src\photo\image-20231121103531935.png)
  * 直接点**是**，进入下一步
* ![image-20231121103741686](..\src\photo\image-20231121103741686.png)
  * 取消配置安全更新
  * 警告直接点**是**
* ![image-20231121103919873](..\src\photo\image-20231121103919873.png)
  * 选择创建和配置数据库
* ![image-20231121104144164](..\src\photo\image-20231121104144164.png)
  * 选择服务器类
* ![image-20231121104242489](..\src\photo\image-20231121104242489.png)
  * 单实例数据库安装
* ![image-20231121104348646](..\src\photo\image-20231121104348646.png)
  * 高级安装
* ![image-20231121104556916](..\src\photo\image-20231121104556916.png)
  * 选择默认语言
* ![image-20231121104640761](..\src\photo\image-20231121104640761.png)
  * 选择安装企业版
* ![image-20231121104730406](..\src\photo\image-20231121104730406.png)
  * 只选择基目录就可以，本次不做选择，只有系统盘
* ![image-20231121104839958](..\src\photo\image-20231121104839958.png)
  * 选择一般用途
* ![image-20231121104948756](..\src\photo\image-20231121104948756.png)
  * 数据库标识符（默认即可）
* ![image-20231121105049362](..\src\photo\image-20231121105049362.png)![image-20231121105123784](..\src\photo\image-20231121105123784.png)
  * 进入**字符集配置**
  * 选择 `Unicode` 编码
* ![image-20231121105413840](..\src\photo\image-20231121105413840.png)
  * **不启用通知**，直接下一步
* ![image-20231121105502685](..\src\photo\image-20231121105502685.png)
  * 数据库存储，不需要改变，直接下一步
* ![image-20231121105553454](..\src\photo\image-20231121105553454.png)
  * 自动备份根据需求进行选择，这里选择不启动自动备份
* ![image-20231121105745435](..\src\photo\image-20231121105745435.png)
  * 根据需求选择口令，这里选择全部统一
* 其余都默认完成等待安装即可，口令根据后续需求在配置

### 2.3 修改 host

将 `listener.ora` 与 `tnsnames.ora` 中的 `HOST` 替换成本地的 `ip`

* ![image-20231121111304455](..\src\photo\image-20231121111304455.png)

  ![image-20231121111504226](D:\github\CPPStudy\src\photo\image-20231121111504226.png)

  * 安装了 **服务器版**的应该不需要改这个

## 3 使用

### 3.1 远程连接的使用

* 使用 `DBeaver` 服务器管理软件进行远程登陆

  ![image-20231121113445258](..\src\photo\image-20231121113445258.png)

* 但是会出现错误，所以要进入 `SQL Plus` 去修改权限

  * 找到 `SQL Plus`

    ![image-20231121113609785](..\src\photo\image-20231121113609785.png)

  * 登陆

    ![image-20231121113640152](..\src\photo\image-20231121113640152.png)

  * 输入 `alter user system account unlock;` 给用户解锁

    ![image-20231121113748813](..\src\photo\image-20231121113748813.png)

  * 输入 `connect system/**** as sysdba;` 给用户授权

    * system 是用户名
    * \**** 是密码

    ![image-20231121114136302](..\src\photo\image-20231121114136302.png)

* 登陆选项修改成 `Normal` 即可登陆

  ![image-20231121114446751](..\src\photo\image-20231121114446751.png)

### 3.2 客户端使用

`SQL Plus`
