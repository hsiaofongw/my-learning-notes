# 基于 cgroup 的透明代理

## 动机

在构建 docker 容器镜像过程中，经常会在下列环节遇到网络中断的问题导致构建失败：

1. 拉取 layer；
2. 运行自定义程序构建 layer；

对于 1，dockerd 提供了配置文件接口便于实现网络优化。但是对于 2，目前来说没有一个统一的配置接口（或者配置方式）去实现网络优化，例如说，一个 Dockerfile 文件中有下列代码：

```Dockerfile
RUN /path/to/a param1 param2
RUN b.sh param1
RUN ./c param1 param2 param3
RUN /path/d
```

其中，a，b.sh，./c 所接受的网络配置接口不尽相同，例如，a 只接受通过配置文件的方式，或者命令行参数的方式，设定网络优化配置，b.sh 只接受全大写的 HTTPS_PROXY 环境变量，而 ./c.sh 只接受 socks_proxy 或者 SOCKS_PROXY 这两个环境变量，/path/d 不接受任何环境变量或命令行参数。

我们的目的是，寻找一种统一的、一次性的网络优化配置方式，使其对构建一个 docker 容器镜像的所有阶段生效。

### 问题的澄清

我们说的网络优化是指，通过一定的配置，使得特定任务中运行的程序所发送和接收的网络封包，能够走更好的网络路径（路由），从而降低任务因为网络中断而导致的失败的概率。

我们说的配置接口（或者配置方式）是指程序和用户之间的一种约定，这种约定是关于「用户以何种方式去影响程序运行过程中和网络相关的行为」的。例如环境变量 HTTPS_PROXY 是一种配置接口，因为它影响了程序运行过程中的网络行为。

## cgroup 方式

这种基于 cgroup 的网络优化配置方式，称为 cgroup 方式。

### cgroup 的简介

cgroup 全称是 control group，它是 Linux 内核（后续简称 Linux）提供的用来对进程进行归组并且设定资源限额（CPU、内存、IO 限额等）的一种方式。例如，我可以参加一个 cgroup 叫做 /group1，然后再创建三个进程 a，b，c 并且把这三个进程移交到这个叫做 /group1 的 cgroup，以这种方式把 a，b，c 这三个进程归为一组，后续通过对 /group1 这个 cgroup 设定资源配额，可以约束 a，b，c 这三个进程的总体行为，例如，可以使得这些个进程的内存使用量加起来不超过多少。

一个新创建的 cgroup 默认没有任何资源限额附加在其中，所以充其量也就相当于一个名字，一个标签，一个没有附加任何资源限额在内的 cgroup 的作用可以是对多个进程进行归组，并且给这个「组」一个名字。一个 cgroup 可以有 0 个进程、1 个或多个进程。

iptables（以及 ip6tables）支持根据 cgroup 的 path 来对封包进行打标签，例如：

```sh
iptables -t mangle -A OUTPUT -m cgroup --path /group1 -j MARK --set-mark 101
```

上述命令匹配所有来自 /group1 这个 cgroup 里面的进程发出的封包，并且对这些封包打上 fwmark 101 的标记。

大部分 iptables 的 extensions（通过 `-m` 参数使用），和 targets（通过 `-j` 参数使用）的说明都可以在[这个页面](https://man7.org/linux/man-pages/man8/iptables-extensions.8.html)上查到。

iptables 本质上操作的是 Linux 的 netfilter hooks，对封包打 fwmark 标记的这一行为并不会修改封包本身的任何内容（包括封包头部），仅仅是在 Linux 内核网络协议栈中对封包进行了「标记」（或者说，一种关联），Linux 有办法判断一个封包是否被打上了特定的 fwmark 标记。

有了 fwmark 标记，我们可以有针对性地，针对打了特定 fwmark 标记的封包设定定制化的路由策略，下列命令：

```sh
ip rule add fwmark 101 table 101
```

使得 Linux 专门在 ID 为 101 的路由表上查找具有 fwmark 101 标记的封包的路由，也就是说，假设现如今 Linux 需要查找一个 IP 封包的 nexthop，那么，假如知道了它具有 fwmark 101 这个标记，Linux 就不会去系统默认路由表查找路由，而是去 ID 为 101 的路由表去查找路由。而且这两个数字不需要相等。

下一步，我们就可以在 101 路由表中添加默认路由：

```sh
ip route add default via 10.0.1.1 dev tun0 table 101
```

该路由表仅仅作用于特定 fwmark 的封包，不会影响到整个系统的默认路由。经过了上述配置，可以使得 /group1 这个 cgroup 中的所有进程发出的封包，走的默认路由都和别的进程不一样，换句话说，可以让这个 cgroup 内所有的进程发出的封包都走自定义的默认路由，而不是默认的默认路由。

这个特殊 cgroup 专属的默认路由的设定可以非常灵活，它的地址可以是另一台路由器的 IP 地址，可以是一个虚拟机的 IP 地址，也可以是一个 tun 虚拟网卡的 IP 地址，甚至可以是一个 docker 容器的 IP 地址。它还可以是本机（lo 网卡），当以 dev lo 作为默认路由时，可以结合用户态的 [TPROXY](https://www.kernel.org/doc/Documentation/networking/tproxy.txt) 应用程序一起使用。
