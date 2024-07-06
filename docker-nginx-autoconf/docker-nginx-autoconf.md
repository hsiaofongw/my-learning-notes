# 基于 nginx 和 docker user-defined network 的自动配置

## 背景和动机

对于小规模的业务情形，多个应用（applications）通常部署在单台服务器中，服务器的 80 端口和 443 端口由一个应用层网关独占，然后根据 Host（或者 SNI）把流量路由到正确的后端。

以 nginx 为例，就最简单的情形而言，系统管理员在 nginx 配置中声明多个 server 块分别对应不同的 server_name 的不同，然后手动地配置到每一个后端（以端口来区分）的分流：

```
server {
  listen 80;

  server_name app1.example.com

  location / {
    proxy_pass http://127.0.0.1:10081;
  }
}

server {
  listen 80;

  server_name app2.example.com;

  location / {
    proxy_pass http://127.0.0.1:10082;
  }
}
```

当 app 的数量增多时，这样做非常麻烦，因为管理员需要为每个 app 手动分配一个端口号，为此管理员需要记住哪些端口号是已经被使用了的，然后，每新增一个 app，都需要手动更新一次 nginx 配置。

我们需要找到一种更加便捷的工作流程，使得：

- 不需要手动地为每个 app 分配端口号；也不需要一个个地在 nginx 中配置 virtual server 和端口转发；
- 每当有 app 新增/失效时，相应的 layer 7 路由自动生效/失效；

## 解决办法

nginx 的 `server_name` 指令支持以正则表达式的方式实现对请求头 Host（或 SNI）的匹配，并且可以将正则表达式的具名捕获组（named capture group）捕获到的值以变量的形式使用在配置中。

例如，在 127.0.0.1 上部署下列配置的 nginx：

```
server {
  listen 80;
  listen [::]:80;
  server_name ~^(?<domain>[a-zA-Z0-9\-_]+)\.example.com;
  location / {
    return 200 $domain;
  }
}
```

可以实现下列效果：

```
$ curl --resolve app1.example.com:80:127.0.0.1 app1.example.com
app1

$ curl --resolve hello.example.com:80:127.0.0.1 hello.example.com
hello
```

这就有了根据 Host 实现动态分流的基础。

docker 允许我们创建自定义的容器间网络（inter-container network），在自定义网络中，容器和容器互相之间可以「直呼其名」——以名称的方式向对方通信，不需要知道任何网络层的地址（IP 地址）。

下列命令创建一个叫做 testnet 的 docker 容器间网络，网络类型默认为 bridge：

```
docker network create testnet
```

然后我们在这个 testnet 中创建一个叫做 host1 的 container：

```
docker run --network testnet --name host1 -dit --rm ubuntu:22.04
```

然后其它该网络中的主机就可以直接以 host1 这个名称向它发起通信了：

```
$ docker run --network testnet -it --rm busybox ping host1
PING host1 (172.20.0.2): 56 data bytes
64 bytes from 172.20.0.2: seq=0 ttl=64 time=0.594 ms
64 bytes from 172.20.0.2: seq=1 ttl=64 time=0.181 ms
64 bytes from 172.20.0.2: seq=2 ttl=64 time=0.165 ms
^C
--- host1 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.165/0.313/0.594 ms
```

要把这两个功能结合，我们可以用 nginx 的 `proxy_pass` 指令，但是要注意的是，当 `proxy_pass` 指令的参数中包含（需要解析的）变量时，nginx 需要根据 `resolver` 指令声明的解析器对名称的地址进行解析，我们可以在容器内部的 /etc/resolv.conf 中看到这个解析器地址：

```
$ docker run --network testnet -it busybox cat /etc/resolv.conf
nameserver 127.0.0.11
```

更新后的 nginx server 块配置如下所示：

```
server {
  resolver 127.0.0.11;

  listen 80;
  listen [::]:80;
  server_name ~^(?<domain>[a-zA-Z0-9\-_]+)\.foo\.example\.com$;
  location / {
      proxy_pass http://$domain;
  }
}
```

我们需要将 nginx 和 app 都容器化，然后放到同一个 docker 的 user-defined 网络中上述功能才能生效。

最好再结合 CoreDNS 的模板化解析功能自动地将某一层级下的所有子域名都解析为同一个地址，以 foo.example.com 为例，运行下来配置的 coredns 实例：

```
foo.example.com.:53 {
  log
  template IN ANY {
    match ^[0-9a-zA-Z\-_]+[.]foo[.]example[.]com[.]$
    answer "{{ .Name }} 60 IN CNAME gateway.example.com"
    fallthrough
  }
  cache
}
```

则对此 dns 服务器做出的所有模式为“\*.foo.example.com“的查询都会得到指向“gateway.example.com“的 CNAME 结果，一般这里的 CNAME 的真实地址就是指向上面那台 docker 的宿主机。
