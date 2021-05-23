

虚拟化网络专题（2） - Linux虚拟网络设备(bridge & veth pair)

[TOC]

# Linux虚拟网络设备(bridge & veth pair)

## bridge

一般的局域网交换中都会使用到交换机，Linux系统中的网桥就是相当于本系统内部的一个交换机，可以将连接到本网桥的网络设备进行二层联通。

下面是一个局域网中使用交换机实现网络连通：

![image-20210516154535085](D:\GitRepoWork\Github\psbec\CloudNative-Network\Linux\image\image-20210516154535085.png)



Linux bridge原理与计算机相同，bridge可以将多个namespace的网络空间连通：

![image-20210516154931457](D:\GitRepoWork\Github\psbec\CloudNative-Network\Linux\image\image-20210516154931457.png)

我们看下网桥的创建和使用，linux下提供了`brctl`命令管理网桥。



**实验目标**：创建网桥`br0`，并分配IP地址`172.16.0.1`作为连接本网桥设备的网关地址。

**操作过程**：

**Step1**：创建网桥：

```
[root@worker2 ~]# brctl addbr br0
[root@worker2 ~]# brctl show
bridge name     bridge id               STP enabled     interfaces
br0             8000.000000000000       no
```



**Step2**：分配IP地址，并启用此网桥：

```
[root@worker2 ~]# ip addr add 172.16.0.1/24 dev br0
[root@worker2 ~]# ifconfig br0 up
[root@worker2 ~]# ifconfig br0
br0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.16.0.1  netmask 255.255.255.0  broadcast 0.0.0.0
        inet6 fe80::145a:1cff:fe94:fe47  prefixlen 64  scopeid 0x20<link>
        ether 16:5a:1c:94:fe:47  txqueuelen 1000  (Ethernet)
```



到此，我们就可以把这个网桥当做系统中的一个交换机设备使用了，该交换机同时提供了网关功能。



## veth pair

物理世界中，PC机需要通过一个网线连接交换机，软件世界中如何将namespace与内核的bridge连接起来呢？答案就是通过veth pair，我们可以把veth pair想象成一个连接了两个网卡的设备，一端连接namespace，另外一端连接bridge。

`veth pair`设备有个特点，每创建一个veth pair都对应了两个虚拟网络设备，并且这两个网络设备是联通的，也就是从一个设备发送数据，就会从另外一个设备收到。就是通过这个特性，可以实现namespace和bridge之间的连接。

用户可以使用`ip link`命令增加`veth pair`设备，下面的范例中该`veth pair`设备的两个网卡为`veth20`和`veth21`：

```
ip link add veth20 type veth peer name veth21
```



创建出来的网卡设备位于当前系统空间中，查看：

```
[root@worker2 ~]# ifconfig  -a
veth20: flags=4098<BROADCAST,MULTICAST>  mtu 1500
        ether d2:a0:a1:fd:2a:7e  txqueuelen 1000  (Ethernet)
		...
		
veth21: flags=4098<BROADCAST,MULTICAST>  mtu 1500
        ether 36:4c:d6:8e:2e:10  txqueuelen 1000  (Ethernet)
		...
```



## 连接`bridge`和`veth pair`

接下来我们来验证namespace与bridge的联通，最终网络图如下：

![image-20210516223417446](D:\GitRepoWork\Github\psbec\CloudNative-Network\Linux\image\image-20210516223417446.png)



**操作过程**：

**Step1**：创建两个namespace，nns1和nns2：

```
[root@worker2 ~]# ip netns add nns1
[root@worker2 ~]# ip netns add nns2
[root@worker2 ~]# ip netns list
nns2
nns1
```



**Step2**：创建`veth pair`设备：

```
[root@worker2 ~]# ip link add veth10 type veth peer name veth11
[root@worker2 ~]# ip link add veth20 type veth peer name veth21
```



**Step3**：将`veth11`放到`nns1`空间中，将`veth21`放到`nns2`空间中：

```
[root@worker2 ~]# ip link set veth11 netns nns1
[root@worker2 ~]# ip link set veth21 netns nns2
```



**Step4**：设置nns1空间中的网卡地址为`172.16.0.11/24`，设置nns2空间中的网卡地址为`172.16.0.21/24`，然后使能网卡：

```
[root@worker2 ~]# ip netns exec nns1 ip addr add 172.16.0.11/24 dev veth11
[root@worker2 ~]# ip netns exec nns1 ifconfig veth11 up
[root@worker2 ~]# ip netns exec nns2 ip addr add 172.16.0.21/24 dev veth21
[root@worker2 ~]# ip netns exec nns2 ifconfig veth21 up
```



**Step5**：将系统空间中的`veth10`和`veth20`添加到网桥中：

```
[root@worker2 ~]# brctl addif br0 veth10
[root@worker2 ~]# brctl addif br0 veth20
[root@worker2 ~]# brctl show
bridge name     bridge id               STP enabled     interfaces
br0             8000.4a6812e35d87       no              veth10
                                                        veth20
```



**Step6**：测试网络联通性：

```
[root@worker2 ~]# ping 172.16.0.11
PING 172.16.0.11 (172.16.0.11) 56(84) bytes of data.
64 bytes from 172.16.0.11: icmp_seq=1 ttl=64 time=0.114 ms
...

[root@worker2 ~]# ping 172.16.0.21
PING 172.16.0.21 (172.16.0.21) 56(84) bytes of data.
64 bytes from 172.16.0.21: icmp_seq=1 ttl=64 time=0.585 ms
...
```



完工。



## 检查连通性

通过`ip link`命令，我们可以看到系统空间与网络命名空间之间的连接性，命令输出可以参考文档：http://linux-ip.net/gl/ip-cref/ip-cref-node17.html。

我们先在系统空间执行此命令：

```
[root@worker2 ~]# ip link 
5: veth10@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br0 state UP mode DEFAULT group default qlen 1000
    link/ether 4a:68:12:e3:5d:87 brd ff:ff:ff:ff:ff:ff link-netnsid 0
8: veth20@if7: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br0 state UP mode DEFAULT group default qlen 1000
    link/ether d2:a0:a1:fd:2a:7e brd ff:ff:ff:ff:ff:ff link-netnsid 1
```



稍微解释一下`5: veth10@if4`的含义：

1. 第一个冒号前的5表示当前网卡设备编号，全系统唯一；
2. 第一个冒号后，@符号前，这段字符串为当前网络设备名，也就是`veth10`；
3. @后面的字符表示与本设备连接的另外一端设备编号；



由于4号设备并不在系统空间内，所以系统中看不到这个网卡信息。

接下来我们进入到`nns1`空间中看下`ip link`信息：

```
[root@worker2 ~]# ip netns exec nns1 ip link
4: veth11@if5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 2e:3e:ce:b8:ca:78 brd ff:ff:ff:ff:ff:ff link-netnsid 0
```



这里可以看到4号设备出现了，名字为`veth11`，对端设备为5号设备。

至此我们也能看到`veth pair`设备的连接关系了。



# 参考文档



- https://man7.org/linux/man-pages/man4/veth.4.html