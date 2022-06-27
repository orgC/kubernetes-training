

# 架构



Calico是一个纯3层的数据中心网络方案，Calico的原理是通过修改每个主机节点上的iptables和路由表规则，实现容器间数据路由和访问控制，并通过ETCD协调节点配置信息的。因此Calico服务本身和许多分布式服务一样，需要运行在集群的每一个节点上



![calico](/Users/chenjunkai/work/kubernetes-training/calico.assets/calico1.png)



# 主要组件



- Felix：Calico Agent，跑在每台需要运行Workload的节点上，主要负责配置路由及ACLs等信息来确保Endpoint的连通状态；
- etcd：分布式键值存储，主要负责网络元数据一致性，确保Calico网络状态的准确性；
- BGP Client（BIRD）：主要负责把Felix写入Kernel的路由信息分发到当前Calico网络，确保Workload间的通信的有效性；
- BGP Route Reflector（BIRD）：大规模部署时使用，摒弃所有节点互联的mesh模式，通过一个或者多个BGP Route Reflector来完成集中式的路由分发。





# Calico两种网络模式

  Calico本身支持多种网络模式，从`overlay`和`underlay`上区分。`Calico overlay` 模式，一般也称Calico IPIP或VXLAN模式，不同Node间Pod使用IPIP或VXLAN隧道进行通信。`Calico underlay` 模式，一般也称calico BGP模式，不同Node Pod使用直接路由进行通信。在overlay和underlay都有`nodetonode mesh`(全网互联)和`Route Reflector`(路由反射器)。如果有安全组策略需要开放IPIP协议；要求Node允许BGP协议，如果有安全组策略需要开放TCP 179端口；官方推荐使用在Node小于100的集群





CMD

安装client： https://github.com/projectcalico/calico/releases/download/v3.23.1/calicoctl-linux-amd64 

```


[root@master1 ~]# DATASTORE_TYPE=kubernetes KUBECONFIG=~/.kube/config calicoctl node status
Calico process is running.

IPv4 BGP status
+----------------+-------------------+-------+------------+-------------+
|  PEER ADDRESS  |     PEER TYPE     | STATE |   SINCE    |    INFO     |
+----------------+-------------------+-------+------------+-------------+
| 192.168.26.142 | node-to-node mesh | up    | 2022-06-16 | Established |
| 192.168.26.143 | node-to-node mesh | up    | 2022-06-16 | Established |
| 192.168.26.144 | node-to-node mesh | up    | 04:35:10   | Established |
+----------------+-------------------+-------+------------+-------------+

```



# 流程



![img](https://upload-images.jianshu.io/upload_images/13360402-3ddc5a436ce6fa16.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)





# 抓包测试方案





```

kubectl create deploy tools-v2 --image quay.io/junkai/tools:v2

kubectl create deployment demo --image quay.io/junkai/demo:1.0 
kubectl scale --replicas=3 deploy/demo 



```

