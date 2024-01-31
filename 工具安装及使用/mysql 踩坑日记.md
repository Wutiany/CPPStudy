# mysql 踩坑日记

# 数据库连接

## Public Key Retrieval is not allowed

* 身份插件的问题

* 修改设置或者进入数据库修改

  * 修改设置

    ```
    skip-grant-tables
    default-authentication-plugin=mysql_native_password
    ```

  * 进入数据库修改，将所有 `id` 的连接（%）都设置为 `mysql_native_password`

    ```shell
    $ ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY 'password';
    ```


## Access denied for user 'root'@'ip' (using password: YES)

* 未被授权（仅授权 localhost）：在 `mysql` 中使用 `show grants;` 查看授权

* 修改授权

  ```shell
  $ GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'password' WITH GRANT OPTION;
  ```

* 会报错，要创建一个 `'root'@'%'` 用户

  ```mysql
  CREATE USER 'root'@'%' IDENTIFIED BY 'password';
  GRANT ALL PRIVILEGES ON *.* TO 'root'@'%';
