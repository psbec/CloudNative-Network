





# Linux虚拟网络设备

## Linux虚拟网络设备



### tun/tap设备

这两个虚拟网络设备可以被看做是p2p的以太网络设备，用户可以通过这两个网络设备收发报文，操作起来与物理网卡一般无二。





Tun/tap interfaces are *software-only interfaces*, meaning that they exist only in the kernel and, unlike regular network interfaces, they have no physical hardware component (and so there's no physical "wire" connected to them). You can think of a tun/tap interface as a regular network interface that, when the kernel decides that the moment has come to send data "on the wire", instead sends data to some userspace program that is attached to the interface (using a specific procedure, see below). When the program attaches to the tun/tap interface, it gets a special file descriptor, reading from which gives it the data that the interface is sending out. In a similar fashion, the program can write to this special descriptor, and the data (which must be properly formatted, as we'll see) will appear as input to the tun/tap interface. To the kernel, it would look like the tun/tap interface is receiving data "from the wire".
The difference between a tap interface and a tun interface is that a tap interface outputs (and must be given) full ethernet frames, while a tun interface outputs (and must be given) raw IP packets (and no ethernet headers are added by the kernel). Whether an interface functions like a tun interface or like a tap interface is specified with a flag when the interface is created.







TUN/TAP两种设备的主要应用场景就是隧道应用，[VTUN](http://vtun.sourceforge.net/) 工程就是基于此原理实现的VPN功能。

两种设备的差异：

- TUN是纯三层网络驱动，只处理IP报文，所以此设备是不需要mac地址的；
- TAP是二层网络驱动，所以 此设备有mac地址。



TUN（Tunel）设备模拟网络层设备，处理三层报文，如IP报文。TAP设备模型链路层设备，处理二层报文，比如以太网帧。TUN用于路由，而TAP用于创建网桥。



命令帮助信息：

```
[root@worker2 ~]# ip tuntap help
Usage: ip tuntap { add | del | show | list | lst | help } [ dev PHYS_DEV ]
          [ mode { tun | tap } ] [ user USER ] [ group GROUP ]
          [ one_queue ] [ pi ] [ vnet_hdr ] [ multi_queue ] [ name NAME ]

Where: USER  := { STRING | NUMBER }
       GROUP := { STRING | NUMBER }
```



接下来以root用户创建tun/tap设备：

```
ip tuntap add dev tun0 mod tun
ip tuntap add dev tap0 mod tap
```



对应的删除设备命令为：

```
ip tuntap del dev tap0 mod tap
ip tuntap del dev tun0 mod tun
```



给`tun0`设备增加IP地址，使能设备：

```
[root@worker2 ~]# ip addr add 10.1.0.1 dev tun0
[root@worker2 ~]# ifconfig tun0 up
[root@worker2 ~]# ifconfig tun0
tun0: flags=4241<UP,POINTOPOINT,NOARP,MULTICAST>  mtu 1500
        inet 10.1.0.1  netmask 255.255.255.255  destination 10.1.0.1
...
```



我们尝试在windows机器上ping新增的地址，`tun0`地址设备的地址是虚地址，操作之前需要设置路由：

```
C:\Users\Administrator>route add 10.1.0.0 mask 255.255.255.0 192.168.31.31
C:\Users\Administrator>ping 10.1.0.1
正在 Ping 10.1.0.1 具有 32 字节的数据:
来自 10.1.0.1 的回复: 字节=32 时间<1ms TTL=64
```



## 网桥-bridge

一般的局域网交换中都会使用到交换机，Linux系统中的网桥就是相当于本系统内部的一个交换机，可以将连接到本网桥的网络设备进行二层联通。

下面是一个局域网中使用交换机实现网络连通：

![image-20210516154535085](D:\GitRepoWork\Github\psbec\CloudNative-Network\Linux\image\image-20210516154535085.png)



Linux bridge原理与计算机相同，bridge可以将多个namespace的网络空间连通：

![image-20210516154931457](D:\GitRepoWork\Github\psbec\CloudNative-Network\Linux\image\image-20210516154931457.png)



物理世界中，PC机需要通过一个网线连接交换机，软件世界中如何将namespace与内核的bridge连接起来呢？答案就是通过veth pair，我们可以把veth pair想象成一个连接了两个网卡的设备，一端连接namespace，另外一端连接bridge。

接下来先介绍下veth pair设备。



## veth pair

Linux内核提供了`veth pair`设备，这个设备有个特点，每次创建两个虚拟网络设备，两个网络设备是联通的，也就是从一个设备发送数据，就会从另外一个设备收到。就是通过这个特性，可以实现namespace和bridge之间的连接。

基本原理：

1. 创建一个`veth pair`设备

```
 The veth devices are virtual Ethernet devices.  They can act as
       tunnels between network namespaces to create a bridge to a
       physical network device in another namespace, but can also be
       used as standalone network devices.
```



# 参考文档



- https://www.kernel.org/doc/html/latest/networking/tuntap.html
- https://backreference.org/2010/03/26/tuntap-interface-tutorial/
- 