# 基于 DNS 劫持和 SNI 嗅探的透明代理

## 目的

提供一种简单易配置的透明代理实现方式。

## 原理简介

一个应用发起 HTTP 请求会经过下列阶段：

1. 查询请求目标的主机名称对应的网络层地址（通常是 IPv4 地址或者 IPv6 地址）；
2. 向上述地址建立连接，这一步是必须的，因为 HTTP 协议需要建立在一个可靠的信道上；
3. 构造 HTTP 请求报文，通过上一步骤建立起的可靠信道发送。

如果我们在这其中的第 1 个阶段对 DNS 请求进行拦截，返回一个精心构造的（通常来说也是固定的）IP 地址，就能够按需地把这一 HTTP 请求路由到任何想要的地点。具体来说，如果我们劫持这里的 DNS 请求并且返回 DNS 回答 172.21.0.3，那么，这个 HTTP 请求就会被发到 172.21.0.3。

但这时网络层封包的目的地址信息已经被重置（成了 172.21.0.3），我们如何还原出最初的网络层请求地址呢？

- 对于 HTTP（明文）请求，从请求报文 Header 中的 Host 字段取得；
- 对于 HTTPS 请求，从 SNI 信息中取得。

以上方法取到的都是主机名，例如 www.example.com 这种，要得到 IP 地址，再做一次 DNS 解析即可，只不过这次的 DNS 解析应当是无拦截的、真实的 DNS 解析。

### 概念澄清

什么是「透明代理」？

不同的应用具有不同的代理配置方式，透明代理可以使得不需要对应用进行配置或者源码的修改，也能使代理对应用生效。换句话说，所谓透明代理，是指一种对应用来说无感的代理环境。

## 过程演示

### 搭建测试 web 服务器

创建名为 servernet 的测试网络：

```sh
docker network create --subnet 172.23.0.0/24 servernet  # IP 子网自取，不冲突即可。
```

保存下列内容到当前目录下的 webserver/default.conf 文件：

```
server {
    listen 80;
    listen [::]:80;

    location / {
        add_header Content-Type "text/plain";
        return 200 $realip_remote_addr;
    }
}
```

后台启动 webserver，container 名称为 ngx：

```sh
docker run --network servernet -dit --name ngx -v $(pwd)/webserver:/etc/nginx/conf.d --rm nginx:1.27.0
```

验证 webserver 已启动，向 http://ngx 地址发送 HTTP 请求，其中 ngx 会由 docker 自动为自定义网络附加的 DNS 解析器来进行解析（为容器的 IP 地址）：

```sh
docker run --network servernet -it --name curl --rm curlimages/curl:8.8.0 ngx
```

此命令应当返回该容器自身在这个名为 servernet 的自定义网络中的 IP 地址。

### 搭建示例代理服务器

代理服务器位于用户网络和服务器网络可达范围的交集，在 docker 中可以轻易进行这样的模拟，为此，我们只需再创建一个「用户网络」，该网络默认与服务器网络隔离：

```sh
docker network create --subnet 172.24.0.0/24 usernet
```

容易验证，usernet 确实对于 servernet 网络默认不可达：

```
$ docker run --network usernet -it --rm curlimages/curl:8.8.0 -v ngx
* Could not resolve host: ngx
* Closing connection
curl: (6) Could not resolve host: ngx

$ docker run --network usernet -it --rm busybox:1.36.1 ping -c1 ngx
ping: bad address 'ngx'

$ docker run --network usernet -it --rm busybox:1.36.1 ping -c1 ping-host
PING ping-host (172.24.0.2): 56 data bytes
64 bytes from 172.24.0.2: seq=0 ttl=64 time=0.077 ms

--- ping-host ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 0.077/0.077/0.077 ms
```

接下来我们创建一个名为 proxy 的 container（用来模仿 proxy 的行为以及扮演 proxy 的角色），它特别的地方在于，它同时位于 usernet（用户网络）和 servernet（服务器网络）：

```sh
mkdir proxy
cat <<EOF > proxy/default.conf
server {
    listen 80 default_server;
    listen [::]:80 default_server;

    resolver 127.0.0.11; # 127.0.0.11 是 docker 自定义网络默认 DNS（stub）解析器

    location / {
        proxy_pass http://$host;
    }
}
EOF

docker run -dit --rm --name proxy -v $(pwd)/proxy:/etc/nginx/conf.d nginx:1.27.0
docker network connect servernet proxy
docker network connect usernet proxy
docker network disconnect bridge proxy
```

然后我们在用户 usernet 中查询它的 IP 地址：

```
$ docker run --network usernet -it --rm busybox:1.36.1 nslookup proxy
Server:		127.0.0.11
Address:	127.0.0.11:53

Non-authoritative answer:

Non-authoritative answer:
Name:	proxy
Address: 172.24.0.2
```

### 通过代理访问测试 HTTP 服务

接下来就是发挥 DNS 透明代理的重要时刻：要使得 usernet 中的应用也能访问它原本访问不到的 servernet 中的服务，只需要让 usernet 的 DNS 请求被解析为 proxy 的地址即可：

```
$ docker run --network usernet --add-host ngx=172.24.0.2 -it --rm curlimages/curl:8.8.0 -v ngx
* Host ngx:80 was resolved.
* IPv6: (none)
* IPv4: 172.24.0.2
*   Trying 172.24.0.2:80...
* Connected to ngx (172.24.0.2) port 80
> GET / HTTP/1.1
> Host: ngx
> User-Agent: curl/8.8.0
> Accept: */*
>
* Request completely sent off
< HTTP/1.1 200 OK
< Server: nginx/1.27.0
< Date: Thu, 11 Jul 2024 19:02:29 GMT
< Content-Type: application/octet-stream
< Content-Length: 10
< Connection: keep-alive
<
* Connection #0 to host ngx left intact
172.23.0.3
```

于是我们看到，现如今 usernet 中的应用成功地通过了 proxy（或者说将地址解析到 proxy 这个中间人角色）访问到了它原本访问不到的 servernet 提供的服务 ngx，并且，我们还看到了 ngx 返回的是 proxy 在 servernet 中的地址（172.23.0.0/16 网段）。这里的 DNS 劫持行为，是通过 `--add-host` 命令行参数来模拟的，这样做，可以使得该容器发出的针对特定名称的 DNS 查询得到特定的结果，尽管 curl 命令行工具自身也支持 `--resolve` 参数，但那样涉及到了对应用本身的配置，因此就脱离「透明代理」的范畴了，毕竟严格意义上来说只有应用无感的代理才配叫做透明代理。

### 搭建 HTTPS 测试服务器

修改 webserver/default.conf 文件，将它改为：

```
server {
    listen 80;
    listen 443 ssl;
    server_name myip.com;

    ssl_certificate_key /custom/certs/myip.com.key;
    ssl_certificate /custom/certs/myip.com.crt;
    ssl_protocols TLSv1.2 TLSv1.3;

    location / {
        add_header Content-Type "text/plain";
        return 200 $realip_remote_addr;
    }
}
```

其中 myip.com.crt 文件和 myip.com.key 文件分别是自签发的 SSL 证书和私钥。假设证书和私钥都位于当前目录下的 ssl 文件夹。通过如下命令再次启动测试 HTTPS 服务器：

```sh
docker run \
    --network servernet \
    -dit --name ngx --ip 172.23.0.2 \
    -v $(pwd)/ssl:/custom/certs \
    -v $(pwd)/webserver:/etc/nginx/conf.d \
    --rm nginx:1.27.0
```

搭建测试 DNS 服务器，因为正如本文开头所说，基于 DNS 劫持的代理有一个步骤涉及到向「真正的」DNS 服务器发起解析请求（来得到真实的 IP 目的地址）：

```sh
mkdir dns
cat <<EOF > dns/Corefile
myip.com:53 {
  template IN A {
    answer "{{ .Name }} 60 IN A 172.23.0.2"
  }
  log
}
EOF

docker run \
    -v $(pwd)/dns:/etc/coredns \
    --network usernet
    -dit \
    --name coredns \
    --rm \
    coredns/coredns:1.11.1 \
        -conf /etc/coredns/Corefile

docker network connect servernet coredns # 添加这个是为了便于在 servernet 中直接测试 webserver 的 https 服务是否正常
```

查看 coredns 本身的 IP 地址：

```
docker run --rm -it --network servernet busybox:1.36.1 nslookup -type=a coredns
Server:		127.0.0.11
Address:	127.0.0.11:53

Non-authoritative answer:
Name:	coredns
Address: 172.23.0.4

docker run --rm -it --network usernet busybox:1.36.1 nslookup -type=a coredns
Server:		127.0.0.11
Address:	127.0.0.11:53

Non-authoritative answer:
Name:	coredns
Address: 172.24.0.3
```

这个测试 DNS 服务器负责扮演那个所谓的真实 DNS 服务器负责解析测试网站 myip.com 的真实地址。

假设 ssl/ca.crt 是签发上述证书的 CA 证书文件，那么我们可以使用如下方式来验证这个 HTTPS 服务已经成功启用：

```sh
docker run \
   --network servernet \
   --dns 172.23.0.4 \
   -v $(pwd)/ssl:/custom/certs \
   -it --rm curlimages/curl:8.8.0  \
   -4 --cacert /custom/certs/ca.crt \
   https://myip.com
```

其中：

- `--network servernet` 表示在 servernet 中运行这个 container，因为 ngx 就是部署在 servernet 中的，这样可以直接测试它，而不需要再绕道其它网络；
- `--dns 172.23.0.4` 是我们的模拟真实 DNS 服务器 coredns 在 servernet 中的地址，这一设置之后，容器就会默认向这个 DNS 服务器发起请求，也就是说会向 172.23.0.4 这台 DNS 服务器查询 myip.com 的地址是多少；

至此，HTTPS 测试服务器和仿真 DNS 服务器建立成功。自签 SSL 证书是通过 openssl 命令行工具生成的，但是具体方式限于篇幅不在本文中讨论。

### 通过代理访问测试 HTTPS 服务

为了让 proxy 也能够转发 443 端口的请求，我们需要对它的配置进行修改，添加 proxy/nginx.conf 文件，内容如下：

```
user nginx;
worker_processes auto;

error_log /var/log/nginx/error.log notice;
pid /var/run/nginx.pid;


events {
    worker_connections 1024;
}


http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
    '$status $body_bytes_sent "$http_referer" '
    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;

    sendfile on;
    #tcp_nopush     on;

    keepalive_timeout 65;

    #gzip  on;

    include /etc/nginx/conf.d/*.conf;
}


stream {
    resolver 172.24.0.3 ipv6=off; # resolver 是 DNS 服务器在 usernet 中的地址
    ssl_preread on;

    server {
        listen 0.0.0.0:443;
        proxy_pass $ssl_preread_server_name:443;
    }
}
```

其中，除了 `stream {}` 块之外，其它的都是默认配置，可以从 nginx 运行的容器中拷贝得到。然后停止 proyx 并再次启动，注意把 proxy/nginx.conf 文件挂载到 /etc/nginx/nginx.conf，并且把 proxy/default.conf 挂载到 /etc/nginx/conf.d/default.conf。

然后我们可以在 usernet 中，向 proxy 发起真实目的地为 ngx 的请求：

```sh
docker run \
    --network usernet \
    --add-host myip.com:172.24.0.2 \
    -v $(pwd)/ssl:/custom/certs \
    -it \
    --rm curlimages/curl:8.8.0 \
        -4 --cacert /custom/certs/ca.crt https://myip.com
```

注意其中的 `--add-host myip.com:172.24.0.2`，它相当于模拟了 DNS 劫持，这个容器发出的解析 myip.com 地址的 DNS 请求，会被劫持并且返回 172.24.0.2 这个 proxy 在 usernet 中的地址，于是接下来 HTTPS 请求就会被发送到 proxy 的 443 端口，然后 proxy 向 ngx 的 443 端口建立连接，并且进行传输层的转发。

由于 proxy 不需要知道 HTTPS 的密文（SNI 是明文的），也不需要修改或者重新构造 HTTP 协议的请求或者响应内容本身，所以不需要进行证书劫持。

## 总结

在这篇文章中，我们主要是通过 docker 已经 docker 强大的自定义网络功能来进行网络环境的模拟，并且在这个虚拟的网络环境中，去演示如何通过 DNS 劫持配合 SNI（或者 Host）嗅探的方式来实现透明代理。
