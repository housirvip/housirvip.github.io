---
title: 搭建纯净 DNS 服务
date: 2017-08-24 12:53:47
tags:
  - Dns
  - Linux
  - 树莓派
---

## 起因

* 国内近年来 DNS 污染严重，甚至就连一些地方宽带商都胡乱投毒，整一些花里胡哨的广告页面，什么，你想要干净清爽的网络浏览？

![](http://static.nicesite.vip/2018-06-12-090052.jpg)

## 经过

### 安装开发环境

```bash
sudo apt-get install git build-essential cmake -y
```

* 一般的 ubuntu 和 debian 系统都可以做到，但是我之前用的 ubuntu mate 会报错，找不到 build-essential，那么使用 aptitude 来解决依赖项

```bash
sudo apt-get install aptitude -y
sudo aptitude install build-essential -y
```

### 安装依赖项

* 这里列举下一般容易缺少的依赖项
- [LibPcap](http://www.tcpdump.org/#latest-release)
- [Libsodium](https://github.com/jedisct1/libsodium)
- [OpenSSL](https://www.openssl.org)
- Flex
- Bison
* 这些依赖项可以去选择手动编译，也可以直接

```bash
sudo apt-get install -y libsodium-dev libpcap-dev libssl-dev flex bison
```

### 安装 [Pcap_DNSProxy](https://github.com/chengr28/Pcap_DNSProxy)

```bash
cd
git clone https://github.com/chengr28/Pcap_DNSProxy.git
cd Pcap_DNSProxy/Source/Auxiliary/Scripts
chomd a+x CMake_Build.sh
sudo ./CMake_Build.sh
```

![](http://static.nicesite.vip/2018-06-12-090241.jpg)

* 编译完成后，生成的程序在`Pcap_DNSProxy/Source/Release`下，可以选择把这个 cp 到`/usr/local/`下，我是没改动位置就直接这么用了

```bash
cd 
cd Pcap_DNSProxy/Source/Release
sudo vi Pcap_DNSProxy.service
```

* WorkingDirectory填写你的程序位置文件夹
* ExecStart填写程序位置文件夹跟上程序名

{% note warning %}
[Service]
Type=forking
WorkingDirectory=/home/pi/Pcap_DNSProxy/Source/Release
ExecStart=/home/pi/Pcap_DNSProxy/Source/Release/Pcap_DNSProxy
GuessMainPID=yes
Restart=on-failure
RestartSec=15s
{% endnote %}

```bash
sudo ./Linux_Install.Systemd.sh
# 以后想重启，关闭什么的
sudo service Pcap_DNSProxy restart
sudo service Pcap_DNSProxy stop
sudo service Pcap_DNSProxy status
```

## 结果

* dig 命令测试 DNS 回到

{% note success %}
pi@raspberrypi:~/Pcap_DNSProxy/Source/Release $ dig @127.0.0.1 www.google.com

; <<>> DiG 9.10.3-P4-Raspbian <<>> @127.0.0.1 www.google.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 56212
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;wwW.GOoGlE.coM.                        IN      A

;; ANSWER SECTION:
wwW.GOoGlE.coM.         266     IN      A       216.58.200.36

;; Query time: 241 msec
;; SERVER: 127.0.0.1#53(127.0.0.1)
;; WHEN: Thu Aug 24 06:34:31 UTC 2017
;; MSG SIZE  rcvd: 59
{% endnote %}

![](http://static.nicesite.vip/2018-06-12-090255.jpg)

