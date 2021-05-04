kubeadm是官方社区推出的一个用于快速部署kubernetes集群的工具。

这个工具能通过两条指令完成一个kubernetes集群的部署：

```
# 创建一个 Master 节点
$ kubeadm init

# 将一个 Node 节点加入到当前集群中
$ kubeadm join <Master节点的IP和端口 >
```

## 1. 安装要求

在开始之前，部署Kubernetes集群机器需要满足以下几个条件：

- 一台或多台机器，操作系统 CentOS-8-x86_64-1905-dvd1.iso
- 硬件配置：4GB或更多RAM，2个CPU或更多CPU，硬盘30GB或更多
- 可以访问外网，需要拉取镜像，如果服务器不能上网，需要提前下载镜像并导入节点
- 禁止swap分区
- 安装参考: https://github.com/Mountains-and-rivers/k8s-code-analysis/blob/master/01-%E6%90%AD%E5%BB%BA%E8%B0%83%E8%AF%95%E7%8E%AF%E5%A2%83.md

## 2. 准备环境

| 角色   | IP             |
| ------ | -------------- |
| master | 192.168.31.209 |
| node01  | 192.168.31.240 |
| node02  | 192.168.31.223 |

```
# 关闭防火墙
systemctl stop firewalld
systemctl disable firewalld

# 关闭selinux
setenforce 0  # 临时
sed -i 's/enforcing/disabled/' /etc/selinux/config  # 永久

# 关闭swap
swapoff -a  # 临时
sed -ri 's/.*swap.*/#&/' /etc/fstab    # 永久

# 根据规划设置主机名
hostnamectl set-hostname <hostname>

# 在master添加hosts
cat >> /etc/hosts << EOF
192.168.31.209 master
192.168.31.240 node01
192.168.31.223 node02
EOF

# 将桥接的IPv4流量传递到iptables的链
cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system  # 生效

# 时间同步
rpm -ivh http://mirrors.wlnmp.com/centos/wlnmp-release-centos.noarch.rpm

yum install wntp -y

ntpdate ntp1.aliyun.com
```

## 3. 所有节点安装Docker/kubeadm/kubelet

Kubernetes默认CRI（容器运行时）为Docker，因此先安装Docker。

### 3.1 安装Docker

```
sudo yum install -y yum-utils   device-mapper-persistent-data  lvm2

sudo yum-config-manager  --add-repo  https://download.docker.com/linux/centos/docker-ce.repo
wget https://download.docker.com/linux/centos/7/x86_64/edge/Packages/containerd.io-1.2.6-3.3.el7.x86_64.rpm

yum install -y containerd.io-1.2.6-3.3.el7.x86_64.rpm
sudo yum install docker-ce-19.03.8 docker-ce-cli-19.03.8 containerd.io
sudo dnf update -y
dnf clean packages
sudo yum install docker-ce docker-ce-cli containerd.io -y
systemctl start docker
systemctl enable docker
```

```
$ cat > /etc/docker/daemon.json << EOF
{
  "registry-mirrors": ["https://b9pmyelo.mirror.aliyuncs.com"]
}
EOF
```

### 3.2 添加阿里云YUM软件源

```
$ cat > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

### 3.3 安装kubeadm，kubelet和kubectl

由于版本更新频繁，这里指定版本号部署：

```
yum install -y kubelet-1.20.0 kubeadm-1.20.0 kubectl-1.20.0
systemctl enable kubelet
```

## 4. 部署Kubernetes Master

在192.168.31.209（Master）执行。

```
kubeadm init --apiserver-advertise-address=192.168.31.209 --service-cidr=10.96.0.0/12  --pod-network-cidr=10.244.0.0/16 \

```

由于默认拉取镜像地址k8s.gcr.io国内无法访问，这里指定阿里云镜像仓库地址。

使用kubectl工具：

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
$ kubectl get nodes
```

注意：该文件需要copy到每个节点下的$HOME/.kube目录

## 5. 加入Kubernetes Node

在192.168.31.209/23（node01） 192.168.31.223/23（node02）执行。

向集群添加新节点，执行在kubeadm init输出的kubeadm join命令：

```
$ kubeadm join 192.168.31.209:6443 --token esce21.q6hetwm8si29qxwn \
    --discovery-token-ca-cert-hash sha256:00603a05805807501d7181c3d60b478788408cfe6cedefedb1f97569708be9c5
```

默认token有效期为24小时，当过期之后，该token就不可用了。这时就需要重新创建token，操作如下：

```
kubeadm token create --print-join-command
```

## 6. 部署CNI网络插件

```
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

默认镜像地址无法访问，sed命令修改为docker hub镜像仓库。

```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

kubectl get pods -n kube-system
NAME                          READY   STATUS    RESTARTS   AGE
kube-flannel-ds-amd64-2pc95   1/1     Running   0          72s
```

## 7. 测试kubernetes集群

在Kubernetes集群中创建一个pod，验证是否正常运行：

```
$ kubectl create deployment nginx --image=nginx
$ kubectl expose deployment nginx --port=80 --type=NodePort
$ kubectl get pod,svc
```

访问地址：http://NodeIP:Port  

# 8. 搭建nfs

centos8 系统nfs 叫nfs server

```
systemctl enable nfs-server
systemctl start nfs-server
vim /etc/exports # 添加以下内容
/mnt/mongo *(rw,sync,insecure,no_subtree_check,no_root_squash)

exports -r #使配置生效
```

# 9. 给node01 node02 打标签

在master 节点执行(我只有三台机器，请自行添加节点选择器)

```
kubectl label nodes node01 mongo=mongo
kubectl label nodes node02 mongo=mongo
```

# 10. 部署mongodb

```
kubectl apply -f mongo-replica-set/mongo-hostPath/123/mongo.yaml #其他文件为测试文件，可忽略
```



# 11. FAQ

需要科学上网

```
方式1： 小米路由器插件安装 参考：https://github.com/MrSeaning/MIXBOX
然后购买 https://shadowflys.us/ 的服务，手动安装配置shadowsocks
方式2: 购买 https://shadowflys.us/ 下载官方插件配置
```

镜像已经分片上传到仓库目录

mongo：4.0.12 在mongo:4.0.12 目录

```
执行 cat xa* > mongo.tar
```

cvallance/mongo-k8s-sidecar在cvallance目录

```
执行 cat xa* > cvallance.tar
```



nfs 持久化部署在mongo-nfs目录

```
kubectl apply -f . 
```



# 12. 扩展

```
外部访问 mongodb服务以ingress的方式 小伙伴们可以自行探索
```

```
由于镜像为官方提供，有时需要自己编译mongo，比如在arm处理器
```

镜像编译参考官方

mongo 

```
FROM centos:latest
MAINTAINER https://blog.csdn.net/lituxiu
ENV TIME_ZOME Asia/Shanghai
ARG WJ="mongodb-linux-x86_64-rhel70-4.0.8"
#https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-rhel70-4.0.8.tgz
 
ADD $WJ.tgz /usr/local/
RUN mkdir -p /usr/local/mongodb \
        &&   cp -r /usr/local/$WJ/*  /usr/local/mongodb/  \
        && echo "export MONGODB_HOME=/usr/local/mongodb " >>/etc/profile \
        && echo "export PATH=$PATH:$MONGODB_HOME/bin" >>/etc/profile \
        && source /etc/profile \
 
        && cd /usr/local/mongodb \
        && mkdir -p data/db \
        && chmod -R 777 data/db \
        && mkdir logs \
        && touch mongodb.log  \
 
        && echo "${TIME_ZOME}" > /etc/timezone \
        && ln -sf /usr/share/zoneinfo/${TIME_ZOME} /etc/localtime \
        && rm -rf /usr/local/$WJ/
    
#WORKDIR /usr/local/mongodb/bin  #这里要靠近下面的ENTRYPOINT,否则报错找不到./mogod命令
RUN mkdir /usr/local/mongodb/etc
COPY mongodb.conf /usr/local/mongodb/etc/
EXPOSE 27017
#启动
WORKDIR /usr/local/mongodb/bin/

ENTRYPOINT ./mongod -f ../etc/mongodb.conf
```

mongo-k8s-sidecar

https://github.com/cvallance/mongo-k8s-sidecar

```
FROM node:alpine
MAINTAINER Charles Vallance <vallance.charles@gmail.com>

WORKDIR /opt/cvallance/mongo-k8s-sidecar

COPY package.json /opt/cvallance/mongo-k8s-sidecar/package.json

RUN npm install

COPY ./src /opt/cvallance/mongo-k8s-sidecar/src
COPY .foreverignore /opt/cvallance/.foreverignore

CMD ["npm", "start"]
```

