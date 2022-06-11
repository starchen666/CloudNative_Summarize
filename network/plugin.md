# 网络插件

## 前言

    使用service，我们可以通过Kube-proxy组件找到对应的pod ip。但是后续怎么将数据发送给对方pod呢？这就需要网络插件了，它将负责帮助我们传输数据，根据其实现原理不同，常用的插件有ovs，fannel。而他们的本质还是构建了一个overlay网络。

<img src="file:///E:/CloudNative_Summarize/network/overlay.png" title="" alt="" data-align="center">

        [参考overlay网络](https://info.support.huawei.com/info-finder/encyclopedia/zh/Overlay%E7%BD%91%E7%BB%9C.html)

## OVS

> **从上至下理解OVS**
> 
> - sdn
>   
>   - 全名软件定义网络，目的在于将网络控制、转发的功能分离。是一种实现控制可编程的新兴**网络架构**。
>     
>     <img src="file:///E:/CloudNative_Summarize/network/sdn1.jpg?msec=1654926625167?msec=1654932299092?msec=1654932465279" title="" alt="" data-align="center">
>   
>   - [SDN](https://zhuanlan.zhihu.com/p/504887869)
> 
> - openflow
>   
>   - openflow就是一种网络通信协议，它工作在第二层，通过流表来指导数据包的转发。所以sdn控制器就可以通过它的接口来操作不同的流表来控制流量转发。【**前提是控制器和交换机都支持此协议**】
>     
>     <img title="" src="file:///E:/CloudNative_Summarize/network/openflow1.png?msec=1654926607272?msec=1654932299093?msec=1654932465279" alt="" data-align="center" width="275">
>   
>   - [OpenFlow协议_闻啼鸟的博客-CSDN博客_openflow协议](https://blog.csdn.net/Tom942067059/article/details/120049835)
> 
> - ovs(Open vSwitch)
>   
>   - 可以创建虚拟交换机的开源软件，通过软件模拟交换机的功能，且支持openflow协议【理解为openFlow switch的一种具体实现】
>   
>   - [通俗说Openvswitch_ludongguoa的博客-CSDN博客_openvswitch](https://blog.csdn.net/ludongguoa/article/details/121122577)
> 
> - vxlan
>   
>   - 为构建overlay网络提供一套方法论，它将二层以太帧封装到四层的UDP报文中，并在三层网络中传输，可以将所有的机器的连接到一个虚拟二层交换机【**大二层的交换机**】。
>     
>     <img src="file:///E:/CloudNative_Summarize/network/vxlan1.png?msec=1654931342575?msec=1654932299094?msec=1654932465280" title="" alt="" data-align="center">
>   
>   - 本质上是一种**隧道技术**，通过vni标识不同隧道【**vni可以隔离网络**】，vtep设备对原始包封装，即加上vxlan的头部，udp的头部，将其传输到同一vni所在的vtep设备，然后解封装还原出原始数据。
>     
>     <img src="file:///E:/CloudNative_Summarize/network/vxlan.png?msec=1654932490190" title="" alt="" data-align="center">
>   
>   - 可以解决的问题
>     
>     - VM迁移到其他节点机器，ip可以保证不变，可以保证tcp会话，从而业务不中断。因为它是将数据包通过隧道传输，对于vm而言它并不会感知到物理网络的变化。
>     
>     - 支持海量的租户网络，因为加入了vxlan头部，其中vni有24位，所以能支持大规模不同网络隔离
>   
>   - 其他点：vxlan网络中不同子网或者同一子网内通信都会涉及ARP，以及如何确定报文需要给添加vni的标识是什么
>   
>   - [什么是VXLAN_InfoQ写作社区](https://xie.infoq.cn/article/34be5bf471a1fa025749f8366?source=app_share)

## 从openshift sdn的角度再次理解

**个人理解的openshift sdn实现**

> - openshift有两个deamset 一个是ovs，一个是sdn，它们运行在每个节点。
> 
> - 这里的sdn pod就是sdn在openshift具体实现，ovs pod就是ovs控制器
> 
> - 当ovs-pod sdn-pod被调度到节点时，会先初始化节点的网络，比如ovs会新建一个br0的虚拟网桥
> 
> - 当有新的pod生成时，kubelet就会调用cni插件（openshift-sdn），sdn将会先调用ip插件获取pause容器的ip，然后调用ovs-controller，在虚拟交换机上新建一个虚拟接口vethxxx，并将其连到pause容器的eth0接口。并设置openflow流表的规则。

**源码流程**

> [源码解读](https://mp.weixin.qq.com/s?__biz=MzA3MDg4Nzc2NQ==&mid=2652137188&idx=1&sn=98608470be8014acf8cfa1bacb219bfb&scene=21#wechat_redirect)
> 
> <img src="file:///E:/CloudNative_Summarize/network/openshift-sdn1.png" title="" alt="" data-align="center">

**openshift流量走向总结**[理解OpenShift（3）：网络之 SDN - SammyLiu - 博客园](https://www.cnblogs.com/sammyliu/p/10064450.html)

> * 内部通信
>   
>   * pod和pod之间
>     
>     * 同节点
>       
>       * 在ovs br0直接转发到对应pod【根据ovs流表】
>         
>         <img src="file:///E:/CloudNative_Summarize/network/pod-pod.png" title="" alt="" data-align="center">
>     
>     * 不同节点
>       
>       * 转发到ovs br0的vxlan0接口上，然后同宿主机的网卡转发。【**这里就用到Overlay**】
>         
>         <img title="" src="file:///E:/CloudNative_Summarize/network/pod-other-pod.png" alt="" width="485" data-align="center">
>   
>   * pod和service之间
>     
>     * 转发到ovs br0的tun0接口，先进行iptables的NAT过程【**参考kube-proxy**】，然后回到br0的vxlan0，然后转发到宿主机的网卡
>       
>       <img src="file:///E:/CloudNative_Summarize/network/pod-service.png" title="" alt="" data-align="center">
> 
> * 外部通信
>   
>   * pod访问外部
>     
>     * 转发到ovs br0的tun0接口，利用宿主机的iptables进行NAT，将源ip转为宿主机的ip，然后转发到宿主机网络接口，再转发给外网
>       
>       ![](E:\CloudNative_Summarize\network\pod-internet.png)
>   
>   * 外部访问service
>     
>     * router方式，infra节点上同样会有ovs，相比pod和pod之间之间访问，它需要先从br0的tun0接口进入到br0，然后再转给br0的vxlan0，然后经由宿主机网卡转发出去。【**router采用host-network，所以需要先进入ovs，而pod与pod之间访问的时候，pod的网络已经接入br0**】
>       
>       <img src="file:///E:/CloudNative_Summarize/network/internet-pod.png" title="" alt="" data-align="center">
>     
>     * nodeport方式，与上述方式类似【只是得到pod ip的方式不一样】。通过iptables得知具体pod ip后，iptables会转发到ovs br0的tun0接口，然后再经过br0的vxlan0转发出去。
> 
> * **总之一句话：如果涉及到NAT就需要依赖宿主机的iptables，就会先经过tun0，如果是内部通信就还会经过vxlan0走overlay网络。**
>   
>   <img src="file:///E:/CloudNative_Summarize/network/openshift-sdn.png" title="" alt="" data-align="center">

**namespace网络隔离粗浅理解**

>     从一个本地 pod 发出的所有网络流量，在它进入OVS网桥时，都会被打上它所通过的OVS端口ID相对应的 VNID。port:VNID映射会在pod创建时通过查询master上的etcd来确定。从其它节点通过VXLAN发过来的网络包都会带有发出它的pod所在项目的 VNID【oc get netnamespaces，0代表默认全部接收】。
> 
>     **当流量到达br0 vxlan0的时候会查看流表中目的pod的VNID是不是发送方pod的VNID匹配**

## 

## TODO Fannel

### TODO  kube-ovn
