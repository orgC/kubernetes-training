# 部署应用

## 应用部署

```

kubectl create deployment demo --image=httpd --port=80

kubectl scale --replicas=2 deploy/demo
kubectl get pod -owide 

kubectl exec -it demo-654c477f6d-8wndg sh

kubectl describe pod 

curl 10.244.104.140 <pod-ip>
curl <svc-ip>

```



## 节点不可调度

```
kubectl cordon node worker1.myk8s.example.com

kubectl uncordon worker1.myk8s.example.com
```





## label调度

```

kubectl get no --show-labels

kubectl label no worker1.myk8s.example.com mylabel=abc

kubectl edit deploy/demo
...
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: demo
    spec:
      nodeSelector:      # 添加nodeselector 
        mylabel: abc
      containers:
      - image: httpd
        imagePullPolicy: Always
        name: httpd
        ports:
... 


kubectl label no worker1.myk8s.example.com mylabel-

kubectl get no --show-labels

kubectl scale --replicas=4 deploy/demo   # 此时pod  pending

kubectl edit deploy/demo  # 取消 node selector

```



## 污点与容忍

用于保障Pod不被调度到不合适的Node上，其中Taints适用于Node，tolerations适用于Pod

目前支持的taint类型

1. NoSchedule：新的Pod不调度到该Node上，不影响正在运行的Pod
2. PreferNoSchedule： soft版本的NoSchedule，尽量不调度到该Node上
3. NoExecute：新的Pod不会调度到该Node，正在运行的Pod也会被删除，不过可以添加一个时间（tolerationSeconds）



一个容忍度和一个污点相“匹配”是指它们有一样的键名和效果，并且：

* 如果 operator 是 Exists （此时容忍度不能指定 value），或者
* 如果 operator 是 Equal ，则它们的 value 应该相等



### 查看污点

```
[root@master1 ~]# kubectl describe no worker1.myk8s.example.com
Name:               worker1.myk8s.example.com
Roles:              <none>
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=worker1.myk8s.example.com
                    kubernetes.io/os=linux
Annotations:        kubeadm.alpha.kubernetes.io/cri-socket: /var/run/dockershim.sock
                    node.alpha.kubernetes.io/ttl: 0
                    projectcalico.org/IPv4Address: 192.168.26.142/24
                    projectcalico.org/IPv4VXLANTunnelAddr: 10.244.104.128
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Mon, 06 Jun 2022 23:15:18 +0800
Taints:             <none>
```



### Lab1: NoSchedule

```
kubectl create deployment demo --image=httpd --port=80
kubectl scale --replicas=2 deploy/demo

kubectl get pod -A -o wide  | grep worker1 # 每个节点上都有pod

# 添加污点 NoSchedule

kubectl taint nodes worker1.myk8s.example.com key1=value1:NoSchedule  # 此时worker1 上的pod没有被驱逐走
kubectl scale --replicas=4 deploy/demo  # 新的pod不会被调度到worker1上

# 容忍污点

参考以下实例，在deploy中容忍key1:value1 
kubectl edit deploy/demo 
...
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: demo
    spec:
      tolerations:
      - key: "key1"
        operator: "Equal"
        value: "value1"
        effect: "NoSchedule"
      containers:
      - image: httpd
        imagePullPolicy: Always
        name: httpd
... 


或者按照以下格式修改
...
      tolerations:
      - effect: NoSchedule
        key: key1
        operator: Exists
...



# 删除污点
kubectl taint nodes worker1.myk8s.example.com key1:NoSchedule-

# 清理环境
kubectl delete deploy/demo
```



### Lab2：PreferNoSchedule

```
kubectl create deployment demo --image=httpd --port=80
kubectl scale --replicas=2 deploy/demo

kubectl taint nodes worker1.myk8s.example.com key1=value1:PreferNoSchedule

kubectl scale --replicas=4 deploy/demo  # 后续逐渐增加pod数量，可以看到最终还是有pod会被调度到node1 上的

# 删除污点
kubectl taint nodes worker1.myk8s.example.com key1:PreferNoSchedule-

kubectl describe node worker1.myk8s.example.com | grep -i taint

# 清理环境

kubectl delete deploy/demo
```



### Lab3： NoExecute



```
kubectl create deployment demo --image=httpd --port=80
kubectl scale --replicas=2 deploy/demo

kubectl taint nodes worker1.myk8s.example.com key1=value1:NoExecute

kubectl get pod -o wide # 可以看到pod从worker1节点上被驱离 

kubectl edit deploy/demo 
按照以下格式修改
...
      tolerations:
      - effect: NoSchedule
        key: key1
        operator: Exists
...

# 清理环境
kubectl taint nodes worker1.myk8s.example.com key1:NoExecute-
kubectl delete deploy/demo
```

### Lab4： tolerationSeconds

如果 Pod 存在一个 effect 值为 `NoExecute` 的容忍度指定了可选属性 `tolerationSeconds` 的值，则表示在给节点添加了上述污点之后， Pod 还能继续在节点上运行的时间(单位为秒)

```

kubectl create deployment demo --image=httpd --port=80
kubectl scale --replicas=2 deploy/demo


kubectl edit deploy/demo 
按照以下格式修改
...
      tolerations:
      - effect: NoExecute
        key: key1
        operator: Exists
        tolerationSeconds: 30
... 


```





### 使用场景

* 专用节点：如果你想将某些节点专门分配给特定的一组用户使用，你可以给这些节点添加一个污点（即， kubectl taint nodes nodename dedicated=groupName:NoSchedule）， 然后给这组用户的 Pod 添加一个相对应的 toleration（通过编写一个自定义的 准入控制器，很容易就能做到）。 拥有上述容忍度的 Pod 就能够被分配到上述专用节点，同时也能够被分配到集群中的其它节点。 如果你希望这些 Pod 只能被分配到上述专用节点，那么你还需要给这些专用节点另外添加一个和上述 污点类似的 label （例如：dedicated=groupName），同时 还要在上述准入控制器中给 Pod 增加节点亲和性要求上述 Pod 只能被分配到添加了 dedicated=groupName 标签的节点上
* 配备了特殊硬件的节点：在部分节点配备了特殊硬件（比如 GPU）的集群中， 我们希望不需要这类硬件的 Pod 不要被分配到这些特殊节点，以便为后继需要这类硬件的 Pod 保留资源。 要达到这个目的，可以先给配备了特殊硬件的节点添加 taint （例如 kubectl taint nodes nodename special=true:NoSchedule 或 kubectl taint nodes nodename special=true:PreferNoSchedule)， 然后给使用了这类特殊硬件的 Pod 添加一个相匹配的 toleration。 和专用节点的例子类似，添加这个容忍度的最简单的方法是使用自定义 准入控制器。 比如，我们推荐使用扩展资源 来表示特殊硬件，给配置了特殊硬件的节点添加污点时包含扩展资源名称， 然后运行一个 ExtendedResourceToleration 准入控制器。此时，因为节点已经被设置污点了，没有对应容忍度的 Pod 不会被调度到这些节点。但当你创建一个使用了扩展资源的 Pod 时， ExtendedResourceToleration 准入控制器会自动给 Pod 加上正确的容忍度， 这样 Pod 就会被自动调度到这些配置了特殊硬件件的节点上。 这样就能够确保这些配置了特殊硬件的节点专门用于运行需要使用这些硬件的 Pod， 并且你无需手动给这些 Pod 添加容忍度



## 亲和与反亲和

nodeAffinity就是节点亲和性，相对应的是Anti-Affinity，就是反亲和性，这种方法比nodeSelector更加灵活，它可以进行一些简单的逻辑组合了，不只是简单的相等匹配。 调度可以分成`软策略`和`硬策略`两种方式，**软策略** 就是如果你没有满足调度要求的节点的话，POD 就会忽略这条规则，继续完成调度过程，说白了就是满足条件最好了，没有的话也无所谓了的策略；**硬策略**就比较强硬了，如果没有满足条件的节点的话，就不断重试直到满足条件为止，简单说就是你必须满足我的要求，不然我就不干的策略。 nodeAffinity就有两上面两种策略：`preferredDuringSchedulingIgnoredDuringExecution`和`requiredDuringSchedulingIgnoredDuringExecution`，前面的就是软策略，后面的就是硬策略



### Node亲和



```
kubectl get nodes --show-labels
kubectl label nodes worker1.myk8s.example.com disktype=ssd
kubectl get nodes --show-labels


cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: demo
  name: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo
  strategy: {}
  template:
    metadata:
      labels:
        app: demo
    spec:
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 1
            preference:
              matchExpressions:
              - key: disktype
                operator: In
                values:
                - ssd
      containers:
      - image: httpd
        name: httpd
        ports:
        - containerPort: 80
        resources: {}
status: {}
EOF

# 扩展pod实例数量，此时发现所有的pod都是在worker1上 
kubectl scale --replicas=3 deploy/demo

```



### pod 亲和 

根据 POD 之间的关系进行调度, 我们希望with-pod-affinity和busybox-pod能够就近部署，而不希望和node-affinity-pod部署在同一个拓扑域下面，参考以下方案

```
# 创建 busybox pod 
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: busybox-pod
  labels: 
    app: busybox-pod
spec:
  containers:
  - name: busybox
    image: busybox:1.28
    args:
    - sleep
    - "1000000"
EOF

# 查看pod label
kubectl get pod --show-labels

# 部署应用 nginx, 等待节点部署后，此时再看，会发现 nginx 与busybox 在同一个节点上
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: with-pod-affinity
  labels:
    app: pod-affinity-pod
spec:
  containers:
  - name: with-pod-affinity
    image: nginx
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - busybox-pod
        topologyKey: kubernetes.io/hostname
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - node-affinity-pod
          topologyKey: kubernetes.io/hostname
EOF

# 将busybox 节点设置为不可调度，然后重新部署nginx，此时nginx 处于pending 状态
kubectl delete pod with-pod-affinity
kubectl cordon worker2.myk8s.example.com

重新部署nginx 
```

