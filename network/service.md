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

* 直接通过`Pod IP`访问应用

* 通过`Cluster IP`访问，然后通过`kube-proxy`代理到具体的`pod`

* 通过`DNS`解析域名为具体的`Cluster IP`（或者一组具体`Pod IP`）
  
  * 普通service
    
    ```shell
    # 单条dns记录,解析为cluster ip
    svc-name.ns.cluster.local cluster-ip
    ```
  
  * headless service
    
    ```shell
    # 多条dns记录,直接解析为pod的具体ip
    pod-namea.svc-name.ns.cluster.local 1.1.1.1
    pod-nameb.svc-name.ns.cluster.local 1.1.1.2
    pod-namec.svc-name.ns.cluster.local 1.1.1.3
    ```
    
    * 这里内部`DNS`会给具体的pod绑定一条`DNS`记录，这个特性在于，我们可以通过`pod-name`,`svc-name`等信息组装成一个域名去访问此应用【stateful set】

### 特殊的Headless

* #### 特殊点
  
  * 之所以称之为"无头"服务，是因为此`service`没有`Cluster IP`，普通的service即使不指定`Cluster IP`，它也会给分配一个【`Nodeport IP`虽然使用节点IP，但是也会分配一个`Cluster IP`】, 一组Pod没有一个公共入口，就像"无头"。
  
  * kube-proxy不会为其提供负载均衡和代理
  
  * DNS解析域名的时候返回的是一组pod的记录【参考**内部访问**】

* #### 应用场景
  
  * **有状态的服务**`StatefulSet`
    
    * `StatefulSet`会给`pod`分配一个具有稳定的、独一无二的身份标志。这个标志基于 `StatefulSet `控制器分配给每个 `Pod`的唯一顺序索引。`Pod`的名称的形式为`<statefulset name>-<ordinal index>`【其他控制器每次生成的pod-name随机】。
    
    * 对于有状态的应用，我们可以通过域名去访问，这样即使`Pod`重建了，依旧可以通过域名访问此应用，并且其存储使用`pvc`，也不会造成数据丢失。
  
  * 调用方想**自己决定负载均衡**，代理转发策略的时候，就可以使用`Headless`
