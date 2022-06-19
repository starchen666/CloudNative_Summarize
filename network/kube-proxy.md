## Kube-Proxy

### 背景

    无论是`Cluster IP`还是`NodePort`，都会对应一组`EndPoint`。访问`Cluster IP`或者`NodePort`两个类型的`service`时，都需要负载均衡到具体的`pod`，`kube-proxy`就是充当这个角色【实现方式`userspace`,`iptables`,`ipvs`】。`kube-proxy`会`watch` `api-server`中`EndPoints`和`service`的变化。

### userspace

    `kube-proxy`运行在用户空间，监听一个特定的端口，并创建特定的`iptables`规则【目的在于将请求转发给`kube-proxy`】，当有请求进来时，先进入内核`iptables`，然后在回到用户空间的负载均衡器，由后者负责`pod`的选择，并建立到`pod`的连接，完成代理转发。

    [userspace实现原理图](./userspace.jpeg)

    优点：当pod不可用时，可重试其他pod

    缺点：2次用户态和内核态的交互【进去一次，出来一次】，性能较后面两种实现损耗更大

### iptables

    `kube-proxy`负责实时维护到每个`pod`的`iptables`转发规则，基于内核空间的`netfilter`框架实现，将发向`Cluster IP`的请求根据一系列规则链重定向到一个具体的`Pod IP`。

- 主要链说明
  
  - `KUBE-NODEPORTS`：`NodePort`类型的入口链
  
  - `KUBE-SERVICES`：`Cluster IP`类型的入口链
  
  - `KUBE-SVC-*`：负载均衡的作用，均匀的将数据包分发给`KUBE-SEP-*`
  
  - `KUBE-SEP-*`：将流量转给具体的`pod`

[iptables实现原理图](./iptables.jpeg)

优点：在内核空间完成转发，性能较userspace更高（但这里会涉及到`SNAT`和`DNAT`）

缺点：没有灵活的负载均衡策略，当pod不可用时候不能重试（指的是`pod`存活,但不能正常提供服务,[参考此图](#live_readness)），`iptables`底层是链表实现，所以服务数量较多时，`iptables`将会变得很大，更新`iptables`、转发效率都会下降

[两种检测探针](./live_readness.png)

### ipvs

    同iptables类似，`kube-proxy`负责实时维护到每个`pod`的`ipvs rules`。`ipvs`也是基于内核空间的`netfilter`框架实现，但采用了`hash table`来存储规则，因此在规则较多的情况下，`ipvs`相对`iptables`转发效率更高。除此以外，`ipvs`支持更多的负载均衡算法。如果要设置`kube-proxy`为`ipvs`模式，必须在操作系统中安装`ipvs`内核模块

[ipvs实现原理图](./ipvs.jpeg)
