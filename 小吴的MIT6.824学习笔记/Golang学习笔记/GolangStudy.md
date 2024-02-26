# Golang 学习
## Golang环境配置
### Ubuntu 配置
#### Golang下载注意事项
* 下载完之后记得将环境变量配置到 `~/.bashrc` 中
``` shell
sudo vim ~/.bashrc

# 在文件最下方加入一下代码
export GOROOT=/usr/local/go  # Go环境所在的位置
export GOBIN=$GOROOT/bin
export PATH=$PATH:$GOBIN
export GOPATH=/home/GoProject # Go工作目录所在的位置

# 保存之后刷新配置
source ~/.bashrc
```
#### Goland配置注意事项
* Goland导入自己的包报错出现 `GOROOT` 目录下找不到说明打开了 `GO111MODULE=on`，该情况下只在 `GOROOT` 中找包，需要取消配置
  * Goland 输入 `unset GO111MODULE` 先清除命令（如果直接设置会报警告的情况下）
  * 在输入 `go env -w GO111MODULE=off` 进行设置
* 设置完成后也要在 `Goland` 的 `setting` 中关闭 `Go modules` 的 `Enable Go modules integration`

## Golang基础知识
### Golang 常见的四种变量声明
```Golang
var number int
number := variable


```

* 使用 `:=` 进行变量定义的好处
  * 不需要预知对象的类型，类似于 `C++` 中的 `auto` 一样，可以推断变量类型，配合 `interface{}` 万能变量类型使用

###