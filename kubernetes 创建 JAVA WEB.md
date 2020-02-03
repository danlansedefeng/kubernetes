kubernetes 创建 JAVA WEB
===


## 创建java_web
### 1.创建mysql rc及 svc
``` json
docker pull mysql:5.5
docker pull kubeguide/tomcat-app
```

书上MySQL镜像并没有交代版本号，不指定版本号，docker默认拉取的是最新的版本，目前，MySQL的最新版本是8.0.13，而作者对应tomcat使用的MySQL连接l库是5.1.37，这也就导致部署到最后面一直无法出现正常的表格页面。 

MySQL的RC定义文件
``` json
apiVersion: v1
kind: ReplicationController
metadata:
  name: mysql
spec:
  replicas: 1
  selector:
    app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:5.5
        ports:
        - containerPort: 3306
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "123456"
```
创建一个与之关联的kubernetes service  mysql-svc.yaml
``` json
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
  - port: 3306
  selector:
    app: mysql
```

使用kubectl create命令创建资源
``` json
kubectl create -f mysql-rc.yaml
kubectl create -f mysql-svc.yaml
```

### 2.创建tomcat对应的rc文件 myweb-rc.yaml
``` json
apiVersion: v1
kind: ReplicationController
metadata:
  name: myweb
spec:
  replicas: 1
  selector:
    app: myweb
  template:
    metadata:
      labels:
        app: myweb
    spec:
      containers:
      - name: myweb
        image: kubeguide/tomcat-app:v1
        ports:
        - containerPort: 8080
        env:
        - name: MYSQL_SERVICE_PORT
          value: '3306'
```

创建对应的service文件
``` json
apiVersion: v1
kind: Service
metadata:
  name: myweb
spec:
  type: NodePort
  ports:
  - port: 8080
    nodePort: 30002
  selector:
    app: myweb
```
使用kubectl create命令创建资源
``` json
kubectl create -f myweb-rc.yaml
kubectl create -f myweb-svc.yaml
```

### 3.测试
打开浏览器 http://122.114.139.3:30002/demo 进行测试
https://blog.csdn.net/u010039418/article/details/86583867

## Deployment
构建Deployment定义文件，创建tomcat-deployment.yaml
``` json
#与RC不同之处，版本配置不同
apiVersion: extensions/v1beta1
#与RC不同之处，Kind不同
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      tier: frontend
    matchExpressions:
      - {key: tier, operator: In, values: [frontend]}
  template:
    metadata:
      labels:
        app: app-demo
        tier: frontend
    spec:
      containers:
      - name: tomcat-demo
        image: tomcat
# 设置资源限额，CPU通常以千分之一的CPU配额为最小单位，用m来表示。通常一个容器的CPU配额被定义为100~300m，即占用0.1~0.3个CPU；
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
```
创建deployment
``` json
kubectl create -f tomcat-deployment.yaml
```
查看deployment
``` json
[root@k8smaster java_web]# kubectl get deployment
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
frontend   1/1     1            1           24m
```
* DESIRED, Pod副本数量的期望值，及Deployment里定义的Replica；
* CURRENT，当前Replica实际值；
* UP-TO-DATE,最新版本的Pod副本数量，用于指示在滚动升级的过程中，有多少Pod副本已经成功升级；
* AVAILABLE,当前集群中可用的Pod副本数量

查看对应的Replica Set
``` json
kubectl get rs
```
查看Pod
``` json
kubectl get nodes
```
查看Pod的水平扩展过程
``` json
kubectl describe deployments
```
## HPA
构建HPA定义文件
``` json
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
  namespace: default
spec:
  maxReplicas: 10
  minReplicas: 1
  scaleTargetRef:
    kind: Deployment
    name: php-apache
  targetCPUUtilizationPercentage: 90
```
  `这个HPA控制的目标对象为一个名叫php-apache的Deployment里的Pod副本，当这些Pod副本的CPUUtilizationPercentage超过90%时会触发自动扩容行为，扩容或缩容时必须满足的一个约束条件是Pod的副本数要介于1与10之间；`

命令方式实现相同的功能
``` json
kubectl autoscale deployment php-apache –cpu-percent=90 –min=1 –max=10
```

## Service
创建tomcat-service.yaml
``` json
apiVersion: v1
kind: Service
metadata:
  name: tomcat-service
spec:
# 服务如果想被外部调用，必须配置type
  type: NodePort
  ports:
  - port: 8080
    name: service-port
#   服务如果想被外部调用，必须配置nodePort
    nodePort: 31002
  - port: 8005
    name: shutdown-port
  selector:
    tier: frontend
```
上述内容定义了一个名为“tomcat-service”的Servcie，它的服务端口为8080，拥有“tier=frontend”这个Label的所有Pod实例都属于它； 

这里的NodePort没有完全解决外部访问Service的所有问题，比如负载均衡，假如我们又10个Node，则此时最好有一个负载均衡器，外部的请求只需访问此负载均衡器的IP地址，由负载局衡器负责转发流量到后面某个Node的NodePort上。这个负载均衡器可以是硬件，也可以是软件方式，例如HAProxy或者Nginx； 

如果我们的集群运行在谷歌的GCE公有云上，那么只要我们把Service的type=NodePort改为type=LoadBalancer，此时Kubernetes会自动创建一个对应的LoadBalancer实例并返回它的IP地址供外部客户端使用。其它公有云提供商只要实现了支持此特性的驱动，则也可以达到上述目的；

创建 service
``` json
kubectl create -f tomcat-service.yaml 
```

查看Endpoint列表
``` json
kubectl get endpoints
```

查看tomcat-service更多信息
``` json
[root@k8smaster java_web]# kubectl get svc tomcat-service -o yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: "2019-07-22T08:20:57Z"
  name: tomcat-service
  namespace: default
  resourceVersion: "19206"
  selfLink: /api/v1/namespaces/default/services/tomcat-service
  uid: a154eb16-ac59-11e9-ad2f-005056ab3b9e
spec:
  clusterIP: 10.104.107.99
  externalTrafficPolicy: Cluster
  ports:
  - name: service-port
    nodePort: 31002
    port: 8080
    protocol: TCP
    targetPort: 8080
  - name: shutdown-port
    nodePort: 30375
    port: 8005
    protocol: TCP
    targetPort: 8005
  selector:
    tier: frontend
  sessionAffinity: None
  type: NodePort
status:
  loadBalancer: {}
```

## 补充命令
yaml文件修改后应用
``` json
kubectl apply -f XXX.yaml
```
重新启动基于yaml文件的应用
``` json
kubectl delete -f XXX.yaml
kubectl create -f XXX.yaml
```

动态修改副本数量
``` json
kubectl scale rc XXX --replicas=3
```
查看RC的详细信息
``` json
kubectl describe rc 标签名或者选择器名
```

通过RC修改Pod副本数量
``` json
kubectl replace -f rc.yaml
或
kubect edit replicationcontroller replicationcontroller名
```

对RC使用滚动升级，来发布新功能或修复BUG
``` json
kubectl rolling-update replicationcontroller名 --image=镜像名
```
滚动升级
``` json
kubectl rolling-update replicationcontroller名 -f XXX.yaml
```


https://blog.csdn.net/wo18237095579/article/details/89376877


