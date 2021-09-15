---
title: Ubuntu安装OpenVPN服务器
date: 2021-09-16 00:09:36
tags:
- Ubuntu
- OpenVPN
---

# Ubuntu安装OpenVPN服务器

## 1. 安装OpenVPN（2.4.7-1ubuntu2.20.04.3）、easy-rsa（3.0.6-1）

```shell
sudo apt install openvpn
sudo apt install easy-rsa
```

## 2. 生成证书

参考文档`/usr/share/doc/easy-rsa/README.Debian`

### 1. 生成证书文件夹

```shell
make-cadir ca
cd ca
vim vars
```

一般不需要修改`vars`中生成证书的变量。

### 1. 初始化

```shell
./easyrsa init-pki
```

### 2. 创建根证书

```shell
./easyrsa build-ca          # 要求输入密码
./easyrsa build-ca nopass   # 不使用密码

# 要求输入 Common Name，或者直接回车，不输入
$ Common Name (eg: your user, host, or server name) [Easy-RSA CA]:mycroft
```

生成的根证书地址：`/path/to/ca/pki/ca.crt`

### 3. 生成服务器证书和私钥

```shell
./easyrsa build-server-full vpnserver nopass
```

生成的服务器证书和私钥地址：
`/path/to/ca/pki/issued/vpnserver.crt`、
`/path/to/ca/pki/private/vpnserver.key`

### 4. 生成客户端证书和私钥

```shell
./easyrsa build-client-full vpnclient nopass
```

生成的客户端证书和私钥地址：
`/path/to/ca/pki/issued/vpnclient.crt`、
`/path/to/ca/pki/private/vpnclient.key`

### 5. 创建Diffie-Hellman 密钥交换算法

```shell
./easyrsa gen-dh
```

时间会有点长，耐心等待。
生成的dh地址：`/path/to/ca/pki/dh.pem`。

### 6. 创建Hmac 消息摘要算法

```shell
/usr/sbin/openvpn --genkey --secret ta.key
```

生成的HMAC地址：`/path/to/ca/ta.key`。

## 配置VPN

### 1. 将生成的服务端文件拷贝到`OpenVPN`配置文件夹

```shell
cp pki/ca.crt pki/issued/vpnserver.crt pki/private/vpnserver.key pki/dh.pem ta.key /etc/openvpn/
```

### 2. 服务端配置

```shell
# 复制配置样例，并解压出来 server.conf
cp /usr/share/doc/openvpn/examples/sample-config-files/server.conf.gz /etc/openvpn/

# 解压
gzip -d server.conf.gz

# 编辑配置文件
vim server.conf
```

### 3. 修改服务端配置参数

参考配置参数：

- 传输协议：`proto udp`或`proto tcp`；`udp`速度更快
- 端口：`port 1194`
- 根证书：`ca ca.crt`
- 服务端证书：`cert vpnserver.crt`
- 服务端私钥：`key vpnserver.key`
- `dh`密钥交换算法：`dh dh2048.pem`（实际这里`dh.pem`）
- `HMAC`消息摘要算法：`tls-auth ta.key 0`（如果使用内联行嵌入`ta.key`的内容，则需要通过`key-direction 0`添加第二个参数）
- 开启流量通过VPN网络网关：`push "redirect-gateway def1 bypass-dhcp"`
- log日志：`log-append  /var/log/openvpn/openvpn.log`
- 客户端可能使用相同的证书-私钥对：`duplicate-cn`
- 当服务端关闭时，客户端不再自动重连：`explicit-exit-notify 0`

### 4. 启动OpenVPN服务端进程

```shell
sudo systemctl start openvpn@server
```

## 配置客户端参数

复制文件`/usr/share/doc/openvpn/examples/sample-config-files/client.conf`到一个文件夹，对其进行编辑

修改其中的配置参数：

- 服务器地址：`remote 192.168.0.30 1194`
- 从服务器地址列表（可能有多个）中随机选择：`remote-random`
- 配置根证书（内联行）：把`ca`证书内容放在`<ca></ca>`中
- 配置客户端证书（内联行）：把`vpnclient.crt`内容放在`<cert></cert>`中
- 配置客户端私钥内容（内联行）：把`vpnclient.key`内容放在`<key></key>`中
- 配置HMAC（内联行）：把`ta.key`内容放在`<tls-auth></tls-auth>`中，`key-direction 1`
