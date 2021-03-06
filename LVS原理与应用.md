# LVS 原理与应用

## 1.lvs介绍

LVS是Linux Virtual Server的简称，也就是Linux虚拟服务器， 是一个由章文嵩博士发起的自由软件项目，它的官方站点是www.linuxvirtualserver.org           

现在LVS已经是 Linux标准内核的一部分，在Linux2.4内核以前，使用LVS时必须要重新编译内核以支持LVS功能模块，但是从Linux2.4内核以后，已经完全内置了LVS的各个功能模块，无需给内核打任何补丁，可以直接使用LVS提供的各种功能。

LVS主要用于多服务器的负载均衡。

它工作在网络层，可以实现高性能，高可用的服务器集群技术。

它廉价，可把许多低性能的服务器组合在一起形成一个超级服务器。

它易用，配置非常简单，且有多种负载均衡的方法。

它稳定可靠，即使在集群的服务器中某台服务器无法正常工作，也不影响整体效果。另外可扩展性也非常好。

OSI七层模型

![|center](../master/src/lvs-1.png)

F5:BIG-IP LTM 3600
硬件，贵

![|center](../master/src/lvs-2.png)


## 2. Lvs模式
Lvs常用的三种负载均衡模式:

![|center](../master/src/lvs-3.png)

>1.Lvs nat模式

>2.Lvs ip-tun模式

>3.Lvs dr模式


### Lvs nat模式


![|center](../master/src/lvs-4.png)

![|center](../master/src/lvs-5.png)


### Lvs ip-tun模式
web服务和后端lvs都需要支持tun协议


![|center](../master/src/lvs-6.png)

![|center](../master/src/lvs-7.png)

Tunl0 隧道接口，封装数据的时候使用
Teql0 队列接口，用于流量控制


### Lvs dr模式

修改mac地址
要求在同一个物理网段


![|center](../master/src/lvs-8.png)

![|center](../master/src/lvs-9.png)


四种常用的轮叫算法


![|center](../master/src/lvs-10.png)

![|center](../master/src/lvs-11.png)

![|center](../master/src/lvs-12.png)

![|center](../master/src/lvs-13.png)

## 3.LVS十种调度算法

### 静态调度

>①rr（Round Robin）:轮询调度，轮叫调度

轮询调度算法的原理是每一次把来自用户的请求轮流分配给内部中的服务器，从1开始，直到N(内部服务器个数)，然后重新开始循环。

算法的优点是其简洁性，它无需记录当前所有连接的状态，所以它是一种无状态调度

>②wrr：weight,加权（以权重之间的比例实现在各主机之间进行调度）
 
由于每台服务器的配置、安装的业务应用等不同，其处理能力会不一样。所以，我们根据服务器的不同处理能力，给每个服务器分配不同的权值，使其能够接受相应权值数的服务请求。

>③sh:source hashing,源地址散列。(session保持)

主要实现会话绑定，能够将此前建立的session信息保留,
源地址散列调度算法正好与目标地址散列调度算法相反，它根据请求的源IP地址，作为散列键（Hash Key）从静态分配的散列表找出对应的服务器，若该服务器是可用的并且没有超负荷，将请求发送到该服务器，否则返回空。

它采用的散列函数与目标地址散列调度算法的相同。它的算法流程与目标地址散列调度算法的基本相似，除了将请求的目标IP地址换成请求的源IP地址，所以这里不一个一个叙述。

>④Dh:Destination hashing:目标地址散列(IP保持)。

把同一个IP地址的请求，发送给同一个server。

目标地址散列调度算法也是针对目标IP地址的负载均衡，它是一种静态映射算法，

通过一个散列（Hash）函数将一个目标IP地址映射到一台服务器。目标地址散列调度算法先根据请求的目标IP地址，作为散列键（Hash Key）从静态分配的散列表找出对应的服务器，若该服务器是可用的且未超载，将请求发送到该服务器，否则返回空。

### 动态调度：
>①lc（Least-Connection）：最少连接

最少连接调度算法是把新的连接请求分配到当前连接数最小的服务器，最小连接调度是一种动态调度短算法，它通过服务器当前所活跃的连接数来估计服务器的负载均衡，调度器需要记录各个服务器已建立连接的数目，当一个请求被调度到某台服务器，其连接数加1，当连接中止或超时，其连接数减一，在系统实现时，我们也引入当服务器的权值为0时，表示该服务器不可用而不被调度。

简单算法：active*256+inactive(谁的小，挑谁)

>②wlc(Weighted Least-Connection Scheduling)：加权最少连接。

加权最小连接调度算法是最小连接调度的超集，各个服务器用相应的权值表示其处理性能。

服务器的缺省权值为1，系统管理员可以动态地设置服务器的权限，加权最小连接调度在调度新连接时尽可能使服务器的已建立连接数和其权值成比例。

简单算法：（active*256+inactive）/weight【（活动的连接数+1）/除以权重】（谁的小，挑谁）

>③sed(Shortest Expected Delay)：最短期望延迟

基于wlc算法

简单算法：（active+1)*256/weight 【（活动的连接数+1）*256/除以权重】

>④nq（never queue）:永不排队（改进的sed）

无需队列，如果有台realserver的连接数＝0就直接分配过去，不需要在进行sed运算。

>⑤LBLC（Locality-Based Least Connection）：基于局部性的最少连接

基于局部性的最少连接算法是针对请求报文的目标IP地址的负载均衡调度，不签主要用于Cache集群系统，因为Cache集群中客户请求报文的布标IP地址是变化的，这里假设任何后端服务器都可以处理任何请求，算法的设计目标在服务器的负载基本平衡的情况下，将相同的目标IP地址的请求调度到同一个台服务器，来提高个太服务器的访问局部性和主存Cache命中率，从而调整整个集群系统的处理能力。
基于局部性的最少连接调度算法根据请求的目标IP地址找出该目标IP地址最近使用的RealServer，若该Real Server是可用的且没有超载，将请求发送到该服务器；
若服务器不存在，或者该服务器超载且有服务器处于一半的工作负载，则用“最少链接”的原则选出一个可用的服务器，将请求发送到该服务器。

>⑥LBLCR（Locality-Based Least Connections withReplication）：带复制的基于局部性最少链接

带复制的基于局部性最少链接调度算法也是针对目标IP地址的负载均衡，该算法根据请求的目标IP地址找出该目标IP地址对应的服务器组，按“最小连接”原则从服务器组中选出一台服务器，若服务器没有超载，将请求发送到该服务器；

若服务器超载，则按“最小连接”原则从这个集群中选出一台服务器，将该服务器加入到服务器组中，将请求发送到该服务器。

同时，当该服务器组有一段时间没有被修改，将最忙的服务器从服务器组中删除，以降低复制的程度。

## 4.与nginx类比

### lvs
LVS的负载能力强，因为其工作方式逻辑非常简单，仅进行请求分发，而且工作在网络的第4层，没有流量，所以其效率不需要有过多的忧虑。

LVS基本能支持所有应用，因为工作在第4层，所以LVS可以对几乎所有应用进行负载均衡，包括Web、数据库等。

注意：LVS并不能完全判别节点故障，比如在WLC规则下，如果集群里有一个节点没有配置VIP，将会导致整个集群不能使用。

### nginx

Nginx工作在网路第7层，所以可以对HTTP应用实施分流策略，比如域名、结构等。相比之下，LVS并不具备这样的功能，所以Nginx可使用的场合远多于LVS。

并且Nginx对网络的依赖比较小，理论上只要Ping得通，网页访问正常就能连通。

LVS比较依赖网络环境。只有使用DR模式且服务器在同一网段内分流，效果才能得到保证。

Nginx可以通过服务器处理网页返回的状态吗、超时等来检测服务器内部的故障，并会把返回错误的请求重新发送到另一个节点。

目前LVS和LDirectd 也支持对服务器内部情况的监控，但不能重新发送请求。

比如用户正在上传一个文件，而处理该上传信息的节点刚好出现故障，则Nginx会把上传请求重新发送到另一台服务器，而LVS在这种情况下会直接断掉。

Nginx还能支持HTTP和Email（Email功能很少有人使用），LVS所支持的应用在这个电商比Nginx更多。

Nginx同样能承受很高负载并且能稳定运行，由于处理流量受限于机器I/O等配置，所以负载能力相对较差。
Nginx 安装、配置及测试相对来说比较简单，因为有相应的错误日志进行提示。LVS的安装、配置及测试所花的时间比较长，因为LVS对网络以来比较大，很多时候有可能因为网络问题而配置不能成功，出现问题时，解决的难度也相对较大。

Nginx本身没有现成的热备方案，所以在单机上运行风险较大，建议KeepAlived配合使用。

另外，Nginx可以作为LVS的节点机器使用，充分利用Nginx的功能和性能。当然这种情况也可以直接使用Squid等其他具备分发功能的软件。

具体应用具体分析。如果是比较小型的网站（每日PV小于100万），用户Nginx就完全可以应对，如果机器也不少，可以用DNS轮询。LVS后用的机器较多，在构建大型网站或者提供重要服务且机器较多时，可多加考虑利用LVS。
