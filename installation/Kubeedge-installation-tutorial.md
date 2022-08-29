### 本地环境搭建

#### 软件包版本

- Kernel version: 3.10.0-1160.45.1.el7.x86_64
- Kubelet version: v1.18.0
- KubeEdge version： v1.7.0
- Golang version: go1.15.3 linux/amd64 for CentOS7.x-86_x64，go1.15.3 linux/arm64 for Raspberry PI
- Docker version: v19.03

##### 云服务器节点主机名称和ip地址

master 10.101.15.27

node 10.101.15.28-29

配置yum源便于后续软件安装

```
sudo yum install -y yum-utils
sudo yum-config-manager \
--add-repo \
http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

#### 安装docker

```
sudo yum install -y docker-ce-19.03 docker-ce-cli containerd.io
```

更改docker安装版本

```
yum downgrade --setopt=obsoletes=0 -y docker-ce-19.03.9-3.el7 docker-ce-cli-19.03.9-3.el7 containerd.io
systemctl start docker
查看docker版本
docker version
```

配置加速

```
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://82m9ar63.mirror.aliyuncs.com"],
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

基础环境准备

```
#各个机器设置自己的域名
hostnamectl set-hostname xxxx

# 将 SELinux 设置为 permissive 模式（相当于将其禁用）
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

#关闭swap
swapoff -a  
sed -ri 's/.*swap.*/#&/' /etc/fstab

#允许 iptables 检查桥接流量
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```

安装kubelet、kubeadm、kubectl

```
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
   http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF


sudo yum install -y kubelet-1.18.0 kubeadm-1.18.0 kubectl-1.18.0 --disableexcludes=kubernetes

sudo systemctl enable --now kubelet
```

#### 启动k8s-1.18.0

初始化k8s集群主节点

```
kubeadm init \
--apiserver-advertise-address=10.101.15.27 \
--image-repository registry.aliyuncs.com/google_containers \
--kubernetes-version v1.18.0 \
--service-cidr=10.96.0.0/16 \
--pod-network-cidr=192.168.0.0/16
```

加入新control-plane或worker节点

![image-20220819203142004](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20220819203142004.png)

```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of control-plane nodes by copying certificate authorities
and service account keys on each node and then running the following as root:

  kubeadm join cluster-endpoint:6443 --token jky2l2.uv08pw10n5iqzhzr \
    --discovery-token-ca-cert-hash sha256:f71e50b2176bed7db64140f1fedf50d2234f3811e7ca8f70745d6f91e2237eed \
    --control-plane 

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join cluster-endpoint:6443 --token jky2l2.uv08pw10n5iqzhzr \
    --discovery-token-ca-cert-hash sha256:f71e50b2176bed7db64140f1fedf50d2234f3811e7ca8f70745d6f91e2237eed 
```

设置kubectl配置文件

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

配置kubectl（在kubadm reset后）

```
rm -rf $HOME/.kube
```

下载flannel网络组件的yaml文件并安装

```
cd ~ && mkdir flannel && cd flannel
curl -O https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

卸载kubelet

```
kubeadm reset
yum erase -y kubelet kubectl kubeadm kubernetes-cni
```

kubelet.service启动失败问题

```
简单地说就是在kubeadm init 之前kubelet会不断重启。
```

#### 安装Kubeedge

KubeEdge由云和边缘组成。它建立在Kubernetes之上，为联网、应用部署和云与边缘之间的元数据同步提供核心基础设施支持。所以如果我们想要设置KubeEdge，我们需要设置Kubernetes集群(可以使用现有的集群)，云端和边缘端。

- 在cloud side, 需要安装
  -  Docker,
  -  Kubernetes cluster
  -  cloudcore.
- 在 edge side, 需要安装
  -  Docker,
  -  MQTT (We can also use internal MQTT broker) （配置可以选用，不是一定需要）
  -  edgecore.

首先在master节点和edge01节点安装golang

```
yum install wget -y
wget https://golang.google.cn/dl/go1.15.3.linux-amd64.tar.gz
tar zxvf go1.15.3.linux-amd64.tar.gz
mv go /usr/local/

vim /etc/profile在最结尾添加

export HOME=/root
export GOROOT=/usr/local/go
export GOPATH=/opt/idcus/go
export PATH=$PATH:$GOPATH/bin:$GOROOT/bin

source /etc/profile
go version #查看是否生效
```

##### Cloud端安装

安装有两种方式，一种源码编译手动安装，还有一种是使用kubeedge提供的工具-keadm。

下载keadm1.7.0

```
wget https://github.com/kubeedge/kubeedge/releases/download/v1.7.0/keadm-v1.7.0-linux-amd64.tar.gz
```

配置环境变量才能全局使用keadm命令-执行

```
tar -zxvf keadm-v1.7.0-linux-amd64.tar.gz # 解压keadm的tar.gz的包
cd keadm-v1.7.0-linux-amd64/keadm/
cp keadm /usr/sbin/ #将其配置进入环境变量，方便使用
```

使用keadm安装cloudcore，获取token

–advertise-address指的是master机器的ip,可以是内网地址，也可以是公网ip地址，–kubeedge-version=1.7.0 意思是指定安装的kubeEdge的版本，默认下载最新版本。

```
keadm init --advertise-address=10.101.15.27 --kubeedge-version=1.7.0
keadm gettoken
```

![image-20220820214500209](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20220820214500209.png)

![image-20220818220519996](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20220818220519996.png)

##### Edge端安装

```
keadm join --cloudcore-ipport=10.101.15.27:10000 --edgenode-name=edge01 --kubeedge-version=1.7.0 --token=af7cbe344c93a6540c39be768ea75eecf20e385ed9e6afd2dd7271f55e2cd8c9.eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE2NjEyNjY4Mjd9.7XI59QCrSJC4usVo5tzhinETAr7al1kM7s4OkZYjRjo

systemctl status edgecore.service
```

出现链接错误时 删除edgecore.service文件

```
rm -rf /etc/systemd/system/edgecore.service
```

若有问题，[参考链接](https://github.com/kubeedge/kubeedge/issues/2847)

加入CloudCore

![image-20220818225435365](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20220818225435365.png)

从master节点查看是否加入成功

```
kubectl get nodes
```

![image-20220820222549725](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20220820222549725.png)

raspberry节点加入过程同上



### 基于KubeEdge的Counter Demo 计数器应用搭建与访问

KubeEdge Counter Demo 计数器是一个伪设备，用户无需任何额外的物理设备即可运行此演示。计数器在边缘侧运行，用户可以从云侧在 Web 中对其进行控制，也可以从云侧在 Web 中获得计数器值,原理图如下:

![在这里插入图片描述](https://img-blog.csdnimg.cn/0b9be252c49449b68ce48fe6bb86212d.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lpbmpsMTIzNDU2,size_16,color_FFFFFF,t_70#pic_center)

#### 云端操作

创建模型和设备

```
#拉取代码
git clone https://github.com/kubeedge/examples.git $GOPATH/src/github.com/kubeedge/examples 
#创建model
cd $GOPATH/src/github.com/kubeedge/examples/kubeedge-counter-demo/crds

kubectl create -f kubeedge-counter-model.yaml

#创建device
cd $GOPATH/src/github.com/kubeedge/examples/kubeedge-counter-demo/crds
kubectl create -f kubeedge-counter-instance.yaml
```

![image-20220822130754433](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20220822130754433.png)

##### 部署云端应用

云端应用web-controller-app用来控制边缘端的pi-counter-app应用，修改端口为8089

```
cd $GOPATH/src/github.com/kubeedge/examples/kubeedge-counter-demo/web-controller-app
vim main.go
beego.Run(":8089")
make all
make docker
```

构建镜像

-makefile

-dockerfile

![image-20220822131711565](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20220822131711565.png)

部署云端应用web-controller-app

```
kubectl apply -f kubeedge-web-controller-app.yaml
```

![image-20220822140658372](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20220822140658372.png)

##### 部署边缘端应用

边缘端的pi-counter-app应用受云端应用控制，主要与mqtt服务器通信，进行简单的计数功能。

构建镜像

```
cd $GOPATH/src/github.com/kubeedge/examples/kubeedge-counter-demo/counter-mapper
#构建镜像，执行make文件
make all
make docker
```

![image-20220822135918707](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20220822135918707.png)

部署Pi Counter App

```
cd $GOPATH/src/github.com/kubeedge/examples/kubeedge-counter-demo/crds
kubectl apply -f kubeedge-pi-counter-app.yaml
```

边缘端Pi Counter App运行结果

![image-20220822140353096](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20220822140353096.png)

至此，KubeEdge Demo的云端部分和边缘端的部分部署完成

![image-20220822232246755](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20220822232246755.png)

#### 访问应用

因为使用的 hostNetwork 模式，所以直接访问即可,http://10.101.15.27:8089/

<img src="C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20220822232350573.png" alt="image-20220822232350573" style="zoom: 80%;" />

在web页面上选择ON，并点击Execute，可以在edge边缘节点上通过以下命令查看执行结果

```
docker logs -f 863cfe11ba22
```

找到container id 

![image-20220822232620138](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20220822232620138.png)

观察计数执行结果

![image-20220822232841708](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20220822232841708.png)

表示测试成功

###  KubeEdge WeChat Demo

![img](https://gitee.com/liu_hu_wei/examples/raw/master/kubeedge-wechat-demo/workflow.png)

#### 参考资料

- https://www.yuque.com/leifengyang/oncloud/ghnb83#6lMCM
- https://gitee.com/liu_hu_wei/examples
- https://www.cnblogs.com/ltaodream/p/15135365.html
- https://blog.csdn.net/weixin_38159695/article/details/118486461
- https://www.cnblogs.com/cptao/p/10912709.html
- https://blog.csdn.net/yinjl123456/article/details/119300034
- https://github.com/kubeedge
- https://github.com/kubeedge/kubeedge/releases/

### 分析

**问题1：**`kubeadm reset`后`kubeadm init`出现的问题

```
# kubectl get node
Unable to connect to the server: x509: certificate signed by unknown authority (possibly because of "crypto/rsa: verification error" while trying to verify candidate authority certificate "kubernetes")`
```

问题1解决办法：

I got the same error while running $ kubectl get nodes as a root user. I fixed it by exporting kubelet.conf to environment variable.

```
export KUBECONFIG=/etc/kubernetes/kubelet.conf
kubectl get nodes
```



