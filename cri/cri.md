# docker containerd runc究竟是啥

### OCI标准：包括镜像标准和容器运行时标准

- 镜像标准：
  
  ```
  统一标准化容器镜像格式，让标准镜像能够在各容器软件下构建、传递及准备容器镜像运行。
  ```
  
  制作出来的镜像都长一个样，大家都能根据镜像标准去解析出相同的bundle包

- 运行时标准：
  
  ```
  旨在指定容器的配置、执行环境和生命周期。
  容器的配置被指定为config.json支持的平台，并详细说明了启用容器创建的字段。
  指定执行环境是为了确保在容器内运行的应用程序在运行时之间具有一致的环境以及为容器生命周期定义的常见操作
  ```
  
  规定了不同运行时工具如何运行这个bundle包

- RunC: 实现了OCI的运行时标准
  
  既然你规定了启动的配置在config.json，那我runc启动容器进程的时候就去读取你这配置，进行我自己的解析。我容器进程需要rootfs，我也根据你的配置去加载。

### docker创建容器（从上到下）

#### dockerd

- docker的守护进程，负责响应客户端请求。

#### containerd

- **为了oci，从dockerd中剥离出来的开源组件**

- 一个负责容器运行时的docker组件，作为守护进程存在，负责响应dockerd的调用【当然也可以响应其他调用方，比如**cri-containerd**】，high-level容器运行时实现（为容器运行时提供支持）

- 推拉镜像

- 解析镜像为runc构建bundle包，调用runc接口

- 管理容器生命周期

- 管理容器网络

- 管理镜像存储，容器数据存储

#### docker-shim

- 作为容器进程的父进程，调用runc并管理runc创建的进程（做容器状态收集，维持 stdin 等 fd 打开等工作）

- 存在的意义:runc 启动完容器后本身会直接退出, containerd-shim 则会成为容器进程的父进程, 负责收集容器进程的状态, 上报给 containerd, 并在容器中 pid 为 1 的进程退出后接管容器中的子进程进行清理, 确保不会出现僵尸进程。如果 containerd 作为父进程挂掉，整个宿主机上所有的容器都得退出了，而引入 containerd-shim 就可以规避（shim 会 re-parent 到 systemd 这样的 1 号进程上），提供live-restore功能。这里需要注意systemd的 MountFlags=slav

#### runc

- **为了oci，从docker libcontainer弄出来开源容器运行时**

- 负责创建具体的"进程"(容器)，low-level容器运行时实现（真正提供创建容器的能力）

- 这里就涉及到容器视图隔离namespace,资源限制cgroups,文件系统rootfs

### CRI

    kubernetes提出的容器运行时接口，充当的角色就是将k8s调用转换为对底层的oci的调用

* 定义了三类接口
  
  * 针对容器
  
  * 针对PodSandBox
    
    * 抽出了容器和pod的相似的部分，目的在于将pod的实现交给不同oci去实现，因为pod只是k8s中编排的一个概念
  
  * 针对镜像

* 不同的high-level容器运行时，只需要实现对应的接口即可实现k8s对容器运行时的调用，而cri-o则是为了减少调用链路直接让low-level容器运行时去实现它规定的接口完成调用（这也称得上一种high-level运行时的实现，兼容了cri和oci）。

#### 最后一点理解

    总而言之，整个过程就是慢慢向简单纯粹靠近，拥抱变化，尽可能都能插件化，做到组件都能被替换。所以我们可以看到下图中对low-level的运行时在不断发生变化。

[cri](./cri.png)

### 参考链接

[OCI和RunC](https://zhuanlan.zhihu.com/p/438353377)

[Docker、Containerd、RunC 间的联系和区别](https://zhuanlan.zhihu.com/p/451655692)

[docker、oci、runc以及kubernetes梳理](https://blog.csdn.net/chenrui310/article/details/102623524/)

[Docker，containerd，CRI，CRI-O，OCI，runc](https://zhuanlan.zhihu.com/p/490585683)

[K8S Runtime CRI OCI contained dockershim 理解](https://blog.csdn.net/u011563903/article/details/90743853)

[开放容器标准(OCI) 内部分享](https://xuanwo.io/2019/08/06/oci-intro/)
