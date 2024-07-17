# boost

## 安装与配置

* 下载：

  * [Boost C++ Library - 在 SourceForge.net 浏览 /boost-binaries](https://sourceforge.net/projects/boost/files/boost-binaries/)
  * http://www.boost.org/

* 解压源文件

  * bz2：需要下载 bz2 解压，使用 tar -xjf

* 进行编译

  * linux 下使用 bootstrap.sh 设置编译器和所选需要编译的库
    * 命令：`./bootstrap.sh --with-libraries=all --with-toolset=gcc`； --with-libraries=all: 为指定编译 boost 的哪些库，--with-toolset: 指定编译使用的编译器

  * 进行编译：`./b2 toolset=gcc`

    😡：编译出错，缺少 pyconfig.h（确实 python 开发库头文件）

    😊：安装 python3，设置使用 python3：`./bootstrap.sh --with-libraries=all --with-toolset=gcc --with-python=python3`，在进行编译

    👍：安装的 python3 的时候使用：`yum install python36-devel`（centos 系统下）

* 安装 boost
  * 命令：`./b2 install --prefix=/usr`
    * --prefix 指定 boost 安装的目录
    * 不指定目录，默认安装在：/usr/local/include/boost, /usr/local/lib/

* 使用

  * 首先需要配置系统的环境变量：

    * include 文件所在的位置
    * lib 文件所在的位置

    ```shell
    #gcc找到头文件的路径
    C_INCLUDE_PATH=/usr/include
    export C_INCLUDE_PATH
     
    #g++找到头文件的路径
    CPLUS_INCLUDE_PATH=$CPLUS_INCLUDE_PATH:/usr/include/
    export CPLUS_INCLUDE_PATH
     
    #gcc和g++在编译的链接(link)阶段查找库文件的目录列表
    LIBRARY_PATH=$LIBRARY_PATH:/usr/lib
    export LIBRARY_PATH
    ```

  * 编译的时候加入使用的链接库：`g++ main.cpp -g -o main -lboost_system -lboost_thread -lpthread `

    * boost_thread 需要 pthread



