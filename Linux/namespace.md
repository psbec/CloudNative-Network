虚拟化网络专题（1） - Namespace



# Namespace

Namespace是Linux内核提供的一种全局资源隔离方案，位于特定Namespace空间中运行的程序认为系统中所有的资源都是独占的，就像拥有一台独立的物理机。不同Namespace空间的进程完全隔离，一个namespace空间的进程完全感知不到其他namespace进程的存在。

容器化技术正是用了内核的namespace特性，实现了不同容器之间进程的完全隔离。



## 资源隔离

内核到底隔离了哪些资源呢？下面是列表：

```
Namespace Flag            Page                  Isolates
Cgroup    CLONE_NEWCGROUP cgroup_namespaces(7)  Cgroup root directory
IPC       CLONE_NEWIPC    ipc_namespaces(7)     System V IPC, POSIX message queues
Network   CLONE_NEWNET    network_namespaces(7) Network devices, stacks, ports, etc.
Mount     CLONE_NEWNS     mount_namespaces(7)   Mount points
PID       CLONE_NEWPID    pid_namespaces(7)     Process IDs
Time      CLONE_NEWTIME   time_namespaces(7)    Boot and monotonic clocks
User      CLONE_NEWUSER   user_namespaces(7)    T{User and group IDs T}
UTS       CLONE_NEWUTS    uts_namespaces(7)     Hostname and NIS domain name
```



列说明：

- 第一列是namespace名称；
- 第二列是通过API调用时使用的Flag值；
- 第三列对应了man page说明；
- 第四列描述了该Namespace隔离的资源列表



### 查看进程归属namespace

每个进程的`/proc/PID/ns/`目录下保存了该进程使用的namespace属性信息：

```bash
hxg@hubuntu:~/github$ sudo ls -l /proc/1/ns/
总用量 0
lrwxrwxrwx 1 root root 0  5月 11 02:15 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 root root 0  5月 11 02:15 ipc -> 'ipc:[4026531839]'
lrwxrwxrwx 1 root root 0  5月 11 02:15 mnt -> 'mnt:[4026531840]'
lrwxrwxrwx 1 root root 0  5月 11 02:15 net -> 'net:[4026531992]'
lrwxrwxrwx 1 root root 0  5月 11 02:15 pid -> 'pid:[4026531836]'
lrwxrwxrwx 1 root root 0  5月 11 02:15 pid_for_children -> 'pid:[4026531836]'
lrwxrwxrwx 1 root root 0  5月 11 02:15 time -> 'time:[4026531834]'
lrwxrwxrwx 1 root root 0  5月 11 02:15 time_for_children -> 'time:[4026531834]'
lrwxrwxrwx 1 root root 0  5月 11 02:15 user -> 'user:[4026531837]'
lrwxrwxrwx 1 root root 0  5月 11 02:15 uts -> 'uts:[4026531838]'
```

最后一列表示为`DeviceID : inodeNumber`，如果不同的进程，其namespace资源指向的inodeNumber相同，则证明两个进程归属同一个namespace。



### 系统支持的最大namespace数量

`/proc/sys/user/`目录下的`max_XXX_namespaces`文件记录了XXX类型namespace支持的数量，列表如下：

```bash
hxg@hubuntu:~/github$ sudo ls /proc/sys/user/
max_cgroup_namespaces  max_ipc_namespaces  max_pid_namespaces	max_uts_namespaces
max_inotify_instances  max_mnt_namespaces  max_time_namespaces
max_inotify_watches    max_net_namespaces  max_user_namespaces
```

下面查看的是PID namespace支持数量，为23074个：

```bash
hxg@hubuntu:~/github$ sudo cat /proc/sys/user/max_pid_namespaces 
23074
```



## 操作网络命名空间

### 命令

本系列文章目的是为了讲清楚虚拟化网络，所以我们将重点看网络的namespace。

系统提供了`ip netns`命令操作网络namespace，功能如下：

```bash
[root@worker2 ~]# ip netns help
Usage: ip netns list
       ip netns add NAME
       ip netns set NAME NETNSID
       ip [-all] netns delete [NAME]
       ip netns identify [PID]
       ip netns pids NAME
       ip [-all] netns exec [NAME] cmd ...
       ip netns monitor
       ip netns list-id
```



### 查看命名空间

默认情况下系统中没有任何网络命名空间，为了验证功能，我们需要手工创建。

下面的命令用于创建网络命名空间`nst1`：

```bash
[root@worker2 ~]# ip netns add nst1
```

创建出来的命名空间文件位于`/var/run/netns/`目录下：

```bash
[root@worker2 ~]# ls /var/run/netns
nst1
```

接下来我们看下`nst1`命名空间中的网络配置，空的命名空间只有`lo`设备，看起来真的像是一台独立的物理机：

```
[root@worker2 ~]# ip netns exec nst1 ifconfig lo up
[root@worker2 ~]# ip netns exec nst1 ifconfig -a
lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
		...
```



# 参考

- https://man7.org/linux/man-pages/man7/namespaces.7.html