# InfoSec-Lab2-apollo




## Architecture Design

模拟实现128位信息的加密通信过程。

实验设计包含三方：A，B，C。A与B之间进行128位信息的加密交换。C负责为A和B各生成一对公私钥对，并分别把对应的私钥传给A和B，并把他们的公钥公开。实现本实验需求的架构设计如下图所示，

{{< image src="https://image.p1nant0m.com/%E6%9C%AA%E5%91%BD%E5%90%8D%E7%BB%98%E5%9B%BE.jpg" caption="项目数据流图" src_s="https://image.p1nant0m.com/%E6%9C%AA%E5%91%BD%E5%90%8D%E7%BB%98%E5%9B%BE.jpg" src_l="https://image.p1nant0m.com/%E6%9C%AA%E5%91%BD%E5%90%8D%E7%BB%98%E5%9B%BE.jpg" >}}

## Protocol Design

在该实验需求中，ServerA与ServerB之间使用Asymmetric Encryption Scheme进行加密通信，ServerC充当密钥签发中心，负责为ServerA与ServerB生成公私钥对，并将私钥分发给对应的Client。ServerA与ServerB的地位是等价的，都是希望进行加密通信的对等主体，也都是向KDC申请密钥的主体，在下述分析过程中不再区分ServerA与ServerB，我们统称为通信方。



### ServerC——Key Distribution Center

ServerC中KDC的常驻守护进程监听来自`443`端口的请求，负责为到访的Client签发公私钥对，并为其提供其所想要进行加密通信方的网络地址以及该主体对应的公钥。

- 集群中所有主体在进行加密通信之前都需要先在KDC中进行注册。KDC会为发起请求的Client签发一对公私钥并记录下该Client信息接收端的网络地址信息与其对应的公钥信息。
- 该注册过程我们假设是在高权限安全环境下进行的，也就是意味着能够进行注册的用户都是合法的。
- KDC与Client之间通信基于TLS（或使用协议一中的认证密钥交换方案建立会话密钥），因此KDC向Client发送的私钥敏感信息是受到保护的。
- KDC假设所有的Client都能够保存好自己被分配的私钥，若私钥遗失则无法从Client端发起重置密钥的请求（无法验证Client身份），此时必须联系KDC管理员申请重置密钥。
- 拥有合法私钥的Client可以向KDC发起重置密钥请求，但是这并不被推荐。因为其它的Client在本地缓存了对等方的公钥，这个公钥在这时已经失效了。此时它需要重新访问KDC获取其最新的公钥，以实现与该Client的加密通信。
- KDC使用自签名证书来证书自己的合法身份，在本次实验中为了方便起见就不将该自签名证书添加至系统证书根信任链中，我们通过在客户端中设置`InsecureSkipVerify: true` 来免除系统对证书的校验。



#### **KDC与Client间通信的报文格式**

Client 请求报文 (向KDC注册自身)

```
[0x00-0x01] Action (0x01) 向KDC表示这是一个请求注册报文

[0x01-0x0b] ClientName_Hasing_Hex_Truncated_10

[0x0b-0x1f] padding_receiverAddr

[0x1f-0x24] padding_receiverPort

[0x24-0x25] EOF Bytes (0x00)
```

KDC响应报文

```
[0x00-0x80] PrivateKey
```

Client请求报文

```
[0x00-0x01] Action (0x02) 向KDC表示这是一个请求获取通信方公钥与网络地址信息的报文

[0x01-0x0b] ClientName_Hasing_Hex_Truncated_10

[0x01-0x15] RemoteServer_Hasing_Hex_Truncated_10

[0x15-0x35] Signature of ClientName_Hasing _Hex_Truncated_10
```

KDC响应报文

```
[0x00-0x01] ValidMark (0x01表示请求Client与其想要进行通信的对象已经在KDC注册并且签名通过验证)

[0x01-0x15] TargetServer.receiverAddr **Padding**

[0x15-0x1a] targetServer.receiverPort **Padding**

[0x1a-0x9a] TargetServer.receiverPublicKey
```



### Communication Peers

在本实验需求中，ServerA与ServerB都是希望进行加密通信的主体，它们向KDC注册自身并通过KDC发现其它同僚。现以ServerA想要向ServerB发起一次加密通信的过程为例，讲解本实验设计。

ServerA若想要与ServerB进行通信，就必须在KDC中注册自身，同理，ServerB也需要在KDC中注册自身，这样ServerA与ServerB就加入了这个集群之中。

- ServerA通过TLS加密保护的信道接收由KDC生成的私钥并将其保存至本地。
- ServerA事先知道ServerB在这个集群中的唯一标识符（Identification）。通过这个标识符，ServerA能够向KDC发起一次询问，请求ServerB所在的网络地址、ServerB信息接收端所监听的端口以及ServerB的公钥等相关信息。于此同时，ServerA向KDC发送的报文中携带了ServerA对其Identification的签名用以证明它是该集群中的合法成员。
- ServerA若能够通过KDC的验证则会收到来自KDC的响应。该响应报文中包含了ServerB的网络地址以及其公钥等信息。
- ServerA利用ServerB的网络地址向其发起一次TCP连接，可以注意到该TCP连接并没有使用TLS进行保护，所有在这个连接中传输的数据都是未经加密的。
- ServerA使用ServerB的公钥对其要发送给ServerB的数据进行加密，并将加密后的数据推入TCP建立的信道之中。
- ServerB监听到来自ServerA的连接请求，与ServerA建立起TCP连接，并接收由ServerA中传输过来的加密数据
- ServerB使用自己的私钥对来自ServerA的加密数据进行解密，并将解密后的明文写入文件和标准输出流中（写入文件中的数据以16进制的形式表示，至于输出流中的数据是转换为ASCII字符后进行展示，我们这里假设传输的数据都是可打印字符）
- 这样ServerA与ServerB之间一次加密通信过程就完成了。



## Implementation

本实验基于Golang Cobra框架实现了一个CLI应用，具体的代码可参考仓库。这里仅给出使用本CLI应用进行ServerA与ServerB加密通信，以及KDC密钥签发的过程。

首先我们需要启动KDC的守护进程，该进程在443端口监听来自Client的请求，为客户端提供密钥签发注册以及远程服务器发现的服务。

使用命令：`sudo ./kdc` 启动KDC服务，监听443端口。


{{< image src="https://image.p1nant0m.com/image-20220315221341126.png" caption="启动 KDC 服务" src_s="https://image.p1nant0m.com/image-20220315221341126.png" src_l="https://image.p1nant0m.com/image-20220315221341126.png">}}

使用命令：`./apollo regist -k localhost -l 443 -t localhost -p 7070 -n serverA -f privateK_A.key ` 向KDC发起注册请求，其中`-k` 表示KDC的`Hostname` ；`-l` 表示KDC服务的监听端口；`-t` 表示ServerA Receiver的`Hostname` ；`-p` 表示ServerA Receiver服务的监听端口；`-n` 表述ServerA在集群中的标识符；`-f` 表述存储KDC签发私钥的文件路径。

{{< image src="https://image.p1nant0m.com/image_1_1.png" caption="向KDC发起注册请求" src_s="https://image.p1nant0m.com/image_1_1.png" src_l="https://image.p1nant0m.com/image_1_1.png" >}}

同理，使用命令：`./apollo regist -k localhost -l 443 -t localhost -p 7071 -n serverB -f privateK_B.key` 向KDC注册ServerB。

我们可以在KDC输出的日志中看到相关的注册信息，

{{< image src="https://image.p1nant0m.com/image_22.png" caption="在KDC输出的日志中查看相关的注册信息" src_s="https://image.p1nant0m.com/image_22.png" src_l="https://image.p1nant0m.com/image_22.png" >}}

紧接着我们令ServerA向KDC获取ServerB的网络地址信息以及公钥，使用命令：`./apollo getremote --cn serverA -r serverB -f remote_B -p ./privateK_A.key` 。其中，`--cn` 表示当前Client在集群中的标识符，`-r` 标识ServerA想要请求的ServerB在集群中的标识符，`-f` 表示若KDC成功验证ServerA的合法性返回的ServerB网络地址以及公钥信息保存的文件路径，`-p` 表示ServerA对应的私钥文件，该文件用于对于唯一标识进行签名，表明ServerA的合法性。该命令的执行结果如下，

{{< image src="https://image.p1nant0m.com/image_33.png" caption="ServerA向KDC获取ServerB的网络地址信息以及公钥" src_s="https://image.p1nant0m.com/image_33.png" src_l="https://image.p1nant0m.com/image_33.png" >}}

利用上述过程获取到的信息，ServerA与ServerB已经能够进行通信了，首先我们创建一个文件`helloB` 并在其中写入如下内容，

{{< image src="https://image.p1nant0m.com/image_44.png" caption="向待发送文件写入内容" src_s="https://image.p1nant0m.com/image_44.png" src_l="https://image.p1nant0m.com/image_44.png" >}}

使用命令：`./apollo setupreceiver --privk-file=privateK_B.key --listen-port=7071` 开启ServerB Receiver服务，如下所示，

{{< image src="https://image.p1nant0m.com/image_55.png" caption="开启ServerB Receiver服务" src_s="https://image.p1nant0m.com/image_55.png" src_l="https://image.p1nant0m.com/image_55.png" >}}

使用命令：`./apollo send --remote-info=./remote_B --sending-file=./helloB` 向ServerB Receiver发送加密数据，如下所示，


{{< image src="https://image.p1nant0m.com/image_66.png" caption="向ServerB Receiver发送加密数据" src_s="https://image.p1nant0m.com/image_66.png" src_l="https://image.p1nant0m.com/image_66.png" >}}

在 ServerB Receiver 端我们可以看见解密后的内容如下所示，

{{< image src="https://image.p1nant0m.com/image_77.png" caption="在 ServerB Receiver 端 查看解密后的内容" src_s="https://image.p1nant0m.com/image_77.png" src_l="https://image.p1nant0m.com/image_77.png" >}}

至此我们便完成了ServerA与ServerB依靠KDC的密钥分发与服务发现实现利用Asymmetric Encryption Scheme进行加密通信的需求。

## Helpful Commands

**Openssl Usage**

```Bash
# TLS Client tools
openssl s_client -connect localhost:443

# generate a private key with the correct length
openssl genrsa -out server.key 2048

# optional: create a self-signed certificate
openssl req -new -x509 -key server.key -out server.pem -days 3650

```

**RegistA**

```Bash
./apollo regist -k localhost -l 443 -t localhost -p 7070 -n serverA -f privateK_A.key
```

**RegistB**

```Bash
./apollo regist -k localhost -l 443 -t localhost -p 7071 -n serverB -f privateK_B.key
```

**GetRemoteInfo**

```Bash
./apollo getremote --cn serverA -r serverB -f remote_B -p ./privateK_A.key
```

**Send**

```Bash
./apollo send --remote-info=./remote_B --sending-file=./helloB
```

**Setupreceiver**

```Bash
./apollo setupreceiver --privk-file=privateK_B.key --listen-port=7071
```



## Chatting

这个实验的设置确实有点一头雾水。你既然选择实现一个KDC（Key Distribution Center）作为一个密钥签发与管理中心，那么就意味着这是一个中心化的密钥管理存储解决方案，那为什么还要选择Asymmetric Encryption Scheme去做呢？作为KDC应该去实现Symmetric Encryption Scheme为End-to-End提供会话密钥与存储服务，当用户A需要与用户B进行加密通信时，KDC应该为这两个用户生成这样一个密钥并为他们维护这样一个密钥。当然这里简化了KDC与用户进行交互授权认证的过程，具体的可以参考[kerberos](https://web.mit.edu/kerberos/#what_is)协议。

再者使用Asymmetric Encryption Scheme去加密数据效率是不高的，并且对于数据量较大的场景来说原生的加密方案将无法使用（比如RSA方案中，所加密的数据必须能够映射到所使用的群中的一个元素）Asymmetric Encryption Scheme最典型的应用场景是数字签名和密钥交换。比如我们可以在本实验的场景之下，将ServerC直接移除，仅依靠ServerA与ServerB就可以实现加密通信。ServerA与ServerB可以使用DHKE方案去为通信双方在公开信道下生成一个“秘密”，将该“秘密”经过sha256散列之后作为AES_GCM的密钥，使用Symmetric Encryption Scheme去实现双方的加密通信。这便是TLS/SSL的高度简化版本，有关TLS/SSL的规范可以[点击这里](https://datatracker.ietf.org/doc/html/rfc5246)。

> A typical operation with a KDC involves a request from a user to use some service. The KDC will use cryptographic techniques to authenticate requesting users as themselves. It will also check whether an individual user has the right to access the service requested. If the authenticated user meets all prescribed conditions, the KDC can issue a ticket permitting access.
>
> KDCs mostly operate with [symmetric encryption](https://en.wikipedia.org/wiki/Symmetric_encryption).
>
> -- From WiKi

吐槽归吐槽，本实验刚好让我实践一下刚刚接触的Cobra框架，虽然用到的东西不多，使用方法也不够优雅，可能还有些凌乱，但能进行一次实践终归是有裨益的。害，不知什么时候能够将6.824的Raft K/V写完呢？

