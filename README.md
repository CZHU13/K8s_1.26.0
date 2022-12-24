# kubeadm极速部署Kubernetes 1.26版本集群

# 一、Kubernetes 1.26版本集群部署

## 1.1 Kubernetes 1.26版本集群部署环境准备

### 1.1.1 主机操作系统说明

| 序号 | 操作系统及版本 | 备注 |
| :--: | :------------: | :--: |
|  1   |   CentOS7u9    |      |



### 1.1.2 主机硬件配置说明

| 需求 | CPU  | 内存 | 硬盘  | 角色         | 主机名      |
| ---- | ---- | ---- | ----- | ------------ |----------|
| 值   | 4C   | 8G   | 100GB | master       | master01 |
| 值   | 4C   | 8G   | 100GB | worker(node) | node01   |
| 值   | 4C   | 8G   | 100GB | worker(node) | node02   |



### 1.1.3 主机配置

#### 1.1.3.1  主机名配置

由于本次使用3台主机完成kubernetes集群部署，其中1台为master节点,名称为master01;其中2台为worker节点，名称分别为：node01及node02


#### 1.1.3.2 主机IP地址配置
| 角色      | IP                |
|---------|-------------------|
| master01 | 192.168.127.142   |
| node01  | 192.168.127.144   |
| node02  | 192.168.127.144   |


#### 1.1.3.3 主机名与IP地址解析

~~~bash
cat <<EOF >  /etc/hosts
#!/bin/bash
127.0.0.1  localhost localhost.localdomain localhost4 localhost4.localdomain4
::1        localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.127.142 master01
192.168.127.144 node01
192.168.127.145 node02
EOF
~~~

> 所有集群主机均需要进行配置。
#### 1.1.3.4  防火墙配置

> 所有主机均需要操作。
~~~bash
#关闭现有防火墙firewalld
systemctl disable firewalld
systemctl stop firewalld
firewall-cmd --state
~~~

#### 1.1.3.5 SELINUX配置
> 所有主机均需要操作。修改SELinux配置需要重启操作系统。
~~~bash
sed -ri 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
~~~

#### 1.1.3.6 时间同步配置
>所有主机均需要操作。最小化安装系统需要安装ntpdate软件。
~~~powershell
yum install ntpdate -y
# crontab -l
0 */1 * * * /usr/sbin/ntpdate time1.aliyun.com
~~~

#### 1.1.3.7 升级操作系统内核
> 所有主机均需要操作。
~~~bash
#导入elrepo gpg key
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
~~~

~~~bash
# 安装elrepo YUM源仓库
yum -y install https://www.elrepo.org/elrepo-release-7.0-4.el7.elrepo.noarch.rpm
~~~

~~~bash
# 安装kernel-ml版本，ml为长期稳定版本，lt为长期维护版本
yum --enablerepo="elrepo-kernel" -y install kernel-lt.x86_64
~~~

~~~bash
#设置grub2默认引导为0
grub2-set-default 0
~~~

~~~bash
# 重新生成grub2引导文件
grub2-mkconfig -o /boot/grub2/grub.cfg
~~~

~~~bash
# 更新后，需要重启，使用升级的内核生效。
reboot
~~~

~~~bash
# 重启后，需要验证内核是否为更新对应的版本
uname -r
~~~

#### 1.1.3.8  配置内核转发及网桥过滤

>所有主机均需要操作。

~~~bash
# 添加网桥过滤及内核转发配置文件
cat << EOF >> /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
vm.swappiness = 0
EOF

sysctl --system

# 加载br_netfilter模块
modprobe br_netfilter

# 查看是否加载
lsmod | grep br_netfilter
# br_netfilter           22256  0
# bridge                151336  1 br_netfilter
~~~

#### 1.1.3.9 安装ipset及ipvsadm

> 所有主机均需要操作。
~~~bash
# 安装ipset及ipvsadm
yum -y install ipset ipvsadm


# 配置ipvsadm模块加载方式
# 添加需要加载的模块
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack
EOF

# 授权、运行、检查是否加载
chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack
~~~



#### 1.1.3.10 关闭SWAP分区
> 修改完成后需要重启操作系统，如不重启，可临时关闭，命令为swapoff -a
~~~bash
#临时关闭swap分区
swapoff -a
#命令永久关闭swap分区 需要重启Linux服务器
sed -ri 's/.*swap.*/#&/' /etc/fstab
~~~


## 1.2  Docker准备
### 1.2.1 Docker安装YUM源准备
>使用阿里云开源软件镜像站。
~~~bash
yum install -y wget
wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo
~~~

### 1.2.2 Docker安装
~~~bash
yum -y install docker-ce
~~~
### 1.2.3 启动Docker服务
~~~bash
systemctl enable --now docker
~~~

### 1.2.4 修改cgroup方式
>/etc/docker/daemon.json 默认没有此文件，需要单独创建
~~~bash
# 在/etc/docker/daemon.json添加如下内容
cat << /etc/docker/daemon.json >> EOF
{
        "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF
~~~

~~~bash
systemctl restart docker
~~~

### 1.2.5 cri-dockerd安装

#### 1.2.5.1 golang环境准备

~~~bash
wget https://storage.googleapis.com/golang/getgo/installer_linux
chmod +x ./installer_linux
./installer_linux
source ~/.bash_profile
~~~

#### 2.2.5.2 构建并安装cri-dockerd

~~~bash
# 克隆cri-dockerd源码
git clone https://github.com/Mirantis/cri-dockerd.git
cd cri-dockerd
# 创建bin目录并构建cri-dockerd二进制文件
mkdir bin
go get && go build -o ../bin/cri-dockerd
# 创建/usr/local/bin,默认存在时，可不用创建
mkdir -p /usr/local/bin
# 安装cri-dockerd
install -o root -g root -m 0755 bin/cri-dockerd /usr/local/bin/cri-dockerd
# 复制服务管理文件至/etc/systemd/system目录中
cp -a packaging/systemd/* /etc/systemd/system
#指定cri-dockerd运行位置
sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service
# 启动服务
systemctl daemon-reload
systemctl enable cri-docker.service
systemctl enable --now cri-docker.socket
~~~


## 1.3 kubernetes 1.26.X  集群部署

### 1.3.1  集群软件及版本说明

|          | kubeadm     | kubelet                       | kubectl     |
| -------- |-------------|-------------------------------|-------------|
| 版本     | 1.26.X      | 1.26.X                        | 1.26.X      |
| 安装位置 | 集群所有主机      | 集群所有主机                        | 集群所有主机      |
| 作用     | 初始化集群、管理集群等 | 用于接收api-server指令，对pod生命周期进行管理 | 集群应用命令行管理工具 |



### 1.3.2  kubernetes YUM源准备
#### 1.3.2.1 谷歌YUM源
~~~bash
cat <<EOF >  /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
        https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
~~~
#### 1.3.2.2 阿里云YUM源
~~~bash
cat <<EOF >  /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
~~~

### 1.3.3 集群软件安装
> 所有节点均可安装
~~~bash
# 默认安装
yum -y install  kubeadm  kubelet kubectl
~~~

~~~bash
查看指定版本
yum list kubeadm.x86_64 --showduplicates | sort -r
yum list kubelet.x86_64 --showduplicates | sort -r
yum list kubectl.x86_64 --showduplicates | sort -r
~~~

~~~bash
# 安装指定版本
yum -y install  kubeadm-1.26.X-0  kubelet-1.26.X-0 kubectl-1.26.X-0
~~~

### 1.3.4 配置kubelet
>为了实现docker使用的cgroupdriver与kubelet使用的cgroup的一致性，建议修改如下文件内容。
~~~bash
vi /etc/sysconfig/kubelet
# KUBELET_EXTRA_ARGS="--cgroup-driver=systemd"
~~~

~~~bash
# 设置kubelet为开机自启动即可，由于没有生成配置文件，集群初始化后自动启动
systemctl enable kubelet
~~~

### 1.3.5  集群镜像准备

> 可使用VPN实现下载。

~~~bash
kubeadm config images list --kubernetes-version=v1.26.0
~~~

~~~bash
images=(
        registry.k8s.io/kube-apiserver:v1.26.0
        registry.k8s.io/kube-controller-manager:v1.26.0
        registry.k8s.io/kube-scheduler:v1.26.0
        registry.k8s.io/kube-proxy:v1.26.0
        registry.k8s.io/pause:3.9
        registry.k8s.io/etcd:3.5.6-0
        registry.k8s.io/coredns/coredns:v1.9.3
)
for imageName in "${images[@]}" ; do
	docker pull $imageName
done
~~~

### 1.3.6 集群初始化

~~~bash
# master节点执行
kubeadm init --kubernetes-version=v1.26.0 --pod-network-cidr=10.224.0.0/16 --apiserver-advertise-address=192.168.127.142  --cri-socket unix:///var/run/cri-dockerd.sock

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
~~~

如果不添加--cri-socket选项，则会报错，内容如下：  
Found multiple CRI endpoints on the host. Please define which one do you wish to use by setting the 'criSocket' field in the kubeadm configuration file: unix:///var/run/containerd/containerd.sock, unix:///var/run/cri-dockerd.sock
To see the stack trace of this error execute with --v=5 or higher

```bash
kubeadm join 192.168.127.142:6443 --token c12af7.eccv488cyew0unrl \
--discovery-token-ca-cert-hash sha256:6383e5198b6d054d3e3fd5a972a5dd2e024e2853decde14fcd690e3cb7c48f2d --cri-socket unix:///var/run/cri-dockerd.sock
```


### 1.3.7  集群应用客户端管理集群文件准备
> master01执行
~~~bash
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
ls /root/.kube/
config
~~~

~~~bash
[root@k8s-master01 ~]# export KUBECONFIG=/etc/kubernetes/admin.conf
~~~

### 1.3.8  集群网络准备

> 使用calico部署集群网络
>
> 安装参考网址：https://projectcalico.docs.tigera.io/about/about-calico


#### 1.3.8.1  calico安装
> 使用calico部署集群网络  
> 安装参考网址：https://projectcalico.docs.tigera.io/about/about-calico  
> master01执行

```shell
# 下载operator资源清单文件
wget https://raw.githubusercontent.com/projectcalico/calico/v3.24.5/manifests/tigera-operator.yaml
# 应用资源清单文件，创建operator
kubectl create -f tigera-operator.yaml
# 通过自定义资源方式安装
wget https://raw.githubusercontent.com/projectcalico/calico/v3.24.5/manifests/custom-resources.yaml
# 修改文件第13行，修改为使用kubeadm init ----pod-network-cidr对应的IP地址段  cidr: 10.224.0.0/16
#  当node无法正常运行时，可考虑在此文件中添加相关内容。    
#      nodeAddressAutodetectionV4:
#        interface: ens.*

# 应用资源清单文件
kubectl create -f custom-resources.yaml
```

~~~bash
# 监视calico-sysem命名空间中pod运行情况
watch kubectl get pods -n calico-system
#NAME                                       READY   STATUS    RESTARTS   AGE
#calico-kube-controllers-67df98bdc8-2j82t   1/1     Running   0          2m28s
#calico-node-2lrsb                          1/1     Running   0          2m29s
#calico-node-8fcds                          1/1     Running   0          2m29s
#calico-node-wbswq                          1/1     Running   0          2m29s
#calico-typha-5c7b6b55b-hb6z2               1/1     Running   0          2m20s
#calico-typha-5c7b6b55b-njpjt               1/1     Running   0          2m29s
~~~

~~~bash
# 查看kube-system命名空间中coredns状态，处于Running状态表明联网成功。
kubectl get pods -n kube-system
NAME                                   READY   STATUS    RESTARTS   AGE
coredns-6d4b75cb6d-js5pl               1/1     Running   0          12h
coredns-6d4b75cb6d-zm8pc               1/1     Running   0          12h
etcd-k8s-master01                      1/1     Running   0          12h
kube-apiserver-k8s-master01            1/1     Running   0          12h
kube-controller-manager-k8s-master01   1/1     Running   0          12h
kube-proxy-7nhr7                       1/1     Running   0          12h
kube-proxy-fv4kr                       1/1     Running   0          12h
kube-proxy-vv5vg                       1/1     Running   0          12h
kube-scheduler-k8s-master01            1/1     Running   0          12h
~~~


#### 2.3.8.2  calico客户端安装

~~~bash
# 下载二进制文件
curl -L https://github.com/projectcalico/calico/releases/download/v3.21.4/calicoctl-linux-amd64 -o calicoctl
~~~

~~~bash
# 安装calicoctl
mv calicoctl /usr/bin/

# 为calicoctl添加可执行权限
chmod +x /usr/bin/calicoctl

# 查看添加权限后文件
ls /usr/bin/calicoctl

# 查看calicoctl版本
calicoctl  version
# Client Version:    v3.21.4
# Git commit:        220d04c94
# Cluster Version:   v3.21.4
# Cluster Type:      typha,kdd,k8s,operator,bgp,kubeadm
~~~

~~~bash
# 通过~/.kube/config连接kubernetes集群，查看已运行节点
DATASTORE_TYPE=kubernetes KUBECONFIG=~/.kube/config calicoctl get nodes
NAME
master01
~~~


### 1.3.9  集群工作节点添加
> 因容器镜像下载较慢，可能会导致报错，主要错误为没有准备好cni（集群网络插件），如有网络，请耐心等待即可。
~~~bash
kubeadm join 192.168.127.142:6443 --token c12af7.eccv488cyew0unrl \
--discovery-token-ca-cert-hash sha256:6383e5198b6d054d3e3fd5a972a5dd2e024e2853decde14fcd690e3cb7c48f2d --cri-socket unix:///var/run/cri-dockerd.sock
~~~

~~~bash
# 在master节点上操作，查看网络节点是否添加
DATASTORE_TYPE=kubernetes KUBECONFIG=~/.kube/config calicoctl get nodes
~~~
NAME  
master01  
node01  
node02  

# 二、 验证集群可用性

~~~bash
# 查看所有的节点
kubectl get nodes
~~~
NAME           STATUS   ROLES           AGE   VERSION  
k8s-master01   Ready    control-plane   12h   v1.26.0  
k8s-worker01   Ready    <none>          12h   v1.26.0  
k8s-worker02   Ready    <none>          12h   v1.26.0  


~~~bash
# 查看集群健康情况
kubectl get cs
~~~
Warning: v1 ComponentStatus is deprecated in v1.19+  
NAME                 STATUS    MESSAGE                         ERROR  
controller-manager   Healthy   ok  
scheduler            Healthy   ok  
etcd-0               Healthy   {"health":"true","reason":""}  



~~~bash
# 查看kubernetes集群pod运行情况
kubectl get pods -n kube-system
~~~
NAME                                   READY   STATUS    RESTARTS   AGE  
coredns-6d4b75cb6d-js5pl               1/1     Running   0          12h  
coredns-6d4b75cb6d-zm8pc               1/1     Running   0          12h  
etcd-k8s-master01                      1/1     Running   0          12h  
kube-apiserver-k8s-master01            1/1     Running   0          12h  
kube-controller-manager-k8s-master01   1/1     Running   0          12h  
kube-proxy-7nhr7                       1/1     Running   0          12h  
kube-proxy-fv4kr                       1/1     Running   0          12h  
kube-proxy-vv5vg                       1/1     Running   0          12h  
kube-scheduler-k8s-master01            1/1     Running   0          12h  



~~~bash
# 再次查看calico-system命名空间中pod运行情况。
kubectl get pods -n calico-system
~~~
NAME                                       READY   STATUS    RESTARTS   AGE  
calico-kube-controllers-5b544d9b48-xgfnk   1/1     Running   0          12h  
calico-node-7clf4                          1/1     Running   0          12h  
calico-node-cjwns                          1/1     Running   0          12h  
calico-node-hhr4n                          1/1     Running   0          12h  
calico-typha-6cb6976b97-5lnpk              1/1     Running   0          12h  
calico-typha-6cb6976b97-9w9s8              1/1     Running   0          12h
