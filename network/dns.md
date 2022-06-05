## 域名解析

### 应用场景

> * `pod`内部访问集群外部的域名
> 
> * `pod`内部访问集群内的域名(解析`svc`为具体`Cluster IP`)

### Linux解析域名

* 如未能从`/etc/hosts`中读取到域名对应的`IP`，则根据`/etc/resolv.conf`中配置的`DNS`服务器进行域名解析

### DNS解析流程

     [dns解析图](./dns.png)

* 文件说明：
  
  * `/etc/dnsmasq.d/origin-dns.conf`
    
    * `DNSMasq`的配置文件
  
  * `/etc/dnsmasq.d/node-dnsmasq.conf`
    
    - 定义了哪些域名结尾的会被转发到本地的`SkyDNS`
  
  * `/etc/dnsmasq.d/origin-upstream-dns.conf`
    
    * 定义了上游`DNS`服务器

* 以上三个文件在`NetworkManager`启动时候执行`/etc/NetworkManager/dispatcher.d/99-origin-dns.sh`生成【`/etc/resolv.conf`也会被此脚本重写】

* `node`和`pod`都会将域名解析请求转发到宿主机的`DNSMasq`上面，所以本地的`DNSMasq`就是负责`DNS`缓存，以及请求转发功能。
