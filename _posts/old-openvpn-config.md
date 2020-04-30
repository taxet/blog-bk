---
title: OpenVpn服务器配置
date: 2017-12-27 10:25:34
tags: 
- 运维
- OpenVpn
category: 运维
---

## openvpn下载与安装

1. 下载openvpn:

```bash
sudo apt-get install openvpn
```
2. 服务器端下载证书相关依赖:
```bash
sudo apt-get install easy-rsa
```

## 生成服务器端证书

先建立生成证书的目录：
```bash
make-cadir ~/openvpn-ca
cd ~/openvpn-ca
```
进入*vars*文件，找到如下相关语句：
```bash
export KEY_COUNTRY="US"
export KEY_PROVINCE="CA"
export KEY_CITY="SanFrancisco"
export KEY_ORG="Fort-Funston"
export KEY_EMAIL="me@myhost.mydomain"
export KEY_OU="MyOrganizationalUnit"
```
修改成对应的适合自己的数值

在文件最后加入如下语句设置自己服务器的名称：
```bash
export KEY_NAME="server"
```
更新变量：
```bash
source vars
```
清楚之前生成的证书和冗余资料：
```bash
./clean-all
```
生成根证书：
```bash
./build-ca
```
这里会让你确认vars里的相关信息，如无修改一路回车就可以了

生成服务器证书：
```bash
./build-key-server server
```
最后会让你确认到期时间，同时也会让你再次确认，两次都选择y即可

生成混淆码(Diffie-Hellman keys)：
```bash
./build-dh
```

此时，在keys目录下会生成证书相关文件，将ca.crt dh2048.pem server.crt server.key四个文件复制到/etc/openvpn/目录下
```bash
cd keys
sudo cp ca.crt dh2048.pem server.crt server.key /etc/openvpn/
cd ..
```

## 服务器端配置文件

在/etc/openvpn/下建立server.conf文件

*server.conf*
```bash
# '#'和','后面的是注释

# 选择openVPN监视的ip地址，默认为全部监视
# 一般不设置
;local a.b.c.d

# 端口号
port 1194

# 使用TCP还是UDP
proto tcp
;proto udp

# 选择VPN模式
# A TUN device is used mostly for VPN tunnels where only IP-traffic is used. A TAP device allows full Ethernet frames to be passed over the OpenVPN tunnel. hence providing support for non-ip based protocols such as IPX and AppleTalk.
# tun基于ip协议通讯，tap基于以太网通道
dev tap
;dev tun

# 证书文件
ca ca.crt
cert server.crt
key server.key #这个文件应当保密

# D-H文件
dh dh2048.pem

# tun模式下，设置用户的ip域
;server 10.8.0.0 255.255.255.0

# 用户虚拟ip映射文件,一般不做修改
ifconfig-pool-persist ipp.txt

# tap模式下，设置网桥的域
# 第一个为本机地址，第二个为网关地址，第三个和第四个是分配给用户的地址区间
server-bridge 10.8.0.0 255.255.255.0 10.8.0.100 10.8.0.150
# 可以设置多个
;server-bridge

# 设置用户可访问的ip段路由
push "route 10.8.0.0 255.255.255.0"
;push "route 192.168.20.0 255.255.255.255"
# 设置用户的转发网关和DNS
;push "redirect-gateway def1 bypass-dhcp"
;push "dhcp-option DNS 208.67.222.222"

# 加上下面这条可以允许用户之间链接
client-to-client

# 对于一个证书是否允许多个用户使用
duplicate-cn 

# 启动后需要执行的脚本
;up /etc/openvpn/up.sh
# 关闭钱需要执行的脚本
;down /etc/openvpn/down.sh

# 生存确认，第一个数值为寄秒ping一次，第二个数值为多少秒之后确认为超时，一般保持默认
keepalive 10 120

# 额外安全文件，防止block DoS攻击和UDP端口溢出
# 通过 openvpn --genkey --secret ta.key 创建
;tls-auth ta.key 0 #这个文件需要保密

# 加密方法
;cipher BF-CBC #默认
;cipher AES-128-CBC   # AES
;cipher DES-EDE3-CBC  # Triple-DES

# 不知道干嘛的，保持默认
comp-lzo

# 最大用户连接数
；max-client 100

# 用户以及用户组
user nobody
group nogroup

# 日志配置相关，保持默认
persist-key
persist-tun
status openvpn-status.log
verb 3
```

## 转发，网桥与防火墙设置
- 设置转发

在/etc/sysctl.conf文件里设置
```bash
net.ipv4.ip_forward=1
```
之后重启：
```bash
sudo sysctl -p
```

- 设置网桥

tap模式下需要设置网桥

首先安装网桥相关依赖
```bash
apt-get install bridge-utils
```
之后，在/etc/network/interfaces里设置网关的配置
(这里假设默认网卡的名称为eth0)
```bash
auto br0
iface br0 inet static
    address 192.168.111.254
    netmask 255.255.255.0
    network 192.168.111.0
    broadcast 192.168.111.255
    bridge_ports eth0
```
添加网关，将默认网卡放进网关，重置路由
```bash
sudo brctl addbr br0
sudo brctl addif br0 eth0
sudo ifconfig eth0 0.0.0.0 promisc up
sudo route add default gw 192.168.111.253 br0
```
同时openvpn启动好之后要将启动好的虚拟网卡加入网关

*/etc/openvpn/up.sh*
```bash
#!/bin/bash

br="br0"
tap="tap0"

for t in $tap; do
    openvpn --mktun --dev $t
done

for t in $tap; do
    brctl addif $br $t
done

for t in $tap; do
    ifconfig $t 0.0.0.0 promisc up
done
```

- 防火墙设置

关掉就可以了

## 开启openVpn
```bash
sudo systemctl star openvpn@server
```
如果需要开机时也开启：
```bash
sudo systemctl enable openvpn@server
```

## 客户端证书生成
在 *~/openvpn-ca/* 目录下
```bash
./build-key client1
```
之后，在 *./keys/*目录下会生成client1.crt 和client1.key两个文件

## 客户端配置文件
*client.ovpn*
```bash
# 注明是客户端配置文件
client

# 网络以及连接方法，需要与服务器端一致
dev tap
proto tcp

# 远程服务器外部地址和端口号
remote remote-address 1194

# 重试次数
resolv-retry infinite

# 是否需要绑定本地端口，一般是不绑定
nobind

# 用户用户组
user nobody
group nogroup

# 不知道干嘛的，需要与客户端的设置一致
persist-key
persist-tun

# 代理服务器设置
;http-proxy-retry # retry on connection failures
;http-proxy [proxy server] [proxy port]

# 证书文件
# 上述步骤完成后，在~/openvpn-ca/keys/目录下复制ca.crt, client1.crt, client1.key到客户端来
ca ca.crt
cert client1.crt
key client1.key

# 客户端证书确认
remote-cert-tls server

# tls-auth 的key文件
;tls-auth ta.key 1

# 日志相关
comp-lzo
verb 3
```
之后运行
```bash
sudo openvpn --config client1.opvn
```

###TLS相关

待补充

### 参考资料
[DigitOcean -- How To Set Up an OpenVPN Server on Ubuntu 16.04 ](https://www.digitalocean.com/community/tutorials/how-to-set-up-an-openvpn-server-on-ubuntu-16-04#step-2-set-up-the-ca-directory)

[Bridging in OpenVPN](https://community.openvpn.net/openvpn/wiki/OpenVPNBridging)

[ethernet bridging](https://openvpn.net/index.php/open-source/documentation/miscellaneous/76-ethernet-bridging.html)