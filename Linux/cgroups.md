

# cgroup

有了命名空间，实现了进程的隔离，但是实际上同一台主机上的不同namespace的进程是共享该主机的资源的。如果某个命名空间的流氓进程疯狂占用CPU、内存，这样不就会导致其他命名空间的进程没办法运行了吗？

Linux的cgroup（Control Group）就是用来解决上述问题的，cgroup可以限制一组进程对资源的占用。





cgroup 是 Linux 下的一种将进程按组进行管理的机制，在用户层看来，cgroup 技术就是把系统中的所有进程组织成一颗一颗独立的树，树的每个节点是一个进程组，而每颗树又和一个或者多个 `subsystem` 关联，树的作用是将进程分组，而 `subsystem` 的作用就是对这些组进行操作。cgroup 主要包括下面两部分：

- subsystem : 一个 subsystem 就是一个内核模块，他被关联到一颗 cgroup 树之后，就会在树的每个节点（进程组）上做具体的操作。subsystem 经常被称作 `resource controller`，因为它主要被用来调度或者限制每个进程组的资源，但是这个说法不完全准确，因为有时我们将进程分组只是为了做一些监控，观察一下他们的状态，比如 perf_event subsystem。到目前为止，Linux 支持 12 种 subsystem，比如限制 CPU 的使用时间，限制使用的内存，统计 CPU 的使用情况，冻结和恢复一组进程等，后续会对它们一一进行介绍。
- hierarchy : 一个 `hierarchy` 可以理解为一棵 cgroup 树，树的每个节点就是一个进程组，每棵树都会与零到多个 `subsystem` 关联。在一颗树里面，会包含 Linux 系统中的所有进程，但每个进程只能属于一个节点（进程组）。系统中可以有很多颗 cgroup 树，每棵树都和不同的 subsystem 关联，一个进程可以属于多颗树，即一个进程可以属于多个进程组，只是这些进程组和不同的 subsystem 关联。目前 Linux 支持 12 种 subsystem，如果不考虑不与任何 subsystem 关联的情况（systemd 就属于这种情况），Linux 里面最多可以建 12 颗 cgroup 树，每棵树关联一个 subsystem，当然也可以只建一棵树，然后让这棵树关联所有的 subsystem。当一颗 cgroup 树不和任何 subsystem 关联的时候，意味着这棵树只是将进程进行分组，至于要在分组的基础上做些什么，将由应用程序自己决定，`systemd` 就是一个这样的例子。

cgroups为每种可以控制的资源定义了一个子系统，典型的子系统如下：

- **cpu 子系统**：限制进程的 cpu 使用率。
- **cpuacct 子系统**：统计 cgroups 中的进程的 cpu 使用报告。
- **cpuset 子系统**：为 cgroups 中的进程分配单独的 cpu 节点或者内存节点。
- **memory 子系统**：限制进程的 memory 使用量。
- **blkio 子系统**：限制进程的块设备 io。
- **devices 子系统**：控制进程能够访问某些设备。
- **net_cls 子系统**：标记 cgroups 中进程的网络数据包，然后可以使用 tc 模块（traffic control）对数据包进行控制。
- **freezer 子系统**：挂起或者恢复 cgroups 中的进程。
- **ns 子系统**，不同 cgroups 下面的进程使用不同的 namespace。



我们主要是为了使用cgroup，所以这里就不介绍内核实现cgroup的原理，仅讲应用。

我们关心的一个问题：如何使用cgroup？



## cgroup文件系统

cgroup对用户的界面就是文件系统，首先我们看下有哪些：

```
[root@worker2 ~]# mount | grep cgroup
tmpfs on /sys/fs/cgroup type tmpfs (ro,nosuid,nodev,noexec,seclabel,mode=755)
cgroup on /sys/fs/cgroup/systemd type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,xattr ...)
cgroup on /sys/fs/cgroup/blkio type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,blkio)
cgroup on /sys/fs/cgroup/net_cls,net_prio type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,net_prio,net_cls)
cgroup on /sys/fs/cgroup/perf_event type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,perf_event)
cgroup on /sys/fs/cgroup/cpu,cpuacct type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,cpuacct,cpu)
cgroup on /sys/fs/cgroup/pids type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,pids)
cgroup on /sys/fs/cgroup/devices type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,devices)
cgroup on /sys/fs/cgroup/cpuset type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,cpuset)
cgroup on /sys/fs/cgroup/freezer type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,freezer)
cgroup on /sys/fs/cgroup/hugetlb type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,hugetlb)
cgroup on /sys/fs/cgroup/memory type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,memory)
```













```
yum -y install epel-release
yum install htop
```

