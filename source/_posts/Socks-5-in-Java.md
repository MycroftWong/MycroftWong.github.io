---
title: Socks 5 in Java
date: 2022-09-10 12:05:20
categories:
  - Socks 5
tags:
  - Socks 5
  - Proxy
---

# Socks 5 in Java

通过梳理 Socks 5 在 Java 中的实现，去理解 Socks 5 代理的流程。

## HTTP

以 `OkhttpClient` 为例，了解基于 `Socket` 的 `HTTP` 通信方式。

当使用 `Proxy.NO_PROXY`，即 `type = Proxy.Type.DIRECT` 时

1. DNS 查询：`Dns.lookup`
2. 创建 Socket：`new Socket()`
3. 连接 endpoint：`Socket.connect(InetSocketAddress)`
4. 交换数据：`Socket.getInputStream()`、`Socket.getOutputStream()`
5. 关闭 Socket：`Socket.close()`

### 1. DNS 查询

这一步其实可以和创建 Socket 合并，如果 `new Socket()` 时传入服务器地址，则会调用 `InetAddress.getAllByName(String host)` 进行操作系统 `DNS` 查询。

将这一步拆分出来可以扩展更多的功能。

### 2. 创建 Socket

对于 `DIRECT` 模式，通过 `SocketFactory.createSocket()` 进行创建。

工厂类同样可以扩展出更多的功能。

### 3. 连接 endpoint

`endpoint` 一般指的是目标服务器地址，`SocketAddress` 类型，但其只有一个实现类 `InetSocketAddress`。

`SocketAddress` 表示传输层的地址 `IP Socket Address (IP address + port number)`。

### 4. 交换数据

连接成功之后，就可以通过 `OutputStream` 和 `InputStream` 写入和读取数据了。

### 5. 关闭 Socket

`Socket` 在使用之后，会进行复用，如果一定时间没有再使用，则会进行关闭。

所以不是每次使用 `Socket` 之后就会立马关闭。

## Socket 5

同样以 `OkhttpClient` 为例，了解基于其使用 `Socks 5` 代理时的通信过程。

需要使用 `Socket 5`，则需要创建一个 `Proxy` 代理类，指定代理服务器的地址。

1. DNS 查询：`Dns.lookup`
2. 创建 Socket：`new Socket(Proxy proxy)`
3. 连接 endpoint：`Socket.connect(InetSocketAddress)`
4. 交换数据：`Socket.getInputStream()`、`Socket.getOutputStream()`
5. 关闭 Socket：`Socket.close()`

和 `Proxy.Type.DIRECT` 几乎完全相同。但需要注意的两点：

- `DNS` 查询的目的是查询出 `Socks 5` 服务器的 `IP` 地址，而真正需要访问的目标服务器地址，实际上是通过 `Socks 5` 服务器去处理的。如果 `Socks 5`服务器用的是 `IP` 地址，这一步其实可以忽略。
- 在创建 `Socket` 时，将 `Proxy` 作为参数传入进去。

`Socket` 内部有一个 `SocketImpl` 对象，实际的连接和数据交换都是通它来完成的，`Socket` 类则提供调用的接口。

当使用 `Proxy.Type.DIRECT` 时，`Socket` 内部通过 `SocketImpl.createPlatformSocketImpl(false)` 创建 `SocketImpl`。这个 `SocketImpl` 内部就是和 `native` 层进行交互了，后续建立连接，交换数据，都是使用它来完成。

而当使用 `Proxy.Type.SOCKS` 时，`Socket` 通过 `SocketImpl.createPlatformSocketImpl(false)` 之后，再封装了一层 `SocksSocketImpl`，并主要重写了 `connect(SocketAddress endpoint, int timeout)` 方法，用于负责 `Socks 5` 协议握手。

这相当于如果是一个 `HTTP` 的数据交换，实际上是运行在 `Socks 5` 的 `Socket` 上，而这个 `Socket` 使用系统底层的 `Socket`，先完成了 `Socks 5` 的连接、认证过程，再进行后续的数据交换。

而如果没有使用 `Socks 5`，则使用 `SocketImpl.createPlatformSocketImpl(false)` 创建的 `SocketImpl` 进行连接，后进行数据交换。

## 以本机 Socks 5 代理服务器为例

当客户端 A 通过本机 `Socks 5` 代理服务器访问公共网络时（假设使用默认端口 1080）：

1. 与 Socks 5 代理服务器建立连接，如客户端使用随机端口 40001 与 Socks 5 端口 1080 建立 TCP 连接（进行三次握手）
2. 进行 Socks 5 协议握手，进行认证
3. Socks 5 协议握手，认证完成之后，后续的数据流量，就直接写入即可

### DNS

此时，客户端不再解析目标服务器的 `IP`，即不会再进行 `DNS` 查询，这个工作是交由 `Socks 5` 服务器来完成：在 `Socks 5` 协议握手阶段，客户端会将目标服务器的 `IP v4`、`IP v6` 或 `Domain`，发送给服务器。如果服务器收到的是 `IP`，则不再需要查询了，但由于 `DNS` 污染、`CDN` 等的存在，可能会导致客户端 `DNS` 查询失败，或查询到的服务器离 `Socks 5` 服务器比较远。所以这一工作一般会交由 `Socks 5` 服务器来完成。 `DNS` 查询完成后，就可以创建与目标服务器连接的 `Socket`了，然后将客户端的内容，发送给目标服务器，并将从目标服务器收到的响应，发送给客户端。

### 连接

同时，如果客户端需要访问多个服务器，或同时需要与一个服务器建立多个连接，那实际上会与 `Socks 5` 建立多个连接，同样的，`Socks 5` 服务器会与目标服务器一一建立连接。

### UDP

`Socks 5` 有 `UDP relay` 的功能，`UDP relay` 的实现，实际上是，客户端仍然通过 `TCP` 与 `Socks 5` 建立连接，但会告诉服务器，我需要你帮我发送 `UDP` 的数据。`Socks 5` 服务器收到此请求后，会创建 `UDP` `DatagramSocket`，并将监听的 `UDP` 端口，发送给客户端。客户端收到此端口后，就可以与服务器进行 `UDP` 的数据交换了。服务器收到 `UDP` 数据包后，会对其进行拆封，获取目标地址、端口和需要发送的数据，构建自己的 `UDP` 请求，并使用同一个 `DatagramSocket` 发送给目标服务器，收到目标服务器的响应后，同样需要进行封装，使用同一个 `DatagramSocket` 发送给客户端。客户端收到 `UDP` 数据后就应该主动关闭一开始的 `TCP` 连接了。

## Socks 5

`Socks 5` 的协议握手过程可以查看 [socks5协议详解](https://www.ddhigh.com/2019/08/24/socks5-protocol.html)，这里完全不再赘述。

需要添加的一点是，发送的认证信息、域名长度都不是固定的，所以在这些信息之前需要添加一个表示长度的字节，这一点并没有在 [rfc1928](https://www.ietf.org/rfc/rfc1928.txt) 中体现。

`Socks 5` 提供3种可用的命令，用于三种不同的需求：

- CONNECT 客户端通过 `Socks 5` 服务器，向目标服务器发送 `TCP` 请求消息，并接收响应
- BIND 用于客户端接收从服务器的请求。FTP 被动模式是一个著名的例子
- UDP ASSOCIATE 客户端通过 `Socks 5` 服务器，向目标服务器发送 `UDP` 请求消息，并转发响应

### CONNECT

`CONNECT` 是 `Socks 5` 使用最广泛的命令。由于 `HTTP`、`HTTPS` 协议是基于 `TCP` 的，所以大部分 `HttpClient` 都会提供 `Socks 5` 的功能。

比较重要的是，使用 `CONNECT` 命令，客户端就不再通过 `DNS` 解析目标服务器的 `IP`，当然可能 `Socks 5` 服务器是在同一台电脑上。

### BIND

`BIND` 很难理解它的用途，目前 `HTTP` 协议都能完成基本上所有的工作，而且目前有丰富的 `HttpClient` 实现，那么 `BIND` 基本上就不会使用到了。

### UDP ASSOCIATE

又称 `UDP relay`，这也是 `Socks 5` 能够代理 `UDP` 流量的原因。

`UDP relay` 的主要流程：

1. 客户端与 `Socks 5` 服务器建立连接，完成认证
2. 客户端发送 `UDP` 命令，其中包括目标服务器地址
3. 服务端收到后，建立一个 `UDP` 接收器，准备接收来自客户端的 `UDP` 请求，并通过 `Socks 5` 的 `Socket` 将 `IP + port number` 发送给客户端
4. 客户端收到后，使用 `DatagramSocket` 像服务器发送 `UDP` 请求，其中包括：
   - RSV 保留字段，2个字节，X'0000'
   - FRAG 当前片段编号，1个字节，通常不支持，则为 X'00'
   - ATYP 以下地址的地址类型，1个字节
      - IP V4 地址: X'01'
      - DOMAINNAME 域名: X'03'
      - IP V6 地址: X'04'
   - DST.ADDR 目标服务器地址，变长，IP V4 地址 4个字节；IP V6 地址 16个字节；如果是域名，第一个字节为域名长度，后面字节是域名
   - DST.PORT 目标服务器端口，2个字节
   - DATA 用户数据，变长，第一个字节为数据长度，后面字节是数据
5. `Socks 5` 服务器 `UDP` 服务端收到数据后，解析目标服务器地址，并使用 `UDP` 将用户数据发送给目标服务器
6. `Socks 5` 收到从目标服务器的数据后，同样使用第4步，客户端发送给客户端的数据格式，将其转发给客户端
7. 客户端收到数据后，对数据进行解析，并主动关闭 `Socket 5` 连接
8. 服务器关闭连接

需要注意的是，`UDP` 数据包的目标地址，只可能是 `IP` 地址，如果使用 `DatagramSocket` 的程序员传入的是一个域名，那么实际上先进行系统 `DNS` 查询。此时，`DNS` 查询并没有走代理，所以一般情况下，`DNS` 查询这一步都会拆分出来，同样使用 `UDP ASSOCIATE` 功能，先进行 `DNS` 查询，然后再完成之前的请求。

另外，`UDP ASSOCIATE` 是“一次性”的，在完成一次 `UDP` 转发之后，都会关闭 `Socks 5` 连接，如果有新的请求，那么需要重新建立连接。

## 参考文章

[rfc1928](https://www.ietf.org/rfc/rfc1928.txt)

[Socks5工作原理与搭建](https://blog.csdn.net/CZD__CZD/article/details/120493190)

[浅谈在代理环境中的 DNS 解析行为](https://blog.skk.moe/post/what-happend-to-dns-in-proxy/)

[sockslib](https://github.com/fengyouchao/sockslib) - A Java library of SOCKS5 protocol including client and server

[java-socks-proxy-server](https://github.com/bbottema/java-socks-proxy-server) - Java SOCKS 4/5 server implementation for Java

[socks5协议详解](https://www.ddhigh.com/2019/08/24/socks5-protocol.html)
