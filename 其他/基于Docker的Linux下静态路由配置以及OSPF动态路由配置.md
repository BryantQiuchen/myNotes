# 基于Docker的Linux下静态路由配置以及OSPF动态路由配置

## 1 引言

路由器是我们当前日常生活中进行联网的重要手段，提供路由和转发功能，决定数据包从源端到目的端口所经过的路径被称为路由，从路由器输入端的数据包移送到另一路由器输出端称为转发。通过路由器的路由和转发功能，能够实现不同网络之间的互联通信。由于路由器通常连接多个网络或者连接着某一网络中的多个设备，器转发操作并不能像两者之间一一对应这么简单，并且伴随着网络拓扑及网络或设备 ip 的变化，其转发应当能实现 较为灵活的映射变化，因此路由表便应用而生。路由器根据路由表来查找目的地址的下一跳，并将该数据(或者处理后的数据)发给下一跳的设备。因此，路由表的信息形成及及时维护更新对于路由器有效实现转发来说至关重要。

路由表的形成有两种方式，静态和动态两种方式，静态路由通常是人为的去修改路由表增加或减少路由条目，确定网络拓扑后进行配置，通常不会再发生变化，对于小规模的网络，能够很简单的进行配置，但当网络规模较大或者网络拓扑经常发生变化时，这种静态路由配置的方式效率就会很低，在现如今的网络中使用的情况已经很少了。而动态路由方式是按照事先规定的策略自动生成所需要的路由信息，较为常见的动态路由协议有RIP、OSPF、BGP等。以 OSPF 为例，它属于链路状态路由协议。OSPF 提出了区域的概念，每个区域中所有路由器维护着一个相同的链路状态数据库，OSPF利用所维护的链路 状态数据库，通过最短路径优先算法计算得到路由表。

在本次网络仿真中，分别对路由器进行了静态路由和OSPF动态路由配置，在Linux系统中进行路由配置，使Linux系统能够实现路由功能。

## 2 实验过程

本次实验选用了6个路由器节点以及3个主机节点，如果使用Linux虚拟机，需要9个Linux虚拟机同时运行，对于本机来说负荷过重，所以使用了Docker容器，通过Docker容器下的ubuntu系统进行仿真，得益于其轻量级的特点，我们使用多容器同时工作时能够保证性能，除此之外，使用Docker能够使用其内部的功能进行容器间通信，对于搭建网络拓扑较为容易，进行选用Docker平台来完成此次的网络仿真。对于动态路由的配置，本实验选用的是Quagga软件，Quagga是一个路由软件工具，实现了在Unix平台（FreeBSD, Linux, Solaris 和 NetBSD）上的各个路由协议，包括： OSPFv2, OSPFv3, RIP v1 and v2, RIPng and BGP-4 。

### 2.1 网络拓扑

本次实验所使用的拓扑图如图1所示，每个端口号的IP地址如下所示，其子网掩码均为255.255.255.0。

![未命名文件 (2)](/Users/ryan/Desktop/未命名文件 (2).jpg)

图1 实验所用拓扑图

### 2.2 Docker容器的创建

1） Quagga镜像的实现

由于Docker Hub中的ubuntu镜像为极简版本的ubuntu系统，所以首先需要创建带有quagga软件以及常用的网络命令的quagga镜像。

sudo docker pull ubuntu             #下载ubuntu镜像

sudo docker run -itd --privileged=true --name temp ubuntu:latest /bin/bash           #生成临时的容器temp

sudo docker exec -it temp bash                             #进入temp容器中

apt-get install update                                               # 容器中更新安装源

apt install quagga                    #以相同的方式安装 :net-tools，vim，screen，inetutils-ping软件

在/etc/quagga 下创建  zebra.conf bgpd.conf ospfd.conf 等 文件（touch）并编写内容（见图）

在/run 目录下创建quagga文件夹执行

chown -R quagga.quagga /run/quagga

退出temp容器后执行

sudo docker commit temp  quagga       #生成自己的quagga镜像



zebra.conf:

    	hostname Router
    	password zebra
    	enable password zebra
    	log stdout

bgpd.conf:

    	hostname Router
    	password zebra
    	log stdout

ospfd.conf:

    	hostname Router
    	password zebra
    	log stdout

ripd.conf:

    	hostname Router
    	password zebra
    	log stdout

2）创建对应容器（只列出rt1）

我们需要9个Docker容器，都为装有Quagga路由软件的镜像文件。命令如下：

sudo docker run -itd --network=none --name=rt1 --privileged=true quagga

sudo docker run -itd --network=none --name=rt2 --privileged=true quagga

sudo docker run -itd --network=none --name=rt3 --privileged=true quagga

sudo docker run -itd --network=none --name=rt4 --privileged=true quagga

sudo docker run -itd --network=none --name=rt5 --privileged=true quagga

sudo docker run -itd --network=none --name=rt6 --privileged=true quagga

sudo docker run -itd --network=none --name=h1 --privileged=true ubuntu

sudo docker run -itd --network=none --name=h2 --privileged=true ubuntu

sudo docker run -itd --network=none --name=h3 --privileged=true ubuntu

结果如下：

![截屏2021-06-07 下午1.50.29](/Users/ryan/Library/Application Support/typora-user-images/截屏2021-06-07 下午1.50.29.png)

使用 docker ps可以查看正在运行中的容器

![截屏2021-06-07 下午1.53.24](/Users/ryan/Library/Application Support/typora-user-images/截屏2021-06-07 下午1.53.24.png)

### 2.3 容器间通信

容器创建成功后，我们应当完成拓扑的构建，配置容器的IP地址和对应接口的链接关系。

1）获取容器的进程号，命令如下：（以h1为例）

PID1=$(sudo docker inspect -f '{{.State.Pid}}' h1)

![截屏2021-06-07 下午3.08.29](/Users/ryan/Library/Application Support/typora-user-images/截屏2021-06-07 下午3.08.29.png)

2）将进程网络命名空间恢复到主机目录:（以$PID1为例）

mkdir /var/run/netns/

sudo ln -s /proc/$PID1/ns/net /var/run/netns/$PID1

![截屏2021-06-07 下午3.10.04](/Users/ryan/Library/Application Support/typora-user-images/截屏2021-06-07 下午3.10.04.png)

3）配置容器间的通信关系、IP，以及主机的默认网关

以h1的v1端口和rt1的v2端口为例，（其余容器同）命令如下：

sudo ip link add v1 type veth peer name v2
sudo ip link set v1 netns $PID1
sudo ip link set v2 netns $PID2
sudo ip netns exec $PID1 ip link set v1 up
sudo ip netns exec $PID1 ip addr add 192.168.11.1/24 dev v1
sudo ip netns exec $PID2 ip link set v2 up
sudo ip netns exec $PID2 ip addr add 192.168.11.2/24 dev v2

配置结果如下，以rt1为例：

使用ip addr查看端口，分别标识了v2和v3端口的端口号

![截屏2021-06-07 下午3.05.42](/Users/ryan/Library/Application Support/typora-user-images/截屏2021-06-07 下午3.05.42.png)

使用ping命令来测试其直连网络的连通性，ping通表示成功

![截屏2021-06-07 下午3.13.15](/Users/ryan/Library/Application Support/typora-user-images/截屏2021-06-07 下午3.13.15.png)

配置主机h1、h2、h3的默认网关，命令如下：

sudo ip netns exec $PID1 ip route add default via 192.168.11.2

sudo ip netns exec $PID4 ip route add default via 192.168.66.2

sudo ip netns exec $PID7 ip route add default via 192.168.55.2

### 2.4 静态路由配置

在我们完成网络拓扑的搭建后，直连的每个容器是可以互相通信的，但非直连的主机或路由器是没有办法进行通信的，这时候就需要对每个路由器进行静态路由配置，命令如下：（以rt3到192.168.54.0子网为例）

ip route add 192.168.54.0/24 via 192.168.34.2

之后使用ip route查看路由信息，结果如下：

![截屏2021-06-05 下午11.09.52](/Users/ryan/Library/Application Support/typora-user-images/截屏2021-06-06 下午12.21.50.png)

当所有路由器的静态路由都完成配置后，所有网段都可以互相ping通：（以h1到h3为例）

![截屏2021-06-07 下午3.23.04](/Users/ryan/Library/Application Support/typora-user-images/截屏2021-06-07 下午3.23.04.png)

在完成静态路由配置实验后，需要对路由条目进行删除，否则OSPF协议没有任何意义，命令如下：（以rt3为例）

ip route del 192.168.54.0/24 via 192.168.34.2

### 2.5 OSPF协议配置

本实验中划分了4个area，路由器相连的区域为area 0，路由器rt1、rt6、rt5连接主机h1、h2、h3的区域分别为area 1、area 2、area 3。

OSPF配置的命令如下：（只举出rt1，rt2，其余路由器同）

开启Quagga的zebra和ospf服务：

nohup zebra &
nohup ospfd &

可以使用 netstat -tunlp查看进程

![截屏2021-06-07 下午3.31.35](/Users/ryan/Library/Application Support/typora-user-images/截屏2021-06-07 下午3.31.35.png)

进行ospf配置，命令如下：

vtysh
configure terminal
router ospf
router-id 192.168.11.2
network 192.168.11.0/24 area 1 

network 192.168.12.0/24 area 0 

do write

![截屏2021-06-07 下午3.32.31](/Users/ryan/Library/Application Support/typora-user-images/截屏2021-06-07 下午3.32.31.png)

命令show ip ospf neighbor可以查看ospf邻居，由于现只配置了rt1所以显示为空：

![截屏2021-06-07 下午3.33.51](/Users/ryan/Library/Application Support/typora-user-images/截屏2021-06-07 下午3.33.51.png)

同样的对rt2进行配置，配置好后对查看rt2的ip route 和ospf neighbor，结果如下：

![截屏2021-06-07 下午3.36.17](/Users/ryan/Library/Application Support/typora-user-images/截屏2021-06-07 下午3.36.17.png)

表明neighbor ID为192.168.11.2的rt1为rt2的邻居

![截屏2021-06-07 下午3.37.11](/Users/ryan/Library/Application Support/typora-user-images/截屏2021-06-07 下午3.37.11.png)

路由表中可以看到到192.168.11.0/24的路由条目。

进入h1，若h1能够ping通rt2，说明成功：

![截屏2021-06-07 下午3.38.43](/Users/ryan/Library/Application Support/typora-user-images/截屏2021-06-07 下午3.38.43.png)

对所有的路由器作上述配置后，就完成了ospf动态路由协议的配置，这时我们进入rt1查看rt1的路由表，可以清楚的看到抵达每一个子网的路由条目，结果如下：



![截屏2021-06-07 下午3.42.00](/Users/ryan/Library/Application Support/typora-user-images/截屏2021-06-07 下午3.42.00.png)

分别进入h1、h2、h3，若能够互相ping通，则说明实验成功，结果如下：

h1 ping h2

![截屏2021-06-07 下午3.44.39](/Users/ryan/Library/Application Support/typora-user-images/截屏2021-06-07 下午3.44.39.png)

h1 ping h3

![截屏2021-06-07 下午3.44.54](/Users/ryan/Library/Application Support/typora-user-images/截屏2021-06-07 下午3.44.54.png)

h2 ping h1

![截屏2021-06-07 下午3.45.56](/Users/ryan/Library/Application Support/typora-user-images/截屏2021-06-07 下午3.45.56.png)

h2 ping h3

![截屏2021-06-07 下午3.46.09](/Users/ryan/Library/Application Support/typora-user-images/截屏2021-06-07 下午3.46.09.png)

h3 ping h1

![截屏2021-06-07 下午3.46.45](/Users/ryan/Library/Application Support/typora-user-images/截屏2021-06-07 下午3.46.45.png)

h3 ping h2

![截屏2021-06-07 下午3.46.57](/Users/ryan/Library/Application Support/typora-user-images/截屏2021-06-07 下午3.46.57.png)

## 3 总结

本实验是借助Docker容器在Linux系统下分别进行静态路由配置和OSPF动态路由配置，在Docker平台上实现一个ubuntu容器是很简单的，但多个容器之间的通信却不是很好完成，通过这次实验，对于Docker跨容器通信有了更加深刻的认识，同时在Linux系统下使用Quagga进行网络配置以及一些常用的命令有了深刻的认识，同时在网络配置的过程中，对于OSPF协议的认识也更加深刻了



