## 搭建virtualbox虚拟机集群

转载于 https://www.cnblogs.com/harrychinese/p/virtualbox.html



===============================

#### VirtualBox常用网络

===============================
NetworkAddress Translation(NAT)
NAT
 是VirtualBox建立的虚拟机默认的形式. 虚拟机之间无法沟通, 虚拟机能连接外部网络. GuestOS只能看到从Host发来的数据请求,
 但主机不能访问GuestOS(可以通过端口转发来访问虚拟机). 对于大数据平台, 虚拟机之间的数据传递是必要的, 因此不能使用NAT.

NAT Network或叫做NAT Service(注意不是NAT, 而是NAT Network)
这是在VirtualBox4.3时引入的一种新的NAT类型模式。虚拟机之间可以彼此连接, 但主机只能通过端口转发来访问虚拟机. 官网上说, NAT Network下虚拟机可以访问 Internet, 但我测试是不行的. 
NAT service的工作原理和家用的路由器相似，系统群组在网络中应用这种模式来防止外部网络的直接访问，但能让系统内部通过TCP和UDP协议实现互访或访问系统外部网络。

Host-only 主机网络
则是虚拟机只能与主机彼此连接, 但不能访问外部网络. 

Bridged networking 桥接网络
会将虚拟机添加到主机所在的局域网. 不好的是, 外部网络也能连接到虚拟机, 对虚拟机的安全不利. 

Internal networking
可用于创建不同的虚拟机之间的访问机制，但是不能够访问物理主机和外部网路中的机器。

各个网络形式的特点, 看下图就能直观了.

![img](/docs/kubernetes/imgs/194640-20181012221044436-1847177416.png)

官网上说, NAT Network下虚拟机可以访问 Internet, 但我测试是不行的. 

 

===============================

#### 网络形式选择和集群规划

===============================
不管是微服务集群还是大数据集群, 我们都希望各个机器都能相互连通. 效果最好是:

` VM1`<->`VM2`, `VM1(VM2)`->`Internet`, `VM1(VM2)`<->`Host`

因办公环境IP限制, 所以不能采用 Bridged networking 桥接网络, 

1. 为了实现 VM1(VM2)->Internet, GuestOS增加网卡1(eth0), 选择 NAT . 
2. 为了实现 VM1(VM2)<->Host, GuestOS增加网卡2(eth1), 选择 Host-only 网络形式, 两个VM可关联Host的同一个Host-only网卡. 
3. 为了实现 VM1<->VM2, GuestOS增加网卡3(eth2), 选择 NAT Network. 

用下图能很清楚说明了各个网卡的作用.

![img](/docs/kubernetes/imgs/194640-20181012221428120-960138187.png)

 

一般的Linux集群都需要三个以上机器, 下面是集群的网络规划:

| 操作系统 | 机器名 | 集群内IP     | 办公网络IP   |
| -------- | ------ | ------------ | ------------ |
| Windows  | hostos | 192.168.1.1  | 10.224.1.100 |
| Linux    | vm1    | 192.168.1.11 | 无           |
| Linux    | vm2    | 192.168.1.12 | 无           |
| Linux    | vm3    | 192.168.1.13 | 无           |

 

===============================

#### 配置实战

===============================

**步骤1: 准备 VM1**
虚拟机操作系统, 我选用的 boot2docker, 因为它非常小巧(不到100M), 同时预装了docker服务, boot2docker不好的地方是,  没法为将集群中的机器加到 /etc/hosts 文件中. 

先使用如下命令创建vm1 的GuestOS母版, VM2和VM3 将都是VM1 的克隆版. 
docker-machine create --driver virtualbox vm1

![img](/docs/kubernetes/imgs/194640-20181012222223240-96564951.png)

 

docker-machine 创建的 VM 过程, 会完成很多工作, 首先在Host主机上创建一个VirtualBox Host-Only虚拟网卡(IP被设置为 192.168.99 网段), 然后创建了虚机VM1, 并为VM1配置了两个网卡, 网卡1为NAT连接形式, 网卡2为Host-Only连接形式, 网卡2已开启了DHCP服务, 也就是说VM1的IP可能会变化, 稍后需将其修改为静态IP.

 

**步骤2: 修改 VM1 eth1网卡的IP**

使用下面命令获取 VM1 的初始IP, 
docker-machine env vm1

![img](/docs/kubernetes/imgs/194640-20181012222324183-2087607869.png)

 

获得IP为192.168.99.100,  然后用 putty 登陆 VM1, user/password 为 docker/tcuser

可以使用ifconfig eth1 192.168.1.12 命令设置eth1的IP, 但这中方法在Linux重启之后, IP信息将丢失. 
不同Linux 发行版永久设置静态地址的方法不一样, 这里先说明修改 Docker2Boot VM的方法, 一般Linux的操作方法稍后介绍. 

对于Boot2Docker 虚拟机, 每次启动都会重置/etc 和 /opt 目录, 所以不能像普通发行版操作, 我们增加一个 /var/lib/boot2docker/bootlocal.sh 文件, 在其中通过 ifconfig 设置 eth1 的IP地址.

```
sudo -i   # 先切换到 root账号
vi  /var/lib/boot2docker/bootlocal.sh
```

```
#bootlocal.sh 内容如下: 
ifconfig eth1 192.168.1.11 netmask 255.255.255.0 broadcast 192.168.1.255 up
hostname vm1
```

```
vi /var/lib/boot2docker/etc/hostname
#下行是内容
vm1
```

最后, 然后关闭vm1

 

**步骤3: 修改HostOS主机上Host-Only adapter的IP**
到目前为止, vm1的 
Host-Only 连接形式的 ip 已经被修改为 192.168.1.11, 但Host主机的VirtualBox 
Host-Only虚拟网卡IP仍为 192.168.99.1, 不在一个网段, 是时候修改主机的IP为 192.168.1.1 . 

操作步骤为: VirtualBox 界面的管理-全局设定, 然后选择网络, 然后选择"仅主机HostOnly网络", 选定那个 VirtualBox Host-Only Ethernet Adapter, 修改IP地址为: 192.168.1.1, 网络掩码为 255.255.255.0, 关闭DHCP服务器, 以确保GuestOS有一个固定的IP.

上面的步骤和下面的截图都是VirtualBox 4的, VirtualBox 5.2的配置和截图稍有变化, 主要是默认情况下HostOnly的 Ipv4 是自动分配, 我们需要修改注意这个为"手工分配", 设定 IP v4的地址, 这个IP地址就是HostOS的地址, 比如设置为 192.168.1.1 

![img](/docs/kubernetes/imgs/194640-20181012222630942-1371560407.png)

 

**步骤4: 将 VM1 复制一份为 VM2**

打开Virtualbox 软件, 在UI 上关闭VM1虚拟机, 复制一份为VM2并启动它, 修改 IP 和 hostname. 

```bash
sudo -i   # 先切换到 root账号
vi  /var/lib/boot2docker/bootlocal.sh
```

```bash
#bootlocal.sh 内容如下: 
ifconfig eth1 192.168.1.12 netmask 255.255.255.0 broadcast 192.168.1.255 up
hostname vm2
```

```bash
vi /var/lib/boot2docker/etc/hostname
#下行是内容
vm2
```

 最后, 然后关闭vm2.

 

**步骤5: 将 VM1 复制一份为 VM3**

操作方法同上. 

===============================

#### CentOS Linux修改IP的方法

===============================
对于 RHEL/CentOS, 可修改 ifcfg-eth1 配置文件, 然后执行 service network restart 命令,
完整文件名为:  /etc/sysconfig/network-scripts/ifcfg-eth1
文件内容如下: 

```bash
DEVICE=eth1
IPADDR=192.168.1.11
NETMASK=255.255.255.0
GATEWAY=192.168.1.1
ONBOOT=yes
BOOTPROTO=none      
TYPE=Ethernet

NM_CONTROLLED=yes   
PREFIX=16
DEFROUTE=yes
IPV4_FAILURE_FATAL=yes
IPV6INIT=no
ARPCHECK=no
```



关于设置的说明:
ONBOOT 意思是开机即启动本网络设备
BOOTPROTO 可以取值为dhcp或none, 即是否启用dhcp

===============================

#### 参考

===============================
http://xintq.net/2014/09/05/virtualbox/ , 很好的教程     
http://xintq.net/2016/06/29/virtualbox-networking/ , 图文并茂显示各个网络的机制
https://bridge.grumpy-troll.org/2017/11/boot2docker-xhyve-dns/
https://segmentfault.com/a/1190000005650099 