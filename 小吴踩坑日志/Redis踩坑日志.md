# Redis踩坑日志

## Redis connect ping faild, err: ERR Client sent AUTH, but no password is set

* redis 未设置密码的错误，两种设置密码的方式

  * 通过 redis-cli.exe 进行设置密码（客户端关闭之后就会**失效**）

    ```shell
    > config set requirepass password
    > auth password
    ```

  * 通过修改配置文件（redis.windows.conf）

    ```
    requirepass password
    ```

    * linux 和 windows 都可以使用，但是 windows 直接执行客户端会报找不到配置文件的错误
    * windows 需要在 shell 中执行 `> redis-server.exe redis.windows.conf`
