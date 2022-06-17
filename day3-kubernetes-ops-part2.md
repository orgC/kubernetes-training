# storage 



## 本地NFS 存储



### NFS-Server 配置

```

yum install -y nfs-utils rpcbind

mkdir -p /nfs
echo "/nfs *(rw,no_root_squash)" > /etc/exports         # 需要使用no_root_squash，否则没有权限写入

exportfs -rv

chmod 777 /nfs    # 不然的话会有权限问题

# 启动服务
systemctl enable rpcbind
systemctl start rpcbind
systemctl enable nfs-server
systemctl start nfs-server


rpcinfo -p
查看具体目录挂载权限
cat /var/lib/nfs/etab

[root@registry ~]# exportfs -s
/nfs  *(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,no_root_squash,no_all_squash)


# 配置防火墙 
firewall-cmd --permanent --add-service=nfs
firewall-cmd --permanent --add-service=rpc-bind
firewall-cmd --permanent --add-service=mountd
firewall-cmd --reload

# 创建PV目录

for i in {1..10}
do
mkdir -p /nfs/volumes/pv000$i
chmod 777 /nfs/volumes/pv000$i
done;

chmod 777 /nfs/volumes
```



### NFS-Client 配置

```
yum install -y nfs-utils 

cat <<EOF>> /etc/hosts
192.168.26.100 registry.myk8s.example.com
EOF

# 验证nfs 服务
showmount -e registry.myk8s.example.com

```



### 创建PV

```
# PV1
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0001
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteOnce
  nfs:
    path: /nfs/volumes/pv0001
    server: registry.myk8s.example.com
  persistentVolumeReclaimPolicy: Recycle
EOF

# PV2
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0002
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteOnce
  nfs:
    path: /nfs/volumes/pv0002
    server: registry.myk8s.example.com
  persistentVolumeReclaimPolicy: Recycle
EOF

#PV3 
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0003
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteOnce
  nfs:
    path: /nfs/volumes/pv0003
    server: registry.myk8s.example.com
  persistentVolumeReclaimPolicy: Recycle
EOF


```





## rook 介绍

 Rook 是一个可以提供 Ceph 集群管理能力的 [Operator](https://coreos.com/blog/introducing-operators.html)。Rook 使用 CRD 来对 Ceph 之类的资源进行部署和管理

![rook](/Users/chenjunkai/work/kubernetes-training/day3-kubernetes-ops-part2.assets/rook-arch.png)



### Rook包含的组件

**Rook Operator**：Rook 的核心组件，Rook Operator 是一个简单的容器，自动启动存储集群，并监控存储守护进程，来确保存储集群的健康。

**Rook Agent**：在每个存储节点上运行，并配置一个 FlexVolume 插件，和 Kubernetes 的存储卷控制框架进行集成。Agent 处理所有的存储操作，例如挂接网络存储设备、在主机上加载存储卷以及格式化文件系统等。

**Rook Discovers**：检测挂接到存储节点上的存储设备。

Rook 还会用 Kubernetes Pod 的形式，部署 Ceph 的 MON、OSD 以及 MGR 守护进程。

Rook Operator 让用户可以通过 CRD 的是用来创建和管理存储集群。每种资源都定义了自己的 CRD.

**Rook [Cluster](https://rook.github.io/docs/rook/master/cluster-crd.html)**：提供了对存储机群的配置能力，用来提供块存储、对象存储以及共享文件系统。每个集群都有多个 Pool。

**[Pool](https://rook.github.io/docs/rook/master/pool-crd.html)**：为块存储提供支持。Pool 也是给文件和对象存储提供内部支持。

**[Object Store](https://rook.github.io/docs/rook/master/object-store-crd.html)**：用 S3 兼容接口开放存储服务。

**[File System](https://rook.github.io/docs/rook/master/filesystem-crd.html)**：为多个 Kubernetes Pod 提供共享存储。



### Rook 安装

rook 安装的话，需要在计算节点上添加一块裸盘



## PV 类型







# statefulset

StatefulSet 的设计逻辑。它把真实世界里的应用状态，抽象为了两种情况

* **拓扑状态**，这种情况意味着，应用的多个实例之间不是完全对等的关系。这些应用实例，必须按照某些顺序启动，比如应用的主节点 A 要先于从节点 B 启动。而如果你把 A 和 B 两个 Pod 删除掉，它们再次被创建出来时也必须严格按照这个顺序才行。并且，新创建出来的 Pod，必须和原来 Pod 的网络标识一样，这样原先的访问者才能使用同样的方法，访问到这个新 Pod。
* **存储状态**， 这种情况意味着，应用的多个实例分别绑定了不同的存储数据。对于这些应用实例来说，Pod A 第一次读取到的数据，和隔了十分钟之后再次读取到的数据，应该是同一份，哪怕在此期间 Pod A 被重新创建过。这种情况最典型的例子，就是一个数据库应用的多个存储实例。

StatefulSet 的核心功能，就是通过某种方式记录这些状态，然后在 Pod 被重新创建时，能够为新 Pod 恢复这些状态



## Service

Service 是 Kubernetes 项目中用来将一组 Pod 暴露给外界访问的一种机制。比如，一个 Deployment 有 3 个 Pod，那么可以定义一个 Service。然后，用户只要能访问到这个 Service，它就能访问到某个具体的 Pod。



## Headless Service

有时不需要或不想要负载均衡，以及单独的 Service IP。 遇到这种情况，可以通过指定 Cluster IP（`spec.clusterIP`）的值为 `"None"` 来创建 `Headless` Service

对这无头 Service 并不会分配 Cluster IP，kube-proxy 不会处理它们， 而且平台也不会为它们进行负载均衡和路由。 DNS 如何实现自动配置，依赖于 Service 是否定义了选择算符

```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
EOF

```



部署一个statefulset 服务

```

cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.9.1
        ports:
        - containerPort: 80
          name: web
EOF

```

测试



```
# 查看 pod 的hostname
[root@master1 ~]# kubectl exec web-0 -- sh -c 'hostname'
web-0
[root@master1 ~]#
[root@master1 ~]# kubectl exec web-1 -- sh -c 'hostname'
web-1


kubectl run -i --tty --image busybox:1.28.4 dns-test --restart=Never --rm /bin/sh

# 通过nslookup 可以获取服务IP
/ # nslookup web-1.nginx
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-1.nginx
Address 1: 10.244.210.249 web-1.nginx.default.svc.cluster.local
/ #
/ # nslookup web-0.nginx
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-0.nginx
Address 1: 10.244.104.179

```





StatefulSet 这个控制器的主要作用之一，就是使用 Pod 模板创建 Pod 的时候，对它们进行编号，并且按照编号顺序逐一完成创建工作。而当 StatefulSet 的“控制循环”发现 Pod 的“实际状态”与“期望状态”不一致，需要新建或者删除 Pod 进行“调谐”的时候，它会严格按照这些 Pod 编号的顺序，逐一完成这些操作。





## zookeeper

### 部署



```

apiVersion: v1
kind: Service
metadata:
  name: zk-hs
  labels:
    app: zk
spec:
  ports:
  - port: 2888
    name: server
  - port: 3888
    name: leader-election
  clusterIP: None
  selector:
    app: zk
---
apiVersion: v1
kind: Service
metadata:
  name: zk-cs
  labels:
    app: zk
spec:
  ports:
  - port: 2181
    name: client
  selector:
    app: zk
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: zk-pdb
spec:
  selector:
    matchLabels:
      app: zk
  maxUnavailable: 1
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: zk
spec:
  selector:
    matchLabels:
      app: zk
  serviceName: zk-hs
  replicas: 3
  updateStrategy:
    type: RollingUpdate
  podManagementPolicy: OrderedReady
  template:
    metadata:
      labels:
        app: zk
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: "app"
                    operator: In
                    values:
                    - zk
              topologyKey: "kubernetes.io/hostname"
      containers:
      - name: kubernetes-zookeeper
        imagePullPolicy: Always
        image: "quay.io/junkai/training:zk-1.0-3.4.10"
        resources:
          requests:
            memory: "1Gi"
            cpu: "0.5"
        ports:
        - containerPort: 2181
          name: client
        - containerPort: 2888
          name: server
        - containerPort: 3888
          name: leader-election
        command:
        - sh
        - -c
        - "start-zookeeper \
          --servers=3 \
          --data_dir=/var/lib/zookeeper/data \
          --data_log_dir=/var/lib/zookeeper/data/log \
          --conf_dir=/opt/zookeeper/conf \
          --client_port=2181 \
          --election_port=3888 \
          --server_port=2888 \
          --tick_time=2000 \
          --init_limit=10 \
          --sync_limit=5 \
          --heap=512M \
          --max_client_cnxns=60 \
          --snap_retain_count=3 \
          --purge_interval=12 \
          --max_session_timeout=40000 \
          --min_session_timeout=4000 \
          --log_level=INFO"
        readinessProbe:
          exec:
            command:
            - sh
            - -c
            - "zookeeper-ready 2181"
          initialDelaySeconds: 10
          timeoutSeconds: 5
        livenessProbe:
          exec:
            command:
            - sh
            - -c
            - "zookeeper-ready 2181"
          initialDelaySeconds: 10
          timeoutSeconds: 5
        volumeMounts:
        - name: datadir
          mountPath: /var/lib/zookeeper
      securityContext:
        runAsUser: 1000
        fsGroup: 1000
  volumeClaimTemplates:
  - metadata:
      name: datadir
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
          
```



### 准备PV

```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0005
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteOnce
  nfs:
    path: /nfs/volumes/pv0005
    server: registry.myk8s.example.com
  persistentVolumeReclaimPolicy: Recycle
EOF

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0006
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteOnce
  nfs:
    path: /nfs/volumes/pv0006
    server: registry.myk8s.example.com
  persistentVolumeReclaimPolicy: Recycle
EOF

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0007
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteOnce
  nfs:
    path: /nfs/volumes/pv0007
    server: registry.myk8s.example.com
  persistentVolumeReclaimPolicy: Recycle
EOF
```







# 存储

## 运行一个单实例有状态应用（MySQL）



部署文件

```
[root@master1 sts]# cat single-mysql.yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
  - port: 3306
  selector:
    app: mysql
  clusterIP: None
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - image: mysql:5.6
        name: mysql
        env:
          # Use secret in real usage
        - name: MYSQL_ROOT_PASSWORD
          value: password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pv-claim

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi


kubectl apply -f single-mysql.yaml 
```



创建PV

```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0004
spec:
  capacity:
    storage: 20Gi
  accessModes:
  - ReadWriteOnce
  nfs:
    path: /nfs/volumes/pv0004
    server: registry.myk8s.example.com
  persistentVolumeReclaimPolicy: Recycle
EOF
```







# JOB

一个job示例， *要注意 perl 版本*， 



```

apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    spec:
      containers:
      - name: pi
        image: perl:5.34.1
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
  backoffLimit: 4


```



## cronjob

利用 [CronJobs](https://kubernetes.io/zh/docs/concepts/workloads/controllers/cron-jobs) 执行基于时间调度的任务。这些自动化任务和 Linux 或者 Unix 系统的 [Cron](https://en.wikipedia.org/wiki/Cron) 任务类似



cronjob 时间表

```
# ┌───────────── 分钟 (0 - 59)
# │ ┌───────────── 小时 (0 - 23)
# │ │ ┌───────────── 月的某天 (1 - 31)
# │ │ │ ┌───────────── 月份 (1 - 12)
# │ │ │ │ ┌───────────── 周的某天 (0 - 6)（周日到周一；在某些系统上，7 也是星期日）
# │ │ │ │ │                          或者是 sun，mon，tue，web，thu，fri，sat
# │ │ │ │ │
# │ │ │ │ │
# * * * * *
```



```
[root@master1 job]# cat cronjob-hello.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "* * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox:1.28
            imagePullPolicy: IfNotPresent
            command:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
```

相关操作

```
# 部署 
kubectl apply -f cronjob-hello.yaml

# 观察
kubectl get jobs --watch

# 查看 cronjob 
[root@master1 job]# kubectl get cronjobs
NAME    SCHEDULE    SUSPEND   ACTIVE   LAST SCHEDULE   AGE
hello   * * * * *   False     0        5s              13m

# 查看job 
[root@master1 job]# kubectl get job
NAME             COMPLETIONS   DURATION   AGE
hello-27589121   1/1           3s         2m18s
hello-27589122   1/1           3s         78s
hello-27589123   1/1           3s         18s
pi               1/1           47s        28m

# 删除cronjob 
[root@master1 job]# kubectl delete cronjob/hello
cronjob.batch "hello" deleted
```





# 扩展Kubernetes API 

*定制资源（Custom Resource）* 是对 Kubernetes API 的扩展， 

*资源（Resource）* 是 [Kubernetes API](https://kubernetes.io/zh/docs/concepts/overview/kubernetes-api/) 中的一个端点， 其中存储的是某个类别的 [API 对象](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/kubernetes-objects/) 的一个集合。 例如内置的 *pods* 资源包含一组 Pod 对象

*定制资源（Custom Resource）* 是对 Kubernetes API 的扩展，不一定在默认的 Kubernetes 安装中就可用。定制资源所代表的是对特定 Kubernetes 安装的一种定制。 不过，很多 Kubernetes 核心功能现在都用定制资源来实现，这使得 Kubernetes 更加模块化。

定制资源可以通过动态注册的方式在运行中的集群内或出现或消失，集群管理员可以独立于集群 更新定制资源。一旦某定制资源被安装，用户可以使用 [kubectl](https://kubernetes.io/docs/reference/kubectl/) 来创建和访问其中的对象，就像他们为 *pods* 这种内置资源所做的一样







# troubleshooting

## pvc Terminating 

```
PVC_NAME=xxx
kubectl patch pvc {PVC_NAME} -p '{"metadata":{"finalizers":null}}'
```