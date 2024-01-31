# jenkins 开发环境部署

# jenkins 安装

**jenkins 使用 java 构建的，所以安装环境中需要有 java**

* 安装 `java`

  ```shell
  $ apt install -y openjdk-17-jre-headless
  ```

* 获取 Jenkins GPG 密钥（需要官方稳定版的软件源）

  ```shell
  $ curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
    /usr/share/keyrings/jenkins-keyring.asc > /dev/null
  ```

* 将存储库加入源列表

  ```shell
  $ echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
    https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
    /etc/apt/sources.list.d/jenkins.list > /dev/null
  ```

* 设置自启动

  ```shell
  $ systemctl status jenkins  // 查看状态
  $ systemctl enable jenkins
  ```

* 网页版访问 `http://localhost:8080`

  * 远程服务器，`localhost` 修改成 `ip` 或者域名

# jenkins 登录配置

## jenkins web 登录

* 网页版访问地址：`http://localhost:8080`
* 远程服务器（云服务器需要去安全组放开端口）：`http://ip:8080`，或者 `ip` 替换成域名

## 获取初始管理员密码

```shell
$ cat /var/lib/jenkins/secrets/initialAdminPassword
```

# jenkins 配置

* 安装插件

* 设置用户（设置自己的一个管理用户）
* 默认即可

# jenkins 使用

## github 无法登录问题

* 使用 `gitee` 同步 `github` 仓库，使用 `gitee` 来进行设置项目

[如何将一个项目同时提交到GitHub和Gitee(码云)上 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/346400298)

## gitee 使用创建项目

[Jenkins 插件 | Gitee 产品文档](https://help.gitee.com/devops/connect/Jenkins-Plugin/#:~:text=添加 Gitee 链接配置 1 前往 Jenkins -> Manage,Advanced ，可配置是否忽略 SSL 错误（视您的 Jenkins 环境是否支持），并可设置链接测超时时间（视您的网络环境而定） More items)