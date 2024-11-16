# WireGuard 手动连接操作步骤手册

1. 启动 wireguard-go 用户态进程，创建 TUN 设备，wireguard-go 会自动把自己和 TUN 设备绑定，例如：

```
sudo LOG_LEVEL=debug wireguard-go -f utun7
```

2. 给 wireguard 接口更新配置文件，使 wireguard 设定生效（否则 wireguard-go 守护进程甚至都不知道 VPN 对端在哪儿）：

```
sudo wg setconf utun7 wg-manual.conf
```

这时可以观察 wireguard-go 前台程序，看 VPN 认证情况如何。

3. 给 TUN 设备手动设定 IP 地址和网关地址（也就是 VPN 访问服务器地址，或者 VPN 隧道另一端的地址）：

```
sudo ifconfig utun7 inet 10.0.4.105/24 10.0.4.1
```

这时应当验证 10.0.4.1 是通的再进行下一步。

4. 设定路由，如果提示路由重复，则有必要删除已存在的路由再重新添加。这个随业务不同，具体操作也不同（看你要设定什么样的路由）。例如：

```
sudo route add 10.0.4.0/24 10.0.4.1
sudo route add 100.64.4.0/24 10.0.4.1
sudo route add 192.168.14.0/24 10.0.4.1
sudo route add 192.168.31.0/24 10.0.4.1
```

提示：对于路由到 wireguard 接口的流量的目的地址，要确保此地址也处在 wireguard 配置文件的放行列表 (AllowedIPs) 中。否则，wireguard 会无声地丢弃这些封包。

5. 删除 wireguard 用户态进程监听的 socket 文件可以使守护进程自动退出，同时也自动删除相应的 TUN 设备：

```
sudo rm -f /var/run/wireguard/utun7.sock
```

6. 如何清空其余配置？首先关闭 Wi-Fi 连接，然后删除所有路由（别担心，下次连接到 Wi-Fi 时系统会自动重建正确的路由）：

```
sudo route -n flush
sudo netstat -f inet -n -r
```

7. 设定 DNS 服务器，一些域名只能通过内网 DNS 服务器（IP 地址是私网地址）解析，要正确设定 DNS 服务器，才能打开这些内网网站。
