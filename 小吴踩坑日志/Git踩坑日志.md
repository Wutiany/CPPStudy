# Git 踩坑日志

## push 发生未获得本地 ssl 证书

*  **error**： `SSL certificate problem: unable to get local issuer certificate`

* 原因：

  > 问题是 git 默认使用“Linux”加密后端
  >
  > 从 Git for Windows 2.14 开始，现在可以将 Git 配置为使用 SChannel（内置的 Windows 网络层）作为加密后端。这意味着它将使用 Windows 证书存储机制，您无需显式配置 curl CA 存储机制
  >
  > 只需要执行
  >
  > `git config --global http.sslbackend schannel`

* 参考：[无法在具有自签名证书的 Windows 上使用 git 解决“无法获取本地颁发者证书”的问题 - Stack Overflow](https://stackoverflow.com/questions/23885449/unable-to-resolve-unable-to-get-local-issuer-certificate-using-git-on-windows)