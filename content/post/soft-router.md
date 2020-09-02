---
title: 在树莓派上搭建软路由
date: 2017-08-16 18:34:30
tags: 
  - 软路由 
  - 树莓派 
  - Linux
---

## 出发点
* 很早的时候一直在用网件 wndr4300 刷了 [openwrt](https://openwrt.org/) 以及 [lede](https://lede-project.org/) 这种开放式的路由系统，一直沉迷于无界浏览无法自拔。因为笔者最近在学习的一些东西比较新，在国内看不到详细的文档，能上 [google](https://www.google.com) 当然是最好的。

![](http://static.nicesite.vip/2018-06-12-091349.jpg)

## 要求提升

* 当然不能满足于可以访问，作为一个年轻人就要有敢于折腾的精神。速度慢点的路由已经满足不了笔者的需求了。刚好前几天买了个树莓派3，不折腾下似乎有点过分了啊。

![](http://static.nicesite.vip/2018-06-12-091423.jpg)

## 准备工具

### 需要的设备

- 电脑一台
- 树莓派 or 其他linux平台
- 网卡2个，有线，无线，usb网卡都行
- 网线一根

### 需要的软件

- dnsmasq or isc-dhcp-server
- overture or DNScrypt
- v2ray shadowsocks or shadowsocks-rss

### 网络拓扑

![网络拓扑图](http://static.nicesite.vip/2018-06-12-091444.jpg)

## 开始动手

### 配置网卡

* 先把提供本机dhcp服务的网卡地址配置好，给自己赋予一个静态地址


```bash
sudo vim /etc/network/interfaces

# 加入以下内容
auto lo
iface lo inet loopback
auto eth0
allow-hotplug eth0
iface eth0 inet dhcp
auto eth1
allow-hotplug eth1
iface eth1 inet static
        address 192.168.20.1
        network 192.168.20.0
        netmask 255.255.255.0
        broadcast 192.168.20.255
```

* 解释下`eth0`和`eth1`是我的树莓派上两张网卡，这个网卡名字不一定都是这种，根据自己的网卡名称进行修改，输入`ifconfig`来查看

* `eth0`是连接上级路由的，它的网段是`192.168.10.0`，我直接连上就能上网了，把这张卡设置`dhcp`，自动从上级路由获取ip，如果是无线网卡一般是`wlan0`

* `eth1`是本机用来提供路由服务的，设置自己的网段，并且赋予自己固定ip

### 配置 dnsmasq

* 首先安装 dnsmasq 来提供 dhcp 和 dns 缓存服务

```bash
sudo apt-get install dnsmasq
```

* 安装完成后查看安装状态

{% note success %}
pi@raspberrypi:~ $ dnsmasq -v
Dnsmasq version 2.76  Copyright (c) 2000-2016 Simon Kelley
Compile time options: IPv6 GNU-getopt DBus i18n IDN DHCP DHCPv6 no-Lua TFTP conntrack ipset auth DNSSEC loop-detect inotify 
{% endnote %}

* 编辑 dnsmasq 配置文件

```bash
# 不需要提供dhcp服务的网卡，一般是连接外网的那张卡
no-dhcp-interface=eth0
dhcp-range=192.168.20.10,192.168.20.100,72h

cache-size=102400
log-facility=/var/log/dnsmasq/dnsmasq.log

# 指定返回给客户端的ttl时间，一般不需要设置
max-ttl=28800
# 本地 hosts 文件的缓存时间，通常不要求缓存本地，这样更改hosts文件后就即时生效。
local-ttl=360
# 对于上游返回的值没有ttl时，dnsmasq给一个默认的ttl，一般不需要设置
neg-ttl=28800
# 设置在缓存中的条目的最大 TTL。
max-cache-ttl=28800
# 设置在缓存中的条目的最小 TTL。
min-cache-ttl=10800

conf-dir=/etc/dnsmasq.d/
```

* ~~如果你是使用了`isc-dhcp-server`这种额外的 dhcp 服务器，那么就把上面~~

```
no-dhcp-interface=eth0
dhcp-range=192.168.20.10,192.168.20.100,72h
```

* ~~这两句删掉吧，如果不设定`dhcp-range` dnsmasq 默认是不开启 dhcp 的，然后这么配置`isc-dhcp-server`~~

```bash
sudo vim /etc/dhcp/dhcpd.conf

# 添加以下内容
ddns-update-style none;
default-lease-time 600;
max-lease-time 7200;

# 网段
subnet 192.168.20.0 netmask 255.255.255.0 {
  
  # DHCP 分配ip范围
  range 192.168.20.10 192.168.20.100;
  
  # DHCP 给接入设备分配的网关
  option routers 192.168.20.1;
  option subnet-mask 255.255.255.0;
  option broadcast-address 192.168.20.255;
  
  # 分配的DNS服务器
  option domain-name-servers 192.168.20.1;
}
```

* 建议直接用 dnsmasq 好了，忽略上面两条内容即可

* 添加 gfwlist

```bash
cd 
git clone https://github.com/cokebar/gfwlist2dnsmasq.git
cd gfwlist2dnsmasq
chmod a+x gfwlist2dnsmasq.sh
./gfwlist2dnsmasq -o gfwlist.conf -s gfwlist
sudo cp gfwlist.conf /etc/dnsmasq.d/
```

* 在`gfwlist.conf`中都是这样的存在

```
server=/030buy.com/127.0.0.1#5300
ipset=/030buy.com/gfwlist
server=/0rz.tw/127.0.0.1#5300
ipset=/0rz.tw/gfwlist
...
```

* `127.0.0.1#5300`代表匹配的域名通过这个服务器和端口进行 DNS 解析，而这个端口是多少取决于一会你的防污染 DNS 服务器开启的端口，刚生成的是 5353 端口，我的因为被占用了，查找并替换改成 5300

```bash
# 开启dnsmasq
sudo service dnsmasq start

# 设置开机自启动
sudo systemctl enable dnsmasq
```

### 配置转发服务

```bash
sudo vim /etc/sysctl.conf

# 修改一项，把前面注释号#去掉，或者自己加
net.ipv4.ip_forward = 1

# 让上述修改立刻生效
sysctl -p
```

* `iptables`开启网段定向转发

```bash
sudo iptables -t nat -A POSTROUTING -s 192.168.20.0/24 -o eth0 -j MASQUERADE

# 上面这条命令重启就没了，写入文件中保证长期有效
sudo vim /etc/rc.local

# 在 exit0 前加入
iptables -t nat -A POSTROUTING -s 192.168.20.0/24 -o eth0 -j MASQUERADE
```

* ok， 至此，一个具有正常功能的普通路由就可以使用了，连接的设备也是可以上网的了

### ~~配置无污染的上级 DNS~~

* 其实多此一举，v2ray 自带无污染dns服务

* ~~安装 [overture](https://github.com/shawn1m/overture), 去 github 下载 [release-binary](https://github.com/shawn1m/overture/releases), 我的树莓派下载 [overture-linux-arm.zip](https://github.com/shawn1m/overture/releases/download/1.3.5.2/overture-linux-arm.zip)~~

* ~~修改配置文件 `config.json`~~

```json
{
  "BindAddress": ":5300",
  "PrimaryDNS": [
    {
      "Name": "DNSPod",
      "Address": "119.29.29.29:53",
      "Protocol": "udp",
      "SOCKS5Address": "",
      "Timeout": 6,
      "EDNSClientSubnet": {
        "Policy": "disable",
        "ExternalIP": ""
      }
    },
    {
      "Name": "114",
      "Address": "114.114.114.114:53",
      "Protocol": "udp",
      "Timeout": 6,
      "EDNSClientSubnet": {
        "Policy": "disable",
        "ExternalIP": ""
      }
    }
  ],
  "AlternativeDNS": [
    {
      "Name": "OpenDNS",
      "Address": "208.67.222.222:443",
      "Protocol": "tcp",
      "SOCKS5Address": "",
      "Timeout": 6,
      "EDNSClientSubnet": {
        "Policy": "disable",
        "ExternalIP": ""
      }
    }
  ],
  "OnlyPrimaryDNS": false,
  "RedirectIPv6Record": true,
  "IPNetworkFile": "./ip_network_sample",
  "DomainFile": "./domain_sample",
  "DomainBase64Decode": true,
  "HostsFile": "./hosts_sample",
  "MinimumTTL": 0,
  "CacheSize" : 0,
  "RejectQtype": [255]
}
```

* ~~然后直接运行就行 `./overture-linux-arm`, 这样不能关闭这次连接的终端，否则就关闭了，可以选择开一个 screen 来运行，或者`./overture-linux-arm &`~~

* [DNSCrypt](https://dnscrypt.org/) 可以代替 overture 不过配置麻烦，读者可以自行研究，功能更加强大稳定

### 配置 v2ray

* 先去安装 [v2ray](https://www.v2ray.com/)

```bash
cd 
wget https://install.direct/go.sh
sudo bash go.sh
sudo vim /etc/v2ray/config.json
```

* 客户端配置出一个socks5代理端口 8080 (以备不时之需)、一个透明代理端口 1080

```json
{
  "log": {
    "access": "/var/log/v2ray/access.log",
    "error": "/var/log/v2ray/error.log",
    "loglevel": "warning"
  },
  "inbound": {
    "port": 8080,
    "listen": "192.168.20.1",
    "protocol": "socks",
    "settings": {
      "auth": "noauth",
      "udp": false
    }
  },
  "inboundDetour": [
    {
      "protocol": "dokodemo-door",
      "port":1080,
      "settings":{
        "network": "tcp,udp",
        "timeout": 30,
        "followRedirect": true
      }
    },
    {
      "protocol": "dokodemo-door",
      "port":5300,
      "settings":{
        "address":"8.8.8.8",
        "port":53,
        "network": "udp",
        "timeout": 30,
        "followRedirect": false
      }
    }
  ],
  "outbound": {
    "protocol": "vmess",
    "settings": {
      "vnext": [
        {
          "address": "yourserver.com",
          "port": 12345,
          "users": [
            {
              "id": "1sb4165e-1234-4310-9d57-a8a2994r5e0d",
              "alterId": 32,
              "security": "auto"
            }
          ]
        }
      ]
    },
    "streamSettings":{
        "network":"kcp",
        "kcpSettings": {
          "mtu": 1350,
          "tti": 20,
          "uplinkCapacity": 5,
          "downlinkCapacity": 100,
          "congestion": false,
          "readBufferSize": 1,
          "writeBufferSize": 1,
          "header": {
            "type": "none"
          }
        }
      },
    "mux": {
      "enabled": true
    }
  },
  "outboundDetour": [
    {
      "protocol": "freedom",
      "settings": {},
      "tag": "direct"
    }
  ],
  "dns": {
    "servers": [
      "8.8.8.8",
      "localhost"
    ]
  }
}
```

*  iptables 和 ipset 设置

```bash
sudo apt-get install ipset
sudo ipset -N gfwlist iphash
sudo iptables -t nat -A PREROUTING -p tcp -m set --match-set gfwlist dst -j REDIRECT --to-port 1080
sudo iptables -t nat -A OUTPUT -p tcp -m set --match-set gfwlist dst -j REDIRECT --to-port 1080

# 把路由配置写入文件中，长久保存
sudo vim /etc/rc.local

# 在exit0前把这三句加上
ipset -N gfwlist iphash
iptables -t nat -A PREROUTING -p tcp -m set --match-set gfwlist dst -j REDIRECT --to-port 1080
iptables -t nat -A OUTPUT -p tcp -m set --match-set gfwlist dst -j REDIRECT --to-port 1080
```

* 开启无限制的世界

```bash
sudo service v2ray start

# 把 v2ray 设置为开机自启动
sudo systemctl enable v2ray
```

## 效果展示

![树莓派连接](http://static.nicesite.vip/2018-06-12-091503.jpg)

{% note success %} 
housirvipdeiMac:~ housirvip$ dig www.google.com

; <<>> DiG 9.8.3-P1 <<>> www.google.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 23438
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;www.google.com.                        IN      A

;; ANSWER SECTION:
www.google.com.         1978    IN      A       172.217.26.36

;; Query time: 17 msec
;; SERVER: 192.168.20.1#53(192.168.20.1)
;; WHEN: Thu Aug 17 23:59:10 2017
;; MSG SIZE  rcvd: 48 
{% endnote %}

![google ip 验证](http://static.nicesite.vip/2018-06-12-091511.jpg)

![youtube测试](http://static.nicesite.vip/2018-06-12-091522.jpg)





