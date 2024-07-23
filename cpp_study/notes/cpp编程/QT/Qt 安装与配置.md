# Qt 安装文档

## Centos 下安装

### 开发机基本要求

因为需要 openGL 所以要先安装

```shell
sudo yum groupinstall "C Development Tools and Libraries"
sudo yum install mesa-libGL-devel
```

Qt 需要访问的依赖头文件，需要安装

```shell
yum install libfontconfig1-dev
yum install yum install libfreetype6-dev
yum install libx11-dev
yum install libx11-xcb-dev 
yum install libxext-dev
yum install libxfixes-dev
yum install libxi-dev
yum install libxrender-dev
yum install libxcb1-dev
yum install libxcb-glx0-dev
yum install libxcb-keysyms1-dev
yum install libxcb-image0-dev
yum install libxcb-shm0-dev
yum install libxcb-icccm4-dev
yum install libxcb-sync-dev 
yum install libxcb-xfixes0-dev
yum install libxcb-shape0-dev
yum install libxcb-randr0-dev
yum install libxcb-render-util0-dev
yum install libxcb-xinerama-dev
yum install libxkbcommon-dev
yum install libxkbcommon-x11-dev
```

