# 数据库安装

## windows 下mysql安装

参考博客：[Windows MYSQL社区版8.1下载安装（MSI）_mysql社区版下载-CSDN博客](https://blog.csdn.net/weixin_68256171/article/details/132107858)

提示缺少 `Microsoft Visual Studio 2019 Redistributable`，参考博客：[MySQL msi安装缺少‘Microsoft Visual Studio 2019 Redistributable’_vstdio2019redistributable-CSDN博客](https://blog.csdn.net/qq_41563277/article/details/104665146)

[点击这里](https://visualstudio.microsoft.com/downloads/)

点击最底部：**Other Tools and Frameworks**一栏，
下载
**Microsoft Visual C++ Redistributable for Visual Studio 2019**

## Ubuntu 安装 Mysql

* 直接使用包管理安装

  ```shell
  $ sudo apt install mysql-server
  ```

* 服务未开启，使用以下命令开启

  ```shell
  $ sudo /etc/init.d/mysql start
  ```

* 查看版本

  ```shell
  $ mysql -V
  ```

* 初始化数据库

  ```shell
  $ mysql_secure_installation
  ```

* 调整用户身份验证

  * 检查用户对应的身份验证方法

    ```mysql
    SELECT user,authentication_string,plugin,host FROM mysql.user;
    ```

    ![image-20231012135054004](..\src\photo\image-20231012135054004.png)

  * 使用 'ALTER USER' 修身份验证方法

    ```mysql
    ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'your_password';
    ```

  * 修改完成后重新加载权限更新到数据库中

    ```mysql
    FLUSH PRIVILEGES;
    ```

  * 再次登陆时需要验证

    ```shell
    $ mysql -u root -p
    ```

* 创建新用户

  ```mysql
  CREATE USER 'wty'@'localhost' IDENTIFIED BY 'password';
  GRANT ALL PRIVILEGES ON *.* TO 'wty'@'localhost' WITH GRANT OPTION;
  ```

* **配置服务器远程连接**

  * 在防火墙打开一个端口

    ```shell
    $ sudo ufw enable
    $ sudo ufw allow mysql
    ```

  * 安装 ufw 工具

    ```shell
    $ sudo apt install ufw
    ```

  * 也可以为 mysql 打开端口（默认3306）

    ```shell
    $ sudo ufw allow 3306
    ```

  * 将 mysql 启动时运行

    ```shell
    $ sudo systemctl enable mysql
    ```

  * 配置接口（默认127.0.0.1:3306）

    ```shell
    sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
    ```

## Ubuntu 安装 Redis

