# VLESS over TCP with TLS + Fallback to Nginx & VLESS/Vmess over WebSocket (CDN) + Cloudflare WARP
- **本例的VPS系统为 `Debian10` ，使用 `root` 用户**

**查看系统时间是否正确，时区可任意**

```
date -R
```

**更新包存储库**

```
apt update
```

**打开80端口**

```
ufw allow 80/tcp
```

**打开443端口**

```
ufw allow 443/tcp
```

**检查UFW状态**

```
ufw status verbose
```

## 1.安装Xray

**Install & Upgrade Xray-core and geodata with `User=nobody`, but will NOT overwrite `User` in existing service files**

```
bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install
```

**配置Xray**

```
{
    "log": {
        "loglevel": "warning",
        "access": "/var/log/xray/access.log",
        "error": "/var/log/xray/error.log"
    },
    "dns": {
      "servers": [
        "localhost",
        "https+local://dns.google/dns-query"
      ]
    },
    "inbounds": [
        {
            "port": 443,
            "protocol": "vless",
            "settings": {
                "clients": [
                    {
                        "id": "", // 填写你的 UUID
                        "level": 0,
                        "email": "tl@xray.com"
                    }
                ],
                "decryption": "none",
                "fallbacks": [
                    {
                        "dest": 33222,
                        "xver": 1
                    },
                    {
                        "alpn": "h2",
                        "dest": 33223,
                        "xver": 1
                    },
                    {
                        "path": "/websocket", // 必须换成自定义的 PATH
                        "dest": "/dev/shm/wsray.sock",
                        "xver": 1
                    },
                    {
                        "path": "/websocket", // 必须换成自定义的 PATH
                        "dest": "/dev/shm/vmwsray.sock",
                        "xver": 1
                    }
                ]
            },
            "streamSettings": {
                "network": "tcp",
                "security": "tls",
                "tlsSettings": {
                    "alpn": [
                        "h2",
                        "http/1.1"
                    ],
                    "certificates": [
                        {
                            "certificateFile": "/home/user/xray_cert/xray.crt", // 换成你的证书，绝对路径
                            "keyFile": "/home/user/xray_cert/xray.key" // 换成你的私钥，绝对路径
                        }
                    ]
                }
            }
        },
        {
            "listen": "/dev/shm/wsray.sock",
            "protocol": "vless",
            "settings": {
                "clients": [
                    {
                        "id": "", // 填写你的 UUID
                        "level": 0,
                        "email": "ws@xray.com"
                    }
                ],
                "decryption": "none"
            },
            "streamSettings": {
                "network": "ws",
                "security": "none",
                "wsSettings": {
                    "acceptProxyProtocol": true, // 提醒：若你用 Nginx/Caddy 等反代 WS，需要删掉这行
                    "path": "/websocket" // 必须换成自定义的 PATH，和上面分流的一致
                }
            }
        },
        {
            "listen": "/dev/shm/vmwsray.sock",
            "protocol": "vmess",
            "settings": {
                "clients": [
                    {
                        "id": "", // 填写你的 UUID
                        "level": 0,
                        "email": "vmws@xray.com"
                    }
                ]
            },
            "streamSettings": {
                "network": "ws",
                "security": "none",
                "wsSettings": {
                    "acceptProxyProtocol": true, // 提醒：若你用 Nginx/Caddy 等反代 WS，需要删掉这行
                    "path": "/websocket" // 必须换成自定义的 PATH，和上面分流的一致
                }
            }
        }        
    ],
 "outbounds": [
    {
      "tag":"IP4_out",
      "protocol": "freedom",
      "settings": {}
    },
    {
      "tag":"IP6_out",
      "protocol": "freedom",
      "settings": {
        "domainStrategy": "UseIPv6"
      }
    },
    {
      "tag": "direct",
      "protocol": "freedom",
      "settings": {}
    }
  ],
  "routing": {
    "rules": [
      {
        "type": "field",
        "outboundTag": "direct",
        "domain": [
          "full:dns.google"  
        ]
      },
      {
        "type": "field",
        "outboundTag": "IP6_out",
        "domain": [
          "geosite:netflix", // netflix 走 IPv6，可以多写几个，比如 "domain": ["geosite:netflix","geosite:google"] 
          "geosite:instagram",
          "geosite:facebook",
          "geosite:google"
        ]
      },
      {
        "type": "field",
        "outboundTag": "IP6_out",
        "ip": [
          "geoip:netflix",
          "geoip:facebook",
          "geoip:google"
        ]
      },
      {
        "type": "field",
        "outboundTag": "IP4_out",
        "network": "tcp,udp"
      }
    ]
  }
}
```

- Xray 回落分流 WS 比 Nginx 反代 WS 性能更强
- 开启PROXY protocol，显示请求的真实来源 IP 和端口
- 支持 h2 访问：ALPN 协商结果为 h2 时单独转发
- Xray 回落分流 WS 使用 Unix domain socket，比环回地址效率更高
- IPv6 的出口指定和路由设置，配合 WireGuard 走 Cloudflare WARP 网络

**替换默认路由规则文件为 `Loyalsoldier/v2ray-rules-dat`**

```
curl -L https://github.com/Loyalsoldier/v2ray-rules-dat/releases/latest/download/geoip.dat  -o /usr/local/share/xray/geoip.dat
curl -L https://github.com/Loyalsoldier/v2ray-rules-dat/releases/latest/download/geosite.dat  -o /usr/local/share/xray/geosite.dat
```

**在/etc/logrotate.d/下新增配置文件，自动分割Xray日志**

- **复制/etc/logrotate.d/中的任意一个文件(如 `nginx` 文件)，并重命名为 `xray`，删除文件里的所有内容，加入如下配置**

```
/var/log/xray/*.log {
    daily
    missingok
    rotate 14
    compress
    delaycompress
    notifempty
    create 0640 nobody adm
}
```

**Xray菜单**

```
bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ help
```

**启动Xray**

```
systemctl daemon-reload
```

```
systemctl start xray
```

**设置Xray开机自启动**

```
systemctl enable xray
```

**查看Xray运行状态**

```
systemctl status xray
```

## 2.安装Nginx

```
apt install nginx
```

**配置Nginx**

```
user  root;
worker_processes  1;
error_log  /var/log/nginx/error.log warn;
#pid    /var/run/nginx.pid;
events {
    worker_connections  1024;
}
http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for" '
                      '$proxy_protocol_addr:$proxy_protocol_port';
    access_log  /var/log/nginx/access.log  main;
    sendfile        on;
    #tcp_nopush     on;
    keepalive_timeout  120;
    client_max_body_size 20m;
    #gzip  on;
server {
    listen 127.0.0.1:33222 proxy_protocol;
    server_name  mydomain.com;
    root /home/user/www/webpage;
    index index.html;
    
    set_real_ip_from 127.0.0.1;
}
server {
    listen 127.0.0.1:33223 http2 proxy_protocol;
    server_name  mydomain.com;
    root /home/user/www/webpage;
    index index.html;
    
    set_real_ip_from 127.0.0.1;
}
#server {
#    listen       0.0.0.0:80;
#    server_name  mydomain.com;
#    return 301 https://mydomain.com$request_uri;
#}
}
```

- 开启PROXY protocol，显示请求的真实来源 IP 和端口
- 支持 h2 访问
- access 日志中每行末尾有请求的真实来源 IP 和端口

**重启Nginx**

```
systemctl restart nginx
```

**设置Nginx开机自启动**

```
systemctl enable nginx
```

**查看Nginx运行状态**

```
systemctl status nginx
```

**建立网站文件夹**

```
mkdir -p /home/user/www/webpage
```

- 自行部署伪装网站

## 3.安装acme.sh

**安装 acme.sh 所需要的依赖项**

```
apt-get install openssl cron socat curl
```

**安装acme.sh**

```
curl  https://get.acme.sh | sh
```

**让 acme.sh 命令生效**

```
source ~/.bashrc
```

**开启 acme.sh 的自动升级**

```
acme.sh --upgrade --auto-upgrade
```

**设置 acme.sh 默认 CA 为 letsencrypt**

```
acme.sh --set-default-ca  --server  letsencrypt
```

**申请证书**

- **http 方式需要在你的网站根目录下放置一个文件, 来验证你的域名所有权,完成验证. 然后就可以生成证书了.**

```
acme.sh --issue -d mydomain.com --webroot /home/user/www/webpage
```

申请 `ECC` 证书

```
acme.sh --issue -d mydomain.com --webroot /home/user/www/webpage --ecc
```

- **standalone方式 `(standalone方式需要使用80端口，请先停止Nginx)`**

```
acme.sh --issue -d mydomain.com --standalone --keylength ec-256 --force
```

**建立证书文件夹**

```
mkdir -p /home/user/xray_cert
```

**安装证书和密钥**

```
acme.sh --install-cert -d mydomain.com --ecc \
        --fullchain-file /home/user/xray_cert/xray.crt \
        --key-file /home/user/xray_cert/xray.key
```

**赋予`xray.key`可读性权限**

```
chmod -R 755 /home/user/xray_cert/xray.key
```

## 4.安装WireGuard

**安装 curl 和 lsb_release**

```
apt install curl lsb-release -y
```

**添加 backports 源**

```
echo "deb http://deb.debian.org/debian $(lsb_release -sc)-backports main" | tee /etc/apt/sources.list.d/backports.list
apt update
```

**安装网络工具**

```
apt install net-tools iproute2 openresolv dnsutils -y
```

**安装 wireguard-tools**

```
apt install wireguard-tools --no-install-recommends
```

**先执行 `uname -r` 命令查看内核版本。如果是 5.6 以上内核则已经集成了 WireGuard ，就不需要安装了。如果不是，执行下面的命令安装新版内核**

```
apt -t $(lsb_release -sc)-backports install linux-image-$(dpkg --print-architecture) linux-headers-$(dpkg --print-architecture) --install-recommends -y
```

- **安装完重启，并执行 `uname -r` 命令查看内核版本来确认新内核是否被启用**

**使用 wgcf 生成 WireGuard 配置文件**

- **安装 wgcf**

`wgcf` 是 `Cloudflare WARP` 的非官方 CLI 工具，它可以模拟 WARP 客户端注册账号，并生成通用的 WireGuard 配置文件。

```
curl -fsSL git.io/wgcf.sh | bash
```

- **注册 WARP 账户 (生成 /root/wgcf-account.toml 文件保存账户信息)**

```
wgcf register
```

- **生成 WireGuard 配置文件 ( /root/wgcf-profile.conf )**

```
wgcf generate
```

**编辑 WireGuard 配置文件**

- **双栈 WARP 全局网络置换**

**双栈 WARP 全局网络是指 IPv4 和 IPv6 都通过 WARP 网络对外进行网络访问，默认生成的 WireGuard 配置文件就是这个效果。由于默认的配置文件没有外部对 VPS 本机 IP 网络访问的相关路由规则，一旦直接使用 VPS 就会直接失联，所以我们还需要对配置文件进行修改。路由规则需要添加在配置文件的 [Interface] 和 [Peer] 之间的位置，以下是路由规则(包含<>的部分一起替换掉)示例**

```
PostUp = ip -4 rule add from <替换IPv4地址> lookup main
PostDown = ip -4 rule delete from <替换IPv4地址> lookup main
PostUp = ip -6 rule add from <替换IPv6地址> lookup main
PostDown = ip -6 rule delete from <替换IPv6地址> lookup main
```

**此外配置文件中默认的 DNS 是 1.1.1.1，由于它将替换掉系统中的 DNS 设置 (/etc/resolv.conf)，请根据实际情况来进行替换，或者直接删除 DNS 这行。以下配置供参考**

```
DNS = 8.8.8.8,8.8.4.4,2001:4860:4860::8888,2001:4860:4860::8844
```

- **IPv4 Only 服务器添加 WARP IPv6 网络支持**

**解析 engage.cloudflareclient.com 的 ip**

```
nslookup engage.cloudflareclient.com
```

**解析的结果一般为以下两个**

```
162.159.192.1
2606:4700:d0::a29f:c001
```

**将配置文件中的 engage.cloudflareclient.com 替换为 162.159.192.1 ，并删除 AllowedIPs = 0.0.0.0/0 。即配置文件中 [Peer] 部分为**

```
[Peer]
PublicKey = bmXOC+F1FxEMF9dyiK2H5/1SUtzH0JuVo51h2wPfgyo=
AllowedIPs = ::/0
Endpoint = 162.159.192.1:2408
```

原理：AllowedIPs = ::/0参数使得 IPv6 的流量均被 WireGuard 接管，让 IPv6 的流量通过 WARP IPv4 节点以 NAT 的方式访问外部 IPv6 网络。

- **IPv6 Only 服务器添加 WARP IPv4 网络支持**

**将配置文件中的 engage.cloudflareclient.com 替换为 [2606:4700:d0::a29f:c001]，并删除 AllowedIPs = ::/0。即配置文件中 [Peer] 部分为**

```
[Peer]
PublicKey = bmXOC+F1FxEMF9dyiK2H5/1SUtzH0JuVo51h2wPfgyo=
AllowedIPs = 0.0.0.0/0
Endpoint = [2606:4700:d0::a29f:c001]:2408
```

原理：AllowedIPs = 0.0.0.0/0参数使得 IPv4 的流量均被 WireGuard 接管，让 IPv4 的流量通过 WARP IPv6 节点以 NAT 的方式访问外部 IPv4 网络。

**此外配置文件中默认的 DNS 是 1.1.1.1，由于是 IPv4 地址，故查询请求会经由 WARP 节点发出。由于它将替换掉系统中的 DNS 设置 (/etc/resolv.conf)，为了防止当节点发生故障时 DNS 请求无法发出，建议替换为 IPv6 地址的 DNS 优先，或者直接删除 DNS 这行。以下配置供参考**

```
DNS = 2001:4860:4860::8888,2001:4860:4860::8844,8.8.8.8,8.8.4.4
```

**将 WireGuard 配置文件复制到 `/etc/wireguard/` 并命名为 `wgcf.conf`**

```
cp wgcf-profile.conf /etc/wireguard/wgcf.conf
```

**测试 WireGuard 网络接口**

- **开启网络接口（命令中的 wgcf 对应的是配置文件 wgcf.conf 的文件名前缀）**

```
wg-quick up wgcf
```

执行执行 `ip a` 命令，此时能看到名为wgcf的网络接口

- **执行以下命令检查是否连通**

```
# IPv4 Only VPS
curl -6 ip.p3terx.com
# IPv6 Only VPS
curl -4 ip.p3terx.com
```

- **测试完成后关闭相关接口，因为这样配置只是临时性的**

```
wg-quick down wgcf
```

**启用 WireGuard 网络接口**

- **启用守护进程**

```
systemctl start wg-quick@wgcf
```

- **设置开机启动**

```
systemctl enable wg-quick@wgcf
```

**查看 WireGuard 网络接口状态**

```
systemctl status wg-quick@wgcf
```

**设置设置IPv4 与 IPv6 网络优先级**

优先级不是一定要设置的，这仅限于 VPS 本身对外发起的网络请求。

- **IPv4 优先**

编辑 /etc/gai.conf 文件，在末尾添加下面这行配置

```
precedence ::ffff:0:0/96  100
```

- **IPv6 优先**

编辑 /etc/gai.conf 文件，在末尾添加下面这行配置

```
label 2002::/16   2
```

- **验证优先级**

执行 `curl ip.p3terx.com` 命令，显示 IPv4 地址则代表 IPv4 优先，否则为 IPv6 优先。

---

## 感谢项目

* https://github.com/XTLS/Xray-core
* https://github.com/XTLS/Xray-examples
* https://www.nginx.com/
* https://github.com/acmesh-official/acme.sh
* https://github.com/ViRb3/wgcf

## 参考博客和教程

* https://github.com/XTLS/Xray-examples
* https://xtls.github.io/Xray-docs-next/document/level-0/
* https://www.v2rayssr.com/warp-netflix.html
* https://www.v2rayssr.com/xray-nginx.html
* https://p3terx.com/archives/use-cloudflare-warp-to-add-extra-ipv4-or-ipv6-network-support-to-vps-servers-for-free.html
* https://p3terx.com/archives/debian-linux-vps-server-wireguard-installation-tutorial.html
* https://guide.v2fly.org/advanced/tls.html
* https://github.com/acmesh-official/acme.sh/wiki/Install-preparations
* https://github.com/acmesh-official/acme.sh/wiki/%E8%AF%B4%E6%98%8E
* https://github.com/acmesh-official/acme.sh/wiki/Change-default-CA-to-ZeroSSL
* https://github.com/acmesh-official/acme.sh/wiki/Options-and-Params

