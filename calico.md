

# 架构



Calico是一个纯3层的数据中心网络方案，Calico的原理是通过修改每个主机节点上的iptables和路由表规则，实现容器间数据路由和访问控制，并通过ETCD协调节点配置信息的。因此Calico服务本身和许多分布式服务一样，需要运行在集群的每一个节点上



![calico](/Users/chenjunkai/work/kubernetes-training/calico.assets/calico1.png)



# 主要组件



- Felix：Calico Agent，跑在每台需要运行Workload的节点上，主要负责配置路由及ACLs等信息来确保Endpoint的连通状态；
- etcd：分布式键值存储，主要负责网络元数据一致性，确保Calico网络状态的准确性；
- BGP Client（BIRD）：主要负责把Felix写入Kernel的路由信息分发到当前Calico网络，确保Workload间的通信的有效性；
- BGP Route Reflector（BIRD）：大规模部署时使用，摒弃所有节点互联的mesh模式，通过一个或者多个BGP Route Reflector来完成集中式的路由分发。