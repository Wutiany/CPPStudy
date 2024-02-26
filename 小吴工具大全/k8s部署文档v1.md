# 部署文档v1

# 🌄部署

> kubernetes集群主(控制)节点一般为奇数个，如1/3/5；从(负载)节点数量无要求

## 部署前准备工作 preflight

系统安装：发行版本与内核版本

磁盘分区

检查时区，同步时间

关闭交换分区

组建集群的机器都配置到/etc/hosts

ssh免密：控制节点到工作节点

> [docker 安装](https://docs.docker.com/engine/install/) 目前已不再支持docker-shim，从1.22开始推荐containerd

### 安装containerd

修改配置

```shell
# 生成默认配置文件
mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# 替换三处配置

# 1.开启systemd cgroup
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  ...
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true	# false 修改为 true

# 2.替换sandbox_image 镜像地址
[plugins."io.containerd.grpc.v1.cri"]
  ...
  # sandbox_image = "k8s.gcr.io/pause:3.6"
  sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.9"

# 3.镜像源加速
      [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
          endpoint = ["https://hub-mirror.c.163.com"]

# 重启containerd
systemctl restart containerd
```

‍

目前流行的安装方式

## 使用kubeadm 部署集群

* #### 🏃‍♂️执行安装脚本

  ```bash
  #!/bin/bash
  set -x

  WORKDIR=$(pwd)
  DOWNLOAD_DIR=files
  mkdir $DOWNLOAD_DIR
  cd $DOWNLOAD_DIR

  export https_proxy=http://10.10.201.28:7890

  ARCH="amd64"

  #$(curl -L -s https://dl.k8s.io/release/stable.txt)
  KUBECTL_VERSION=$(curl  -L -s https://dl.k8s.io/release/stable.txt)

  CNI_PLUGINS_VERSION="v1.3.0"
  CRICTL_VERSION="v1.28.0"

  # kubeadm kubelet $(curl -sSL https://dl.k8s.io/release/stable.txt)
  RELEASE=$(curl -sSL https://dl.k8s.io/release/stable.txt)
  # kubelet kubeadm service & conf
  RELEASE_VERSION="v0.16.2"

  # download
  curl -LO "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/amd64/kubectl"
  curl -LO "https://github.com/containernetworking/plugins/releases/download/${CNI_PLUGINS_VERSION}/cni-plugins-linux-${ARCH}-${CNI_PLUGINS_VERSION}.tgz" 
  curl -LO "https://github.com/kubernetes-sigs/cri-tools/releases/download/${CRICTL_VERSION}/crictl-${CRICTL_VERSION}-linux-${ARCH}.tar.gz"
  curl -LO --remote-name-all https://dl.k8s.io/release/${RELEASE}/bin/linux/${ARCH}/{kubeadm,kubelet}
  curl -sSLO "https://raw.githubusercontent.com/kubernetes/release/${RELEASE_VERSION}/cmd/krel/templates/latest/kubelet/kubelet.service" 
  curl -sSLO "https://raw.githubusercontent.com/kubernetes/release/${RELEASE_VERSION}/cmd/krel/templates/latest/kubeadm/10-kubeadm.conf"

  cd $WORKDIR

  # install 

  BIN_DIR=/usr/local/bin
  CNI_DIR="/opt/cni/bin"

  sudo install -o root -g root -m 0755 ${DOWNLOAD_DIR}/kubectl ${BIN_DIR}/kubectl
  sudo install -o root -g root -m 0755 ${DOWNLOAD_DIR}/kubelet ${BIN_DIR}/kubelet
  sudo install -o root -g root -m 0755 ${DOWNLOAD_DIR}/kubeadm ${BIN_DIR}/kubeadm

  sudo mkdir -p "$CNI_DIR"
  sudo tar -C "$CNI_DIR" -xz -f  ${DOWNLOAD_DIR}/cni-plugins-linux-amd64-v1.3.0.tgz

  sudo tar -C ${BIN_DIR} -xz -f ${DOWNLOAD_DIR}/crictl-v1.28.0-linux-amd64.tar.gz

  sed "s:/usr/bin:${BIN_DIR}:g"  ${DOWNLOAD_DIR}/kubelet.service | sudo tee /etc/systemd/system/kubelet.service 

  mkdir -p /etc/systemd/system/kubelet.service.d
  sed "s:/usr/bin:${BIN_DIR}:g" ${DOWNLOAD_DIR}/10-kubeadm.conf | sudo tee /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

  ```

* #### 🎇启动kubelet

  ```yaml
  systemctl enable kubelet --now
  ```

* #### 🤲获取默认的初始化参数文件

  ```yaml
  kubeadm config print init-defaults > kubeadm-conf.yaml
  ```

* #### 🧐修改配置文件

  ```yaml
  apiVersion: kubeadm.k8s.io/v1beta3
  bootstrapTokens:
  - groups:
    - system:bootstrappers:kubeadm:default-node-token
    token: abcdef.0123456789abcdef
    ttl: 24h0m0s
    usages:
    - signing
    - authentication
  kind: InitConfiguration
  localAPIEndpoint:
    advertiseAddress: 10.10.201.28 # master节点ip地址，如果 Master 有多个interface，建议明确指定， 修改成自己的 ip
    bindPort: 6443
  nodeRegistration:
    criSocket: unix:///var/run/containerd/containerd.sock
    imagePullPolicy: IfNotPresent
    name: dev    # 通过 hostname 查看主机名，修改成自己的
    taints: null
  ---
  apiServer:
    timeoutForControlPlane: 4m0s
  #  certSANs:          #若有其他IP需要签进证书里，配置此项
  #  - 100.X.X.X
  #  - hostnameX
  apiVersion: kubeadm.k8s.io/v1beta3
  certificatesDir: /etc/kubernetes/pki
  clusterName: kubernetes
  # controlPlaneEndpoint: "10.10.201.28:6443"   # 若开启HA,则配置此项
  controllerManager: {}
  dns: {}
  etcd:
    local:
      dataDir: /var/lib/etcd
  imageRepository: registry.aliyuncs.com/google_containers  #修改为阿里镜像地址
  kind: ClusterConfiguration
  kubernetesVersion: 1.28.0
  networking:
    dnsDomain: cluster.local
    serviceSubnet: 10.96.0.0/12 # 指定Service网段
    podSubnet: 10.244.0.0/16 # 指定Pod网段
  scheduler: {}
  ---
  apiVersion: kubeproxy.config.k8s.io/v1alpha1
  kind: KubeProxyConfiguration
  mode: ipvs
  ---
  apiVersion: kubelet.config.k8s.io/v1beta1
  kind: KubeletConfiguration
  cgroupDriver: systemd
  ```

* #### 🎇执行初始化命令，根据提示操作

  ```yaml
  # 初始化集群
  kubeadm init --config=kubeadm-conf.yaml
  
  # 如果执行到一般出错，需要进行回退
  # kubeadm reset
  # 以下为成功时的输出内容
  
  Your Kubernetes control-plane has initialized successfully!
  
  To start using your cluster, you need to run the following as a regular user:
  
  # 这个要执行一下
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
  
  Alternatively, if you are the root user, you can run:
  
    export KUBECONFIG=/etc/kubernetes/admin.conf
  
  You should now deploy a pod network to the cluster.
  Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
    https://kubernetes.io/docs/concepts/cluster-administration/addons/
  
  You can now join any number of control-plane nodes by copying certificate authorities
  and service account keys on each node and then running the following as root:
  
    kubeadm join 10.10.201.28:6443 --token abcdef.0123456789abcdef \
          --discovery-token-ca-cert-hash sha256:aff1e1ce9aed25bd57c6a7aceaa802cb3e72f5567d8eba466c2238cd76349004 \
          --control-plane
  
  Then you can join any number of worker nodes by running the following on each as root:
  
  kubeadm join 10.10.201.28:6443 --token abcdef.0123456789abcdef \
          --discovery-token-ca-cert-hash sha256:aff1e1ce9aed25bd57c6a7aceaa802cb3e72f5567d8eba466c2238cd76349004
  ```

#### 扩容节点

新节点按步骤执行到启动kubelet

> 区分控制节点与负载节点在于join命令的参数--control-plane，带上为控制节点

执行kubeadm init 最后输出的加入命令

```bash
kubeadm join --token abcdef.1234567890abcdef --discovery-token-ca-cert-hash <token-ca-cert-hash>
kubeadm join --discovery-token-ca-cert-hash sha256:1234..cdef <control-plane-ip>:6443
```

如果没有保存，可以用以下方式获取

```shell
# 获取加入命令
kubeadm token create --print-join-command

# 手动计算hash 得到 sha256:<hash>
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
<hash>
```

#### 🎑移除节点

* 停止调度

  ```shell
  kubectl uncordon node <node-to-remove>
  ```
* 排空节点

  ```shell
  kubectl drain node <node-to-remove>
  ```
* 删除节点

  ```shell
  kubectl delete node <node-to-remove>
  ```
* 节点重置(在要移除的节点上操作)

  ```shell
  kubeadm reset
  ```

‍

## 🎛配置网络插件(必要)

> 此时core-dns 异常，节点notReady, 安装任一网络插件后即可正常使用

### Calico

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.4/manifests/custom-resources.yaml
```
