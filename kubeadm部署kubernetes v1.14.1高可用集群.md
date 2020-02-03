# kubeadm部署kubernetes v1.14.1高可用集群

[toc]

## 第一章 高可用简介

**高可用部署参考**

``` bash
https://www.cnblogs.com/sandshell/p/11570458.html#auto_id_26
https://kubernetes.io/docs/setup/independent/high-availability/
https://github.com/kubernetes-sigs/kubespray
https://github.com/wise2c-devops/breeze
https://github.com/cookeem/kubeadm-ha**
```

### 1.1 拓扑选择
** 配置高可用(HA) kubernetes集群，有一下两种可选的etcd拓扑: **
* 集群master节点与etcd节点共存，etcd也运行在控制平面节点上
* 外部使用etcd节点，etcd节点与master在不同节点上运行

#### 1.1.1 堆叠的etcd拓扑

* 堆叠HA集群是这样的拓扑，其中etcd提供的分布式数据存储集群与由kubeamd管理的运行master组件的集群节点堆叠部署。
* 每个master节点运行kube-apiserver，kube-scheduler和kube-controller-manager的一个实例。kube-apiserver使用负载平衡器暴露给工作节点
* 每个master节点创建一个本地etcd成员，该etcd成员仅与本节点kube-apiserver通信。这同样适用于本地kube-controller-manager 和kube-scheduler实例。
* 该拓扑将master和etcd成员耦合在相同节点上。比设置具有外部etcd节点的集群更简单，并且更易于管理复制。
* 但是，堆叠集群存在耦合失败的风险。如果一个节点发生故障，则etcd成员和master实例都将丢失，并且冗余会受到影响。您可以通过添加更多master节点来降低此风险

因此，您应该为HA群集运行至少三个堆叠的master节点。
这是kubeadm中的默认拓扑。使用kubeadm init和kubeadm join --experimental-control-plane命令时，在master节点上自动创建本地etcd成员。 **
![](https://img2018.cnblogs.com/blog/1178573/201910/1178573-20191009153450188-287987290.png)

#### 1.1.2 外部etcd拓扑

* 具有外部etcd的HA集群是这样的拓扑，其中由etcd提供的分布式数据存储集群部署在运行master组件的节点形成的集群外部。
* 像堆叠ETCD拓扑结构，在外部ETCD拓扑中的每个master节点运行一个kube-apiserver，kube-scheduler和kube-controller-manager实例。并且kube-apiserver使用负载平衡器暴露给工作节点。但是，etcd成员在不同的主机上运行，每个etcd主机与kube-apiserver每个master节点进行通信。
* 此拓扑将master节点和etcd成员分离。因此，它提供了HA设置，其中丢失master实例或etcd成员具有较小的影响并且不像堆叠的HA拓扑那样影响集群冗余。
但是，此拓扑需要两倍于堆叠HA拓扑的主机数。具有此拓扑的HA群集至少需要三个用于master节点的主机和三个用于etcd节点的主机。  **
![](https://img2018.cnblogs.com/blog/1178573/201910/1178573-20191009153606297-230885839.png)

### 1.2负载均衡

* 部署集群前首选需要为kube-apiserver创建负载均衡器
* 注意：负载平衡器有许多中配置方式。可以根据你的集群要求选择不同的配置方案。在云环境中，您应将master节点作为负载平衡器TCP转发的后端。此负载平衡器将流量分配到其目标列表中的所有健康master节点。apiserver的运行状况检查是对kube-apiserver侦听的端口的TCP检查（默认值:6443）。
* 负载均衡器必须能够与apiserver端口上的所有master节点通信。它还必须允许其侦听端口上的传入流量。另外确保负载均衡器的地址始终与kubeadm的ControlPlaneEndpoint地址匹配。
* haproxy/nignx+keepalived是其中可选的负载均衡方案，针对公有云环境可以直接使用运营商提供的负载均衡产品。
部署时首先将第一个master节点添加到负载均衡器并使用以下命令测试连接： **

``` bash
nc -v LOAD_BALANCER_IP PORT  #  nc -v 122.114.139.5 6443
```

### 第2章 部署集群


#### 2.1 基本配置

* 节点信息

| 主机名 | ip地址  | 角色 |
|--------|--------|------|
|k8s-master01| 122.114.139.3  | master|
|k8s-master02|122.114.139.5|master|
|k8s-master03|122.114.139.22|master|
|k8s-node01|122.114.139.23|node|
|k8s-node02|122.114.139.25|node|
|k8s-vip|122.114.139.33|vip|


* 以下操作在所有节点执行

``` bash
#配置主机名
hostnamectl set-hostname k8s-master01
hostnamectl set-hostname k8s-master02
hostnamectl set-hostname k8s-master03
hostnamectl set-hostname k8s-node01
hostnamectl set-hostname k8s-node02

#修改/etc/hosts
cat >> /etc/hosts << EOF
122.114.139.3 k8s-master01
122.114.139.5 k8s-master02
122.114.139.22 k8s-master03
122.114.139.23 k8s-node01
122.114.139.23 k8s-node02
EOF

# 关闭防火墙
sed -i 's/GSSAPIAuthentication yes/GSSAPIAuthentication no/g' /etc/ssh/sshd_config
sed -i 's/#UseDNS yes/UseDNS no/g' /etc/ssh/sshd_config
systemctl restart sshd
systemctl stop postfix && systemctl disable postfix
systemctl stop NetworkManager && systemctl disable NetworkManager
systemctl stop firewalld && systemctl disable firewalld

#关闭swap
swapoff -a
yes | cp /etc/fstab /etc/fstab_bak
cat /etc/fstab_bak | grep -v swap > /etc/fstab

```

* 配置时间同步

``` bash
yum install -y chrony
cp /etc/chrony.conf{,.bak}
# 注释默认ntp服务器
sed -i 's/^server/#&/' /etc/chrony.conf
# 指定上游公共 ntp 服务器
cat >> /etc/chrony.conf << EOF
server 0.asia.pool.ntp.org iburst
server 1.asia.pool.ntp.org iburst
server 2.asia.pool.ntp.org iburst
server 3.asia.pool.ntp.org iburst
EOF
 
timedatectl set-timezone Asia/Shanghai
systemctl enable chronyd && systemctl restart chronyd
timedatectl && chronyc sources
```
* 加载IPVS模块

``` bash
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
EOF
#执行脚本
chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack_ipv4
#安装相关管理工具
yum install ipset ipvsadm -y
```

* 配置内核参数

``` bash
cat > /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_nonlocal_bind = 1
net.ipv4.ip_forward = 1
vm.swappiness=0
EOF
sysctl --system
```

#### 2.2 安装docker

* 以下操作在所有节点执行。

``` bash
# 安装依赖软件包
yum install -y yum-utils device-mapper-persistent-data lvm2
 
# 添加Docker repository，这里改为国内阿里云yum源
yum-config-manager \
--add-repo \
http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
 
# 安装docker-ce
yum install -y docker-ce-18.06.3.ce-3.el7
 
## 创建 /etc/docker 目录
mkdir /etc/docker
 
# 配置 daemon.
cat > /etc/docker/daemon.json <<EOF
{
  "registry-mirrors": ["https://uyah70su.mirror.aliyuncs.com"]
}
EOF
#注意，由于国内拉取镜像较慢，配置文件最后追加了阿里云镜像加速配置。
 
mkdir -p /etc/systemd/system/docker.service.d
 
# 重启docker服务
systemctl daemon-reload && systemctl restart docker && systemctl enable docker
```

#### 2.3 安装负载均衡

* kube-scheduler 和 kube-controller-manager 可以以集群模式运行，通过 leader 选举产生一个工作进程，其它进程处于阻塞模式。
* kube-apiserver可以运行多个实例，但对其它组件需要提供统一的访问地址，该地址需要高可用。本次部署使用 keepalived+haproxy 实现 kube-apiserver VIP 高可用和负载均衡。
* haproxy+keepalived配置vip，实现了api唯一的访问地址和负载均衡。keepalived 提供 kube-apiserver 对外服务的 VIP。haproxy 监听 VIP，后端连接所有 kube-apiserver 实例，提供健康检查和负载均衡功能。
* 运行 keepalived 和 haproxy 的节点称为 LB 节点。由于 keepalived 是一主多备运行模式，故至少两个 LB 节点。
* 本次部署复用 master 节点的三台机器，在所有3个master节点部署haproxy和keepalived组件，以达到更高的可用性，haproxy 监听的端口(6444) 需要与kube-apiserver的端口 6443 不同，避免冲突。
* keepalived 在运行过程中周期检查本机的 haproxy 进程状态，如果检测到 haproxy 进程异常，则触发重新选主的过程，VIP 将飘移到新选出来的主节点，从而实现 VIP 的高可用。
* 所有组件（如 kubeclt、apiserver、controller-manager、scheduler 等）都通过 VIP +haproxy 监听的6444端口访问 kube-apiserver 服务。**

* 负载均衡架构图如下：
![](https://img2018.cnblogs.com/blog/1178573/201910/1178573-20191009161152345-1412329739.png)


#### 2.4 运行HA容器

* 使用的容器镜像为睿云智合开源项目breeze相关镜像，具体使用方法请访问：
https://github.com/wise2c-devops
其他选择：haproxy镜像也可以使用dockerhub官方镜像，但keepalived未提供官方镜像，可自行构建或使用dockerhub他人已构建好的镜像，本次部署全部使用breeze提供的镜像。
在3个master节点以容器方式部署haproxy，容器暴露6444端口，负载均衡到后端3个apiserver的6443端口，3个节点haproxy配置文件相同。 **

* 以下操作在master01节点执行

##### 2.4.1 创建haproxy启动脚本

``` bash
mkdir -p /data/lb
cat > /data/lb/start-haproxy.sh << "EOF"
#!/bin/bash
MasterIP1=122.114.139.3
MasterIP2=122.114.139.5
MasterIP3=122.114.139.22
MasterPort=6443
 
docker run -d --restart=always --name HAProxy-K8S -p 6444:6444 \
-e MasterIP1=$MasterIP1 \
-e MasterIP2=$MasterIP2 \
-e MasterIP3=$MasterIP3 \
-e MasterPort=$MasterPort \
wise2c/haproxy-k8s
EOF
```

##### 2.4.2 创建keepalived启动脚本
``` bash
cat > /data/lb/start-keepalived.sh << "EOF"
#!/bin/bash
VIRTUAL_IP=122.114.139.33
INTERFACE=eth0
NETMASK_BIT=24
CHECK_PORT=6444
RID=10
VRID=160
MCAST_GROUP=224.0.0.18
 
docker run -itd --restart=always --name=Keepalived-K8S \
--net=host --cap-add=NET_ADMIN \
-e VIRTUAL_IP=$VIRTUAL_IP \
-e INTERFACE=$INTERFACE \
-e CHECK_PORT=$CHECK_PORT \
-e RID=$RID \
-e VRID=$VRID \
-e NETMASK_BIT=$NETMASK_BIT \
-e MCAST_GROUP=$MCAST_GROUP \
wise2c/keepalived-k8s
EOF
```

** 复制脚本到其它master节点 **
``` bash
[root@k8s-master02 ~]# mkdir -p /data/lb
[root@k8s-master03 ~]# mkdir -p /data/lb
[root@k8s-master01 ~]# scp start-haproxy.sh start-keepalived.sh 192.168.92.11:/data/lb/
[root@k8s-master01 ~]# scp start-haproxy.sh start-keepalived.sh 192.168.92.12:/data/lb/
```
** 分别在3个master节点运行脚本启动haproxy和keepalived容器： **
``` bash
sh /data/lb/start-haproxy.sh && sh /data/lb/start-keepalived.sh
```

##### 2.4.3 验证HA状态
* 查看容器状态

``` bash
[root@k8s-master01 ~]# docker ps
CONTAINER ID IMAGE COMMAND CREATED STATUS PORTS NAMES
c1d1901a7201 wise2c/haproxy-k8s "/docker-entrypoint.…" 5 days ago Up 3 hours 0.0.0.0:6444->6444/tcp HAProxy-K8S
2f02a9fde0be wise2c/keepalived-k8s "/usr/bin/keepalived…" 5 days ago Up 3 hours Keepalived-K8S
```

* 查看网卡绑定的vip 为122.114.139.33

``` bash
[root@k8s-master03 ~]# ip a|grep eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    inet 122.114.139.22/24 brd 122.114.139.255 scope global eth0
    inet 122.114.139.33/24 scope global secondary eth0
6: veth0525b5b@if5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default
```

* 查看监听的端口为6644

``` bash
[root@k8s-master03 ~]# netstat -tnlp | grep 6444
tcp6       0      0 :::6444                 :::*                    LISTEN      6764/docker-proxy
```

* keepalived配置文件中配置了vrrp_script脚本，使用nc命令对haproxy监听的6444端口进行检测，如果检测失败即认定本机haproxy进程异常，将vip漂移到其他节点。
* 所以无论本机keepalived容器异常或haproxy容器异常都会导致vip漂移到其他节点，可以停掉vip所在节点任意容器进行测试。 **

##### 2.4.4 haproxy和keepalived配置信息

** 关于haproxy和keepalived配置文件可以在github源文件中参考Dockerfile，或使用docker exec -it xxx sh命令进入容器查看，容器中的具体路径： **
* /etc/keepalived/keepalived.conf
* /usr/local/etc/haproxy/haproxy.cfg

#### 2.5 安装kubeadm

``` bash
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
 
#安装kubeadm、kubelet、kubectl,注意这里默认安装当前最新版本v1.14.1:
yum install -y kubeadm-1.14.1 kubelet-1.14.1 kubectl-1.14.1
systemctl enable kubelet && systemctl start kubelet
```

#### 2.6 初始化master节点

初始化参考：
https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/
https://godoc.org/k8s.io/kubernetes/cmd/kubeadm/app/apis/kubeadm/v1beta1


* 创建初始化配置文件
可以使用如下命令生成初始化配置文件

``` bash
kubeadm config print init-defaults > kubeadm-config.yaml
```
* 根据实际部署环境修改信息：

``` bash
[root@k8s-master01 ~]# cat kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta1
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
  advertiseAddress: 122.114.139.3
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: k8s-master01
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta1
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controlPlaneEndpoint: "122.114.139.33:6443"
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.aliyuncs.com/google_containers
kind: ClusterConfiguration
kubernetesVersion: v1.14.0
networking:
  dnsDomain: cluster.local
  podSubnet: "10.244.0.0/16"
  serviceSubnet: 10.96.0.0/12
scheduler: {}

---

apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
featureGates:
  SupportIPVSProxyMode: true
mode: ipvs
```
配置说明：
* controlPlaneEndpoint：为vip地址和haproxy监听端口6444
* imageRepository:由于国内无法访问google镜像仓库k8s.gcr.io，这里指定为阿里云镜像仓库registry.aliyuncs.com/google_containers
* podSubnet:指定的IP地址段与后续部署的网络插件相匹配，这里需要部署flannel插件，所以配置为10.244.0.0/16
* mode: ipvs:最后追加的配置为开启ipvs模式。

在集群搭建完成后可以使用如下命令查看生效的配置文件：
``` bash
kubectl -n kube-system get cm kubeadm-config -oyaml
```
#### 2.7 初始化Master01节点
```
kubeadm init --config=kubeadm-config.yaml --experimental-upload-certs | tee kubeadm-init.log
```
该命令指定了初始化时需要使用的配置文件，其中添加–experimental-upload-certs参数可以在后续执行加入节点时自动分发证书文件。
初始化示例

** kubeadm init主要执行了以下操作：**
* [init]：指定版本进行初始化操作
* [preflight] ：初始化前的检查和下载所需要的Docker镜像文件
* [kubelet-start]：生成kubelet的配置文件”/var/lib/kubelet/config.yaml”，没有这个文件kubelet无法启动，所以初始化之前的kubelet实际上启动失败。
* [certificates]：生成Kubernetes使用的证书，存放在/etc/kubernetes/pki目录中。
* [kubeconfig] ：生成 KubeConfig 文件，存放在/etc/kubernetes目录中，组件之间通信需要使用对应文件。
* [control-plane]：使用/etc/kubernetes/manifest目录下的YAML文件，安装 Master 组件。
* [etcd]：使用/etc/kubernetes/manifest/etcd.yaml安装Etcd服务。
* [wait-control-plane]：等待control-plan部署的Master组件启动。
* [apiclient]：检查Master组件服务状态。
* [uploadconfig]：更新配置
* [kubelet]：使用configMap配置kubelet。
* [patchnode]：更新CNI信息到Node上，通过注释的方式记录。
* [mark-control-plane]：为当前节点打标签，打了角色Master，和不可调度标签，这样默认就不会使用Master节点来运行Pod。
* [bootstrap-token]：生成token记录下来，后边使用kubeadm join往集群中添加节点时会用到
* [addons]：安装附加组件CoreDNS和kube-proxy

**说明：无论是初始化失败或者集群已经完全搭建成功，你都可以直接执行kubeadm reset命令清理集群或节点，然后重新执行kubeadm init或kubeadm join相关操作即可**

#### 2.8 配置kubectl命令

**无论在master节点或node节点，要能够执行kubectl命令必须进行以下配置**
* root用户执行以下命令 

``` bash
cat << EOF >> ~/.bashrc
export KUBECONFIG=/etc/kubernetes/admin.conf
EOF
source ~/.bashrc
```

* 普通用户执行以下命令（参考init时的输出结果）

``` bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
等集群配置完成后，可以在所有master节点和node节点进行以上配置，以支持kubectl命令。针对node节点复制任意master节点/etc/kubernetes/admin.conf到本地。

#### 2.9 安装网络插件
** 由于kube-flannel.yml文件指定的镜像从coreos镜像仓库拉取，可能拉取失败，可以从dockerhub搜索相关镜像进行替换，另外可以看到yml文件中定义的网段地址段为10.244.0.0/16。 **
``` bash
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
cat kube-flannel.yml | grep image
cat kube-flannel.yml | grep 10.244
sed -i 's#quay.io/coreos/flannel:v0.11.0-amd64#willdockerhub/flannel:v0.11.0-amd64#g' kube-flannel.yml
kubectl apply -f kube-flannel.yml
```
#### 2.9.2 安装calico网络插件（可选）：
** 安装参考：https://docs.projectcalico.org/v3.6/getting-started/kubernetes/ **

``` bash
kubectl apply -f \
https://docs.projectcalico.org/v3.6/getting-started/kubernetes/installation/hosted/kubernetes-datastore/calico-networking/1.7/calico.yaml
```
** 注意该yaml文件中默认CIDR为192.168.0.0/16，需要与初始化时kube-config.yaml中的配置一致，如果不同请下载该yaml修改后运行。 **

#### 2.10 加入master节点
 ** 依次将k8s-master02和k8s-master03加入到集群中，示例 **

``` bash
kubeadm join 122.114.139.33:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:f446dba72607d3c118576c80d1db0c9a95401ccf91a3a034ce08865eea910120 \
    --experimental-control-plane --certificate-key e70bb4ecdda72405090c05a86e30cf1633200f206da207bcabab618ebb74ea3f
```

#### 2.11 加入node节点
``` bash
kubeadm join 122.114.139.33:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:f446dba72607d3c118576c80d1db0c9a95401ccf91a3a034ce08865eea910120
```

#### 2.12 验证集群状态
``` bash
[root@k8s-master01 ~]# kubectl get nodes -o wide
NAME           STATUS   ROLES    AGE   VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION               CONTAINER-RUNTIME
k8s-master01   Ready    master   73m   v1.14.1   122.114.139.3    <none>        CentOS Linux 7 (Core)   3.10.0-957.12.1.el7.x86_64   docker://18.6.3
k8s-master02   Ready    master   63m   v1.14.1   122.114.139.5    <none>        CentOS Linux 7 (Core)   3.10.0-957.12.1.el7.x86_64   docker://18.6.3
k8s-master03   Ready    master   63m   v1.14.1   122.114.139.22   <none>        CentOS Linux 7 (Core)   3.10.0-957.12.1.el7.x86_64   docker://18.6.3
k8s-node01     Ready    <none>   56m   v1.14.1   122.114.139.23   <none>        CentOS Linux 7 (Core)   3.10.0-957.12.1.el7.x86_64   docker://18.6.3
k8s-node02     Ready    <none>   43m   v1.14.1   122.114.139.25   <none>        CentOS Linux 7 (Core)   3.10.0-957.12.1.el7.x86_64   docker://18.6.3
[root@k8s-master01 ~]#
```

#### 2.13 验证IPVS
** 查看kube-proxy日志，第一行输出Using ipvs Proxier.**
``` bash
[root@k8s-master01 ~]# kubectl -n kube-system logs -f kube-proxy-82gsd
W1221 05:52:58.994387       1 feature_gate.go:218] Setting GA feature gate SupportIPVSProxyMode=true. It will be removed in a future release.
I1221 05:52:59.148009       1 server_others.go:189] Using ipvs Proxier.
W1221 05:52:59.148809       1 proxier.go:381] IPVS scheduler not specified, use rr by default
I1221 05:52:59.148974       1 server_others.go:216] Tearing down inactive rules.
I1221 05:52:59.259207       1 server.go:555] Version: v1.14.0
I1221 05:52:59.283971       1 conntrack.go:100] Set sysctl 'net/netfilter/nf_conntrack_max' to 131072
I1221 05:52:59.290805       1 conntrack.go:52] Setting nf_conntrack_max to 131072
I1221 05:52:59.291311       1 conntrack.go:83] Setting conntrack hashsize to 32768
I1221 05:52:59.301637       1 conntrack.go:100] Set sysctl 'net/netfilter/nf_conntrack_tcp_timeout_established' to 86400
I1221 05:52:59.301750       1 conntrack.go:100] Set sysctl 'net/netfilter/nf_conntrack_tcp_timeout_close_wait' to 3600
I1221 05:52:59.302186       1 config.go:102] Starting endpoints config controller
I1221 05:52:59.302233       1 controller_utils.go:1027] Waiting for caches to sync for endpoints config controller
I1221 05:52:59.302260       1 config.go:202] Starting service config controller
I1221 05:52:59.302295       1 controller_utils.go:1027] Waiting for caches to sync for service config controller
I1221 05:52:59.402882       1 controller_utils.go:1034] Caches are synced for service config controller
I1221 05:52:59.402950       1 controller_utils.go:1034] Caches are synced for endpoints config controlle
```

#### 2.14查看代理规则
``` bash
[root@k8s-master01 ~]# ipvsadm -ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.96.0.1:443 rr
  -> 122.114.139.3:6443           Masq    1      2          0
  -> 122.114.139.5:6443           Masq    1      1          0
  -> 122.114.139.22:6443          Masq    1      0          0
TCP  10.96.0.10:53 rr
  -> 10.244.0.4:53                Masq    1      0          0
  -> 10.244.0.5:53                Masq    1      0          0
TCP  10.96.0.10:9153 rr
  -> 10.244.0.4:9153              Masq    1      0          0
  -> 10.244.0.5:9153              Masq    1      0          0
UDP  10.96.0.10:53 rr
  -> 10.244.0.4:53                Masq    1      0          0
  -> 10.244.0.5:53                Masq    1      0          0
```

#### 2.14 etcd集群

``` bash
kubectl -n kube-system exec etcd-k8s-master01 -- etcdctl --endpoints=https://122.114.139.3:2379 --ca-file=/etc/kubernetes/pki/etcd/ca.crt --cert-file=/etc/kubernetes/pki/etcd/server.crt --key-file=/etc/kubernetes/pki/etcd/server.key cluster-health

[root@k8s-master01 ~]# kubectl -n kube-system exec etcd-k8s-master01 -- etcdctl --endpoints=https://122.114.139.3:2379 --ca-file=/etc/kubernetes/pki/etcd/ca.crt --cert-file=/etc/kubernetes/pki/etcd/server.crt --key-file=/etc/kubernetes/pki/etcd/server.key cluster-health
member 83575205d77c816 is healthy: got healthy result from https://122.114.139.3:2379
member 60e4e52dae62af03 is healthy: got healthy result from https://122.114.139.5:2379
member d2d206bb5cb6c9ce is healthy: got healthy result from https://122.114.139.22:2379
```

#### 2.15 验证HA
查看网卡，vip自动漂移到master03节点



#### 其它错误
The connection to the server localhost:8080 was refused - did you specify the right host or port?
原因：kubenetes master没有与本机绑定，集群初始化的时候没有设置

解决办法：执行以下命令   export KUBECONFIG=/etc/kubernetes/admin.conf

/etc/kubernetes/admin.conf这个文件主要是集群初始化的时候用来传递参数的


```
