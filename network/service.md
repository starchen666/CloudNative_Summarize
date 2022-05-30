## 外部访问

    从外部访问集群内的应用，有以下几种方式：

* 以代理的方式，客户端访问`LB`，`LB`将请求转发到集群
  
  * 个人理解：这样就是在集群外面加了个负载均衡器，客户端只需要访问这个入口的`VIP`就行，参考[router.md](./router/router.md)

* 暴露内部`service`
  
  * 个人理解：将多个`pod`包裹起来，加上一层`service`的概念，那我们只需要将`service`暴露给外部【说明：外部指的是`pod`网段外，这里只讨论集群外的情况】
  
  * `service`有`Cluster IP`,`NodePort`,`Load Balancer`三种类型，而`Cluster IP`是集群内的虚拟`IP`，并没有网络实体与之对应，因此并不能对外提供服务。所以只能选择类型为`NodePort`或`Load Balancer`的`service`。
    
    * `Load Balancer`：只能运行特定的云平台
    
    * `NodePort`：以`NodeIP:NodePort`形式访问集群，其中`Node`可以为任何一个集群的工作节点，`Node`上会开放此端口，借助于`kube-proxy`将请求转发到具体`pod`

* *TODO*: ingress

## 内部访问

    从内部访问集群内的应用，有以下几种方式：

### Cluster IP

    

### NodePort

### Load Balancer

### 特殊的Headless
