  

#### 《如何为容器提供持久化存储》，kubernetes存储系统的一些核心概念，包括:
 * StorageClass(SC): 存储类，类似存储池的概念，包括了存储池的一些信息；
 
 * PersistenVolume(PV):持久卷，独立的存储资源对象，其生命周期与POD无关；
 * PersistenVolumeClaim(PVC)：声明，代表计算任务对存储资源的需求。
 
 ![avater](https://www.xsky.com/wp-content/uploads/2019/01/1-1.gif)
 
####这种存储结构是为了解决
 
 * 有状态的容器实例需要持久化卷；
 * 容器实例动态提供新卷；
 * 不同类型的应用需要不同访问模式的存储卷；
 * 解耦计算资源和存储资源。
 
 卷(Volume)，持久卷(PersistoneVolume)，存储类(StorageClass)，供给器(Provisioner)，容器存储接口(CSI)，动态供应(Dynamic provisioning)等等到底是什么。
 
#### 一、Kubernetes存储对接如何选择
 如果有使用过Docker经验的话，就会发现在Dovker中就有Volume的概念，Volume本质上是容器上挂载的某个目录
 1. Volume(卷)
 Kubernetes同样也有Volume的概念，它属于Pod内部共享资源存储，生命周期和Pod相同，与Container无关，即使Pod上的容器停止或者重启，Volume不会受到影响，但是如果Pod终止，name这个Volume的生命周期也将结束。

 2. Persistent Volume(持久卷)
 显然，这样的存储无法满足有状态服务的需求，于是有个Persistent Volume(PV)即持久卷，顾名思义，持久卷是能将数据进行持久化存储的一种资源对象。PV，它是独立于Pod的一种资源，是一种网络存储，它的生命周期和Pod无关。云原生的时代，PV的种类也包括很多，包括ISCSI,RBD,NFS,CephFS,GlusterFS,等网络存储。

![avater](https://www.xsky.com/wp-content/uploads/2019/01/2.gif)

可以看出Volume和Persistent Volume的最大区别是生命周期，PV是一种独立的存储，管理员只需要在存储上创建足够的PV，就可以提供给计算节点中的Pod来使用，而且数据是持久存储的，这种手动创建的PV提供给Pod使用的方式也焦作静态供应。

Dynamic Provisioning (动态供应)
ISCSI,RBD,NFS等PV分别代表了各种类型的存储资源，供集群消费，
PersistentVolumeClaim(PVC)就是对这些存储资源的请求。PVC消耗的资源是PV，这很好理解，就像Pod消耗Node资源一样。于是，在Kubernetes中Pod可以不直接使用PV，而是通过PVC来使用PV。

通过PVC可以将Pod和PV解耦，Pod不需要知道确切的文件系统和支持它的持久化引擎，可以把PVC理解为存储的抽象，把底层存储细节给隔离了。另一方面，如果使用PV作为存储的话，需要集群管理员事先创建好这些PV，但是如果使用PVC的话则不需要提前准备好PV，而是通过StorageClass把存储资源定义好，Kubernetes会在需要使用的时候动态创建，这种方式成为动态供应(Dynamic Provisioning)

![avater](https://www.xsky.com/wp-content/uploads/2019/01/3-1.gif)

#### 详细流程分析如下:

* 1.Pod加载存储卷，请求PVC
* 2.PVC根据存储类型(此处为rbd)找到存储类StorageClass
* 3.Provisioner根据StorageClass动态生成一个持久卷PV
* 4.持久卷PV和PVC最终形成绑定关系
* 5.持久卷PV开始提供给Pod使用

简单总结下，如果在Kubernetes中运行有状态服务，比如数据库MySQL,MongoDB或者中间件Redis等，name就需要用持久卷(pv),从而不用担心随着Pod的终止而丢失数据(相比使用了普通Volume).另外，从以上对比可以看出，直接使用PV适合变动较少，不会频繁修改的场景，是比较直接的使用方式。
