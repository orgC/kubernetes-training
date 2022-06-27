# storage 



## 本地 NFS 存储



### NFS-Server 配置

```

yum install -y nfs-utils rpcbind

mkdir -p /nfs
echo "/nfs *(rw,no_root_squash)" > /etc/exports         # 需要使用 no_root_squash，否则没有权限写入

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

for i in {1..9}
do
mkdir -p /nfs/volumes/pv000$i
chmod 777 /nfs/volumes/pv000$i
done;

chmod 777 /nfs/volumes
```



### NFS-Client 配置

```
# 需要在每一个worker节点上执行以下命令 
yum install -y nfs-utils 

cat <<EOF>> /etc/hosts
192.168.26.100 registry.myk8s.example.com
EOF

# 验证nfs 服务
showmount -e registry.myk8s.example.com

```



## rook 介绍

 Rook 是一个可以提供 Ceph 集群管理能力的 [Operator](https://coreos.com/blog/introducing-operators.html)。Rook 使用 CRD 来对 Ceph 之类的资源进行部署和管理

![rook](/Users/chenjunkai/work/kubernetes-training/day4-kubernetes-ops.assets/rook-arch.png)



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



## PV 



### 访问模式

PersistentVolume 卷可以用资源提供者所支持的任何方式挂载到宿主系统上。 提供者（驱动）的能力不同，每个 PV 卷的访问模式都会设置为 对应卷所支持的模式值。 例如，NFS 可以支持多个读写客户，但是某个特定的 NFS PV 卷可能在服务器 上以只读的方式导出。每个 PV 卷都会获得自身的访问模式集合，描述的是 特定 PV 卷的能力。

访问模式有：

- `ReadWriteOnce`

  卷可以被一个节点以读写方式挂载。 ReadWriteOnce 访问模式也允许运行在同一节点上的多个 Pod 访问卷。

- `ReadOnlyMany`

  卷可以被多个节点以只读方式挂载。

- `ReadWriteMany`

  卷可以被多个节点以读写方式挂载。

- `ReadWriteOncePod`

  卷可以被单个 Pod 以读写方式挂载。 如果你想确保整个集群中只有一个 Pod 可以读取或写入该 PVC， 请使用ReadWriteOncePod 访问模式。这只支持 CSI 卷以及需要 Kubernetes 1.22 以上版本。

在命令行接口（CLI）中，访问模式也使用以下缩写形式：

- RWO - ReadWriteOnce
- ROX - ReadOnlyMany
- RWX - ReadWriteMany
- RWOP - ReadWriteOncePod

### 阶段

每个卷会处于以下阶段（Phase）之一：

- Available（可用）-- 卷是一个空闲资源，尚未绑定到任何申领；
- Bound（已绑定）-- 该卷已经绑定到某申领；
- Released（已释放）-- 所绑定的申领已被删除，但是资源尚未被集群回收；
- Failed（失败）-- 卷的自动回收操作失败。

### 回收策略

目前的回收策略有：

- Retain -- 手动回收
- Recycle -- 基本擦除 (`rm -rf /thevolume/*`)
- Delete -- 诸如 AWS EBS、GCE PD、Azure Disk 或 OpenStack Cinder 卷这类关联存储资产也被删除

目前，仅 NFS 和 HostPath 支持回收（Recycle）。 AWS EBS、GCE PD、Azure Disk 和 Cinder 卷都支持删除（Delete）。



### 选择算符

申领可以设置[标签选择算符](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/labels/#label-selectors) 来进一步过滤卷集合。只有标签与选择算符相匹配的卷能够绑定到申领上。 选择算符包含两个字段：

- `matchLabels` - 卷必须包含带有此值的标签
- `matchExpressions` - 通过设定键（key）、值列表和操作符（operator） 来构造的需求。合法的操作符有 In、NotIn、Exists 和 DoesNotExist。

来自 `matchLabels` 和 `matchExpressions` 的所有需求都按逻辑与的方式组合在一起。 这些需求都必须被满足才被视为匹配。



### Demo

#### pv + RWO + recycle 

```
# 创建PV

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


# 创建pvc 
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
EOF


## 创建应用 

cat <<EOF | kubectl apply -f -
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
EOF

# 等到应用ready 之后查看，可以看到nfs节点上存储了数据
[root@registry volumes]# du -sh *
111M	pv0001
0	pv00010

# 
kubectl delete deploy mysql
kubectl delete pvc mysql-pv-claim
kubectl get pv   # 此时pv 恢复为 Available  状态 ， 同时到NFS节点上查看，可以看到，对应目录已经被清空

此时重新执行创建pvc命令，依然可以绑定到原来的PV上

```



#### pv + RWO + retain

```


# 创建PV

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0002
spec:
  capacity:
    storage: 20Gi
  accessModes:
  - ReadWriteOnce
  nfs:
    path: /nfs/volumes/pv0002
    server: registry.myk8s.example.com
  persistentVolumeReclaimPolicy: Retain
EOF


# 创建pvc 
cat <<EOF | kubectl apply -f -
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
EOF

## 创建应用 

cat <<EOF | kubectl apply -f -
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
EOF

kubectl delete deploy mysql
kubectl delete pvc mysql-pv-claim
kubectl get pv   # 此时pv 为 Released 状态 ， 同时到NFS节点上查看，可以看到，对应目录内容还在

此时重新执行上述 创建 PVC 命令，无法获取导致资源，此时，需要手动删除 pv0002， 释放资源 

```

#### PV + RWO + delete 

```

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0003
spec:
  capacity:
    storage: 30Gi
  accessModes:
  - ReadWriteOnce
  nfs:
    path: /nfs/volumes/pv0003
    server: registry.myk8s.example.com
  persistentVolumeReclaimPolicy: Delete
EOF

# 创建pvc 
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 30Gi
EOF

## 创建应用 

cat <<EOF | kubectl apply -f -
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
EOF

# 删除deploy，释放 pvc 后查看 PV中数据的状态 

```



#### PV + label 

```
# 创建pv， 没有label

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0004-nolabel
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteOnce
  nfs:
    path: /nfs/volumes/pv0004
    server: registry.myk8s.example.com
  persistentVolumeReclaimPolicy: Retain
EOF

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
spec:
  accessModes:
    - ReadWriteOnce
  selector:
    matchLabels:
      mylabel: abc
  resources:
    requests:
      storage: 10Gi
EOF


cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0005-label
  labels: 
    mylabel: abc
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteOnce
  nfs:
    path: /nfs/volumes/pv0005
    server: registry.myk8s.example.com
  persistentVolumeReclaimPolicy: Retain
EOF

```



## storageclass



在 ocp 集群上测试 

```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: claim1
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ocs-storagecluster-ceph-rbd
  resources:
    requests:
      storage: 10Gi
EOF
```





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



## DEMO



StatefulSet 中的 Pod 拥有一个唯一的顺序索引和稳定的网络身份标识。



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



StatefulSet 中的 Pod 拥有一个具有黏性的、独一无二的身份标志。 这个标志基于 StatefulSet 控制器分配给每个 Pod 的唯一顺序索引。 Pod 的名称的形式为`<statefulset name>-<ordinal index>`。 `web`StatefulSet 拥有两个副本，所以它创建了两个 Pod：`web-0`和`web-1`。



```
# 查看 pod 的hostname
[root@master1 ~]# kubectl exec web-0 -- sh -c 'hostname'
web-0
[root@master1 ~]#
[root@master1 ~]# kubectl exec web-1 -- sh -c 'hostname'
web-1

kubectl run -i --tty --image busybox:1.28 dns-test --restart=Never --rm /bin/sh

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


# 测试stateful pod的启动顺序, 在一个窗口 查看 
kubectl get pod -w -l app=nginx

# 在另一个窗口删除stateful pod
kubectl delete pod -l app=nginx

# 可以看到，两个pod的启动顺序是严格按照顺序启动的

kubectl run -i --tty --image busybox:1.28 dns-test --restart=Never --rm /bin/sh
nslookup web-0.nginx

Pod 的序号、主机名、SRV 条目和记录名称没有改变，但和 Pod 相关联的 IP 地址可能发生了改变

如果你的应用已经实现了用于测试 liveness 和 readiness 的连接逻辑，你可以使用 Pod 的 SRV 记录（web-0.nginx.default.svc.cluster.local， web-1.nginx.default.svc.cluster.local）。因为他们是稳定的，并且当你的 Pod 的状态变为 Running 和 Ready 时，你的应用就能够发现它们的地址

# 扩容 sts
kubectl scale --replicas=5 sts web

# 缩容 sts
kubectl scale --replicas=3 sts web

# 可以看到，sts 按照顺序扩容，按照顺序缩容 


# 更新镜像， 可以看到pod 按照与序号相反的顺序更新， 
# RollingUpdate 更新策略会更新一个 StatefulSet 中所有的 Pod，采用与序号索引相反的顺序并遵循 StatefulSet 的保证

kubectl patch statefulset web --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/image", "value":"quay.io/junkai/training:nginx-slim-0.7"}]'

# 查看镜像

for p in 0 1 2; do kubectl get pod "web-$p" --template '{{range $i, $c := .spec.containers}}{{$c.image}}{{end}}'; echo; done


## 分段更新
可以使用类似灰度发布的方法执行一次分阶段的发布（例如一次线性的、等比的或者指数形式的发布）。 要执行一次分阶段的发布，你需要设置 partition 为希望控制器暂停更新的序号

kubectl patch statefulset web -p '{"spec":{"updateStrategy":{"type":"RollingUpdate","rollingUpdate":{"partition":3}}}}'

kubectl patch statefulset web --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/image", "value":"quay.io/junkai/training:nginx-slim-0.8"}]'

# 此时pod 没有变化，可以执行以下命令查看 
for p in 0 1 2; do kubectl get pod "web-$p" --template '{{range $i, $c := .spec.containers}}{{$c.image}}{{end}}'; echo; done

# 删除 web-2
kubectl delete pod web-2

# 将分区设置为2 
kubectl patch statefulset web -p '{"spec":{"updateStrategy":{"type":"RollingUpdate","rollingUpdate":{"partition":2}}}}'

# 将分区设置为1， 和0 
kubectl patch statefulset web -p '{"spec":{"updateStrategy":{"type":"RollingUpdate","rollingUpdate":{"partition":1}}}}'

当指定了分区时，如果更新了 StatefulSet 的 .spec.template，则所有序号大于或等于分区的 Pod 都将被更新。 如果一个序号小于分区的 Pod 被删除或者终止，它将被按照原来的配置恢复。


```





StatefulSet 这个控制器的主要作用之一，就是使用 Pod 模板创建 Pod 的时候，对它们进行编号，并且按照编号顺序逐一完成创建工作。而当 StatefulSet 的“控制循环”发现 Pod 的“实际状态”与“期望状态”不一致，需要新建或者删除 Pod 进行“调谐”的时候，它会严格按照这些 Pod 编号的顺序，逐一完成这些操作。



### sts 并行



`OrderedReady` pod 管理策略是 StatefulSets 的默认选项。它告诉 StatefulSet 控制器遵循上文展示的顺序性保证。

`Parallel` pod 管理策略告诉 StatefulSet 控制器并行的终止所有 Pod， 在启动或终止另一个 Pod 前，不必等待这些 Pod 变成 Running 和 Ready 或者完全终止状态。

```

cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  podManagementPolicy: "Parallel"
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



demo

```
# 等到服务 ready 之后执行 , 获取主机名 

for i in 0 1 2; do kubectl exec zk-$i -- hostname; done

# 检查每个服务器的 myid 文件的内容
for i in 0 1 2; do echo -n "myid zk-$i: ";kubectl exec zk-$i -- cat /var/lib/zookeeper/data/myid; done

# 获取 zk StatefulSet 中每个 Pod 的全限定域名（Fully Qualified Domain Name，FQDN)
for i in 0 1 2; do kubectl exec zk-$i -- hostname -f; done

# 临时测试， 如果有需要的话
kubectl run -i --tty --image busybox:1.28 dns-test --restart=Never --rm /bin/sh

# 查看配置文件

kubectl exec zk-0 -- cat /opt/zookeeper/conf/zoo.cfg


# 插入一条记录
kubectl exec zk-0 zkCli.sh create /hello world

# 查询记录
kubectl exec zk-1 zkCli.sh get /hello

# 随机删除一个pod 
kubectl delete pod zk-2 --force --grace-period=0
kubectl delete pod zk-1 --force --grace-period=0
kubectl delete pod zk-0 --force --grace-period=0
```





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





# DaemonSet

Daemonset 确保全部（或者某些）节点上运行一个 Pod 的副本。 当有节点加入集群时， 也会为他们新增一个 Pod 。 当有节点从集群移除时，这些 Pod 也会被回收。删除 DaemonSet 将会删除它创建的所有 Pod。

DaemonSet 的一些典型用法：

- 在每个节点上运行集群守护进程
- 在每个节点上运行日志收集守护进程
- 在每个节点上运行监控守护进程

一种简单的用法是为每种类型的守护进程在所有的节点上都启动一个 DaemonSet。 一个稍微复杂的用法是为同一种守护进程部署多个 DaemonSet；每个具有不同的标志， 并且对不同硬件类型具有不同的内存、CPU 要求



```

kubectl create ns logging 

cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: logging
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      # 这些容忍度设置是为了让守护进程在控制平面节点上运行
      # 如果你不希望控制平面节点运行 Pod，可以删除它们
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
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



# RBAC： role,  binding 



基于角色（Role）的访问控制（RBAC）是一种基于组织中用户的角色来调节控制对 计算机或网络资源的访问的方法

RBAC API 声明了四种 Kubernetes 对象：*Role*、*ClusterRole*、*RoleBinding* 和 *ClusterRoleBinding*

## Role 和 ClusterRole

RBAC 的 *Role* 或 *ClusterRole* 中包含一组代表相关权限的规则。 这些权限是纯粹累加的（不存在拒绝某操作的规则）。

Role 总是用来在某个[名字空间](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/namespaces/) 内设置访问权限；在你创建 Role 时，你必须指定该 Role 所属的名字空间。

与之相对，ClusterRole 则是一个集群作用域的资源。这两种资源的名字不同（Role 和 ClusterRole）是因为 Kubernetes 对象要么是名字空间作用域的，要么是集群作用域的， 不可两者兼具。

ClusterRole 有若干用法。你可以用它来：

1. 定义对某名字空间域对象的访问权限，并将在各个名字空间内完成授权；
2. 为名字空间作用域的对象设置访问权限，并跨所有名字空间执行授权；
3. 为集群作用域的资源定义访问权限。

如果你希望在名字空间内定义角色，应该使用 Role； 如果你希望定义集群范围的角色，应该使用 ClusterRole



## DEMO



```
# role 示例
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""] # "" 标明 core API 组
  resources: ["pods"]
  verbs: ["get", "watch", "list"]

# cluster role 示例

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  # "namespace" 被忽略，因为 ClusterRoles 不受名字空间限制
  name: secret-reader
rules:
- apiGroups: [""]
  # 在 HTTP 层面，用来访问 Secret 资源的名称为 "secrets"
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]
  
  

```

## RoleBinding 和 ClusterRoleBinding



demo

```

# rolebindig 

apiVersion: rbac.authorization.k8s.io/v1
# 此角色绑定允许 "jane" 读取 "default" 名字空间中的 Pods
# 你需要在该命名空间中有一个名为 “pod-reader” 的 Role
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
# 你可以指定不止一个“subject（主体）”
- kind: User
  name: jane # "name" 是区分大小写的
  apiGroup: rbac.authorization.k8s.io
roleRef:
  # "roleRef" 指定与某 Role 或 ClusterRole 的绑定关系
  kind: Role # 此字段必须是 Role 或 ClusterRole
  name: pod-reader # 此字段必须与你要绑定的 Role 或 ClusterRole 的名称匹配
  apiGroup: rbac.authorization.k8s.io


################################

# clusterrolebinding 

apiVersion: rbac.authorization.k8s.io/v1
# 此集群角色绑定允许 “manager” 组中的任何人访问任何名字空间中的 secrets
kind: ClusterRoleBinding
metadata:
  name: read-secrets-global
subjects:
- kind: Group
  name: manager # 'name' 是区分大小写的
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
  
################################
#  基于SA 的 rolebinding

## 创建SA 
kubectl create sa demo-sa

###
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: sa-read-pods
  namespace: default
subjects:
- kind: ServiceAccount
  name: demo-sa 
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
  
```





# 扩展Kubernetes API 

*定制资源（Custom Resource）* 是对 Kubernetes API 的扩展， 

*资源（Resource）* 是 [Kubernetes API](https://kubernetes.io/zh/docs/concepts/overview/kubernetes-api/) 中的一个端点， 其中存储的是某个类别的 [API 对象](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/kubernetes-objects/) 的一个集合。 例如内置的 *pods* 资源包含一组 Pod 对象

*定制资源（Custom Resource）* 是对 Kubernetes API 的扩展，不一定在默认的 Kubernetes 安装中就可用。定制资源所代表的是对特定 Kubernetes 安装的一种定制。 不过，很多 Kubernetes 核心功能现在都用定制资源来实现，这使得 Kubernetes 更加模块化。

定制资源可以通过动态注册的方式在运行中的集群内或出现或消失，集群管理员可以独立于集群 更新定制资源。一旦某定制资源被安装，用户可以使用 [kubectl](https://kubernetes.io/docs/reference/kubectl/) 来创建和访问其中的对象，就像他们为 *pods* 这种内置资源所做的一样



```
cat <<EOF | kubectl apply -f -
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  # 名字必需与下面的 spec 字段匹配，并且格式为 '<名称的复数形式>.<组名>'
  name: crontabs.stable.example.com
spec:
  # 组名称，用于 REST API: /apis/<组>/<版本>
  group: stable.example.com
  # 列举此 CustomResourceDefinition 所支持的版本
  versions:
    - name: v1
      # 每个版本都可以通过 served 标志来独立启用或禁止
      served: true
      # 其中一个且只有一个版本必需被标记为存储版本
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                cronSpec:
                  type: string
                image:
                  type: string
                replicas:
                  type: integer
  # 可以是 Namespaced 或 Cluster
  scope: Namespaced
  names:
    # 名称的复数形式，用于 URL：/apis/<组>/<版本>/<名称的复数形式>
    plural: crontabs
    # 名称的单数形式，作为命令行使用时和显示时的别名
    singular: crontab
    # kind 通常是单数形式的帕斯卡编码（PascalCased）形式。你的资源清单会使用这一形式。
    kind: CronTab
    # shortNames 允许你在命令行使用较短的字符串来匹配资源
    shortNames:
    - ct
EOF


### 创建对象
cat <<EOF | kubectl apply -f -
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  name: my-new-cron-object
spec:
  cronSpec: "* * * * */5"
  image: my-awesome-cron-image
EOF

## CMD
kubectl get crontab
kubectl describe crontabs.stable.example.com  my-new-cron-object

```



## helm

Helm是基于Kubernetes的应用包管理工具，可实现应用程序封装、版本管理、依赖检查、应用分发，是对容器应用所需资源组件进行集中管理，并通过模板化和配置分离提高声明式API的开发和使用效率。Chart包则是应用的载体，实现了应用的分发和共享，支撑了应用市场的发展。

```

curl -LO https://get.helm.sh/helm-v3.9.0-linux-amd64.tar.gz
tar zxvf helm-v3.9.0-linux-amd64.tar.gz
cp helm /usr/local/sbin

helm repo add bitnami https://charts.bitnami.com/bitnami
helm search repo bitnami

helm install bitnami/nginx --generate-name

kubectl get all   # 所有

# 下载nginx chart 查看
helm pull bitnami/nginx
tar zxvf nginx-12.0.4.tgz

```





## operator

Operator主要包含CRD和Controller两个部分：

CRD是在Kubernetes的标准对象之上构建一层直接面向应用的扩展对象，例如Mysql Operator提供的InnoDBCluster和MySQLBackup这两个扩展对象。
Controller是指自定义控制器，以Deployment的形式在Kubernetes中运行，监听CRD的配置变化，并转换成标准对象进行实现。控制器自动化智能化的灵活实现应用生命周期管理能力以及运维能力，是Site Reliability Engineering (SRE)思想在容器集群领域的一个落地实践。



## helm VS operator 



 | helm | operator
-- | -- | --
资源类型 | 标准对象，自定义对象 | 自定义对象 
编排能力 | 支持应用全生命周期管理 | 支持应用全生命周期管理 
运维能力 | 手工，较弱 | 自动故障恢复及异常处理 
设计理念 | 资源模板化，配置分离 | 复杂应用的自动化管理 
实现方式 | 传统镜像 | k8s控制器 
实现难度 | 较低 | 偏高 
灵活性 | 高度灵活修改yaml文件 | 依赖控制程序，不太灵活 
适宜场景 | 通用，普适 | 有状态服务，较复杂应用场景 

- Helm本质是包管理工具，适合于安装标准的通用的应用程序。
- Operator本质是一种设计模式，强项在于支持有状态应用的复杂生命周期管理，将运维过程中的知识和方法固化在控制器中，实现自动化运维。





# troubleshooting

## pvc Terminating 

```
PVC_NAME=xxx
kubectl patch pvc $PVC_NAME -p '{"metadata":{"finalizers":null}}'
```