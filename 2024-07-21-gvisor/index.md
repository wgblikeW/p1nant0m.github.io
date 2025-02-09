# [✨Exploration Series] gVisor: 以 GoogleCTF2023 赛题为契机，深入了解 gVisor - 其 1


{{< admonition type=quote title="胡言乱语" open=false >}}
计算机科学领域的任何问题都可以通过增加一个间接的中间层来解决。
{{< /admonition >}}

## Isolation Level from High To Low

Virtual Machine → Kata Container → gVisor → Container → JVM (Java Virtual Machine) 

Hardware Simulation → Hypervisor (KVM) → Kernel In Userspace → Namespcae + Cgroups + Chroot → Apps

## Sandbox Overview

沙盒(Sandbox)技术是一种通过创建隔离的执行环境来运行程序或代码的方式，用于实现资源隔离以创造安全的运行时环境，使得可以在信任域内的机器执行不受信的用户代码或程序。沙箱能够限制程序的行为，有效防止程序运行过程中对主机系统造成的潜在损害。

机器级虚拟化，如 KVM 和 Xen，通过虚拟机监视器（VMM）向客户机内核暴露虚拟化硬件。这种虚拟化硬件通常是透明的（准虚拟化），并且可以使用额外的机制来提高客户机和主机之间的可见性。

在不同的虚拟机中运行容器可以提供很好的隔离性、兼容性和性能（尽管嵌套虚拟化方案可能会在性能方面会带来挑战），但对于容器来说，实现这种级别的虚拟化方案通常需要额外的代理和代理程序，并且可能产生更大的资源占用和导致更慢的容器启动时间。


{{< image src="https://image.p1nant0m.xyz/202407292316118.png" caption="Virtual hardware Sandboxing" src_s="https://image.p1nant0m.xyz/202407292316118.png" src_l="https://image.p1nant0m.xyz/202407292316118.png" height=100px width=100px >}}

基于规则的执行，如 SECcomp、SELinux 和 AppArmor，允许为应用程序或容器指定细粒度的安全策略。这些方案通常依赖于在宿主机内核中实现的 Hook（LSM）来强制执行规则。这种方式如果能够充分对应用所需要使用到的系统调用特征进行建模，从而使应用的威胁暴露面足够小，那么这是一种极好的方式来将应用沙箱化并保持原生性能。

然而，在实践中，为任意的、从前未知的应用程序可靠地定义策略可能是极其困难的任务，这使得这种方法在普遍应用上面临挑战。

虽然直接使用基于规则的系统调用限制在实践上很难进行规模化，但是其通常与额外的防御层结合使用，以实现纵深防御 (**Defense In Depth**)。

{{< image src="https://image.p1nant0m.xyz/202407292324150.png" caption="Rule Based Sandboxing" src_s="https://image.p1nant0m.xyz/202407292324150.png" src_l="https://image.p1nant0m.xyz/202407292324150.png" >}}

## Introduction: gVisor

作为沙盒技术的一个先进实现，Google 使用 {{< raw >}}<font color="#0080ff"><b>Go</b></font>{{< /raw >}} 开发的开源项目 gVisor，其提供了一种创新的容器隔离方法。gVisor 旨在结合传统虚拟机的隔离性和容器的轻量级特性，通过自己的内核代理 "Sentry" 来拦截和处理容器内的系统调用（gVisor 引入了一层新的安全层级，通过在用户态重新实现内核功能达到构建安全边界的目的），这极大地减少了容器与宿主操作系统的直接交互，从而降低了攻击面，提升了安全性。

gVisor 通过拦截应用程序的系统调用，并充当客户机内核，无需通过虚拟化硬件进行转换。gVisor 可以被看作是客户机内核和 VMM 的结合，或者被看作是加强版的 SECComp。这种架构允许它提供灵活的资源占用（即基于线程和内存映射，而不是固定的客户机物理资源），同时也降低了虚拟化的固定开销。然而，这是以降低应用程序兼容性和增加每个系统调用的开销为代价的。

{{< image src="https://image.p1nant0m.xyz/202407292333690.png" caption="gVisor Sandboxing" src_s="https://image.p1nant0m.xyz/202407292333690.png" src_l="https://image.p1nant0m.xyz/202407292333690.png" >}}

### gVisor 基础架构

{{< image src="https://image.p1nant0m.xyz/202407292335721.png" caption="gVisor Architecture" src_s="https://image.p1nant0m.xyz/202407292335721.png" src_l="https://image.p1nant0m.xyz/202407292335721.png" >}}

- **Sentry**

    Sentry 可以被看作是一个应用程序内核。Sentry 实现了应用程序所需的所有内核功能，包括：系统调用、信号传递、内存管理和页面错误逻辑处理、管理线程模型等。

    当应用程序进行系统调用时，平台（Platform 是 gVisor 中的一个抽象）会将系统调用重定向到 Sentry，Sentry 将完成必要的工作来提供为上层应用程序提供服务。需要注意的是，Sentry 不会将系统调用传递给宿主机内核。作为一个用户空间应用程序，Sentry 会进行一些宿主机系统调用以支持其操作，但**它不允许应用程序直接控制它进行系统调用**。例如，Sentry 无法直接打开文件；超出沙箱的文件系统操作（不是内部 `/proc` 文件、管道等）会被发送到 Gofer 处理。
- **Gofer**

    Gofer 是一个标准的宿主进程，它与每个容器一起启动，并通过套接字或共享内存通道使用 [9P](https://en.wikipedia.org/wiki/9P_(protocol)) 协议与 Sentry 通信。Sentry 进程在一个受限的 seccomp 规则监视器中启动，没有访问文件系统资源的权限。Gofer 调解对这些资源的所有访问，提供额外的隔离层级。


### Platform

平台的选择取决于 runsc 执行的上下文。通常，当在裸机（不在虚拟机内）上运行时，KVM 平台将提供最佳性能。当在虚拟机内运行，或者在没有虚拟化支持的机器上运行时，systrap 平台是更好的选择。

{{< image src="https://image.p1nant0m.xyz/202407302218817.png" caption="Different Platform In gVisor" src_s="https://image.p1nant0m.xyz/202407302218817.png" src_l="https://image.p1nant0m.xyz/202407302218817.png" >}}

- **KVM**: KVM 平台使用内核的 KVM 功能，允许 Sentry 充当客户操作系统和 VMM。KVM 平台在裸机设置上运行效果最佳。虽然没有虚拟化硬件层——沙箱保留了进程模型——但 gVisor 利用现代处理器上可用的虚拟化扩展来提高地址空间切换的隔离性和性能
- **systrap**: systrap 平台依赖于 seccomp 的 `SECCOMP_RET_TRAP` 特性来拦截系统调用。这使得内核向触发线程发送 `SIGSYS` 信号，然后将控制权交给 gVisor 来处理系统调用。

{{< admonition type=info title="systrap 是如何工作的？" open=false >}}
Results in the kernel sending a `SIGSYS` signal to the triggering task without executing the system call. 
{{< /admonition >}}

下列代码片段展示了 gVisor 以 `systrap` 作为 Platform，启动沙盒化进程的入口。
```go
// createStub creates a fresh stub processes.
//
// Precondition: the runtime OS thread must be locked.
func createStub() (*thread, error) {
  // When creating the new child process, we specify SIGKILL as the
  // signal to deliver when the child exits. We never expect a subprocess
  // to exit; they are pooled and reused. This is done to ensure that if
  // a subprocess is OOM-killed, this process (and all other stubs,
  // transitively) will be killed as well. It's simply not possible to
  // safely handle a single stub getting killed: the exact state of
  // execution is unknown and not recoverable.
  return attachedThread(uintptr(unix.SIGKILL)|unix.CLONE_FILES, linux.SECCOMP_RET_TRAP)
}
```

## Defense In Depth

🧀 **瑞士奶酪理论**（英语：Swiss Cheese Model），又称**乳酪理论**或**瑞士起司理论**，是英国[曼彻斯特大学](https://zh.wikipedia.org/wiki/曼徹斯特大學)教授[詹姆斯·瑞森](https://zh.wikipedia.org/w/index.php?title=詹姆斯·瑞森&action=edit&redlink=1)（James Reason）于1990年提出的关于意外发生的风险分析与控管的模型。
主要是讲，[瑞士起司](https://zh.wikipedia.org/wiki/瑞士起司)在制造与[发酵](https://zh.wikipedia.org/wiki/發酵)过程当中，很自然的会产生小孔洞。如果把许多片起司重叠在一起，正常情况下，每片起司的空洞位置不同，光线透不过。只有在很极端的情况下，空洞刚好连成一直线，才会让光线透过去。**导致严重事故发生的从来都不是因为某个单独的原因，而是多个问题同时出现**。

{{< image src="https://image.p1nant0m.xyz/202407302228208.png" caption="The Swiss Cheese Cyber Security Defense-in-Depth Model" src_s="https://image.p1nant0m.xyz/202407302228208.png" src_l="https://image.p1nant0m.xyz/202407302228208.png" >}}

上图的一片片乳酪就像是我们在安全工作中落实的一项项规章制度以及在系统的方方面面部署的各种不同用途的安全工具和平台,但是不是所有的规章制度都能够按照预期100%的执行，并且安全产品也不能够覆盖所有的场景，如此以一来各个层级便出现了漏洞，当存在一条链路能够贯穿所有的层级的漏洞时，安全事件便发生了。

在网络安全或计算机系统安全领域，纵深防御体系（Defense in Depth）是一种这么一种缓解策略，旨在通过多层安全措施来减少单点故障的风险，通过在不同的层级投入资源以试图提高攻击者成功利用疏忽/漏洞并对系统造成广泛影响的成本。

## Performance Overhead 性能开销

**Memory Access**

该图表展示了通过 `sysbench` 测量的内存传输速率，

{{< image src="https://image.p1nant0m.xyz/202407302253582.png" caption="Memory Access Benchmark" src_s="https://image.p1nant0m.xyz/202407302253582.png" src_l="https://image.p1nant0m.xyz/202407302253582.png" >}}

**Memory Usage**

该图展示了基于三个样本应用程序运行的内存开销。这个测试是通过运行许多容器实例（50 个，在 redis 的情况下是5个）并计算 Host 在运行前后的可用内存的平均值。

第一个应用程序是一个休眠实例（`sleep`）—— 一个什么也不做的简单应用程序，第二个应用程序是一个合成的 Node 应用程序，它导入了一定数量的模块并监听请求。第三个应用程序是一个类似的 Ruby 应用程序，它也做同样的事情。最后，我们包括了一个存储大约 1GB 数据的 Redis 实例。在所有情况下，沙箱本身占用一小部分内存，大部分是固定的内存开销。

{{< image src="https://image.p1nant0m.xyz/202407302256324.png" caption="Memory Usage Benchmark" src_s="https://image.p1nant0m.xyz/202407302256324.png" src_l="https://image.p1nant0m.xyz/202407302256324.png" >}}

**CPU performance**

该图表展示了 `sysbench` 对每秒 CPU 事件的测量。每秒事件数是基于一个 CPU 密集型的任务场景：计算指定范围内的所有质数。我们注意到 `runsc` 没有造成性能损失，因为在两种情况下代码都是以原生方式执行的。

{{< image src="https://image.p1nant0m.xyz/202407302303128.png" caption="CPU Events" src_s="https://image.p1nant0m.xyz/202407302303128.png" src_l="https://image.p1nant0m.xyz/202407302303128.png" >}}

该图表显示了一个 TensorFlow 工作负载，其中进行的任务为卷积神经网络（CNN）训练。所指示的时间包括工作负载的完整启动和运行时间。

{{< image src="https://image.p1nant0m.xyz/202407302306498.png" caption="CNN Training Time" src_s="https://image.p1nant0m.xyz/202407302306498.png" src_l="https://image.p1nant0m.xyz/202407302306498.png" >}}

**Start-Up Time**

该图表显示了通过 Docker 启动容器所需的总时间。这个基准测试使用了三个不同的应用程序。首先，一个执行 `true` 命令的 Alpine Linux 容器。其次，一个加载了多个模块并绑定 HTTP 服务器的 Node 应用程序。最后是一个同样加载了多个模块并绑定 HTTP 服务器的 Ruby 应用程序。当目标服务能够正常响应用户请求时视为容器启动完毕，这时候停止我们的实验计时。

{{< image src="https://image.p1nant0m.xyz/202407302309820.png" caption="Start-Up Time" src_s="https://image.p1nant0m.xyz/202407302309820.png" src_l="https://image.p1nant0m.xyz/202407302309820.png" >}}

**System calls**

该图表展示了在不同平台上进行原始系统调用所需的时间。测试是通过一个自定义的二进制文件实现的，该文件执行大量的系统调用并计算所需的平均时间。

{{< image src="https://image.p1nant0m.xyz/202407302315199.png" caption="不同容器运行时实现在进行系统调用所需要的平均时间" src_s="https://image.p1nant0m.xyz/202407302315199.png" src_l="https://image.p1nant0m.xyz/202407302315199.png" >}}

例如，Redis 是一个在用户空间中执行任务相对较轻的应用程序：通常它的任务无外乎从连接的套接字读取数据，读取或修改一些数据，并将结果写回套接字。下图显示了运行一系列全面基准测试的结果。我们可以看到，轻量操作会带来较大的开销，而重型任务，如 LRANGE，在应用程序中完成更多工作，相对开销较小。

{{< image src="https://image.p1nant0m.xyz/202407312103643.png" caption="在 gVisor 沙盒中运行 Redis 并执行一系列 Redis 操作的 RPS" src_s="https://image.p1nant0m.xyz/202407312103643.png" src_l="https://image.p1nant0m.xyz/202407312103643.png" >}}

**Network**

该图表显示了两个实例之间 [`iperf`](https://iperf.fr/) 测试的结果。对于上传场景，指定的运行时间是 [`iperf`](https://iperf.fr/) 客户端使用的；在下载情况中，指定的运行时间是服务器使用的。测试中的另一个端点始终使用原生运行时间。

{{< image src="https://image.p1nant0m.xyz/202407312108229.png" caption="不同运行时下的上传/下载带宽" src_s="https://image.p1nant0m.xyz/202407312108229.png" src_l="https://image.p1nant0m.xyz/202407312108229.png" >}}

该图表显示了简单的 node 和 ruby Web 服务在接收到请求时渲染模板的结果。因为这些合成基准测试每个请求做的工作很少，就跟 Redis 的情况一样，存在较高的性能开销。在实践中，应用程序做的工作越多，结构性成本的影响就越小。

{{< image src="https://image.p1nant0m.xyz/202407312254369.png" caption="不同运行时下的简单 Web 服务 RPS" src_s="https://image.p1nant0m.xyz/202407312254369.png" src_l="https://image.p1nant0m.xyz/202407312254369.png" >}}

**File System**

下图展示了 `fio` 对磁盘进行读写操作的结果。在这种情况下，磁盘很快就成为了性能瓶颈。

{{< image src="https://image.p1nant0m.xyz/202407312258895.png" caption="不同运行时下各种 I/O 操作的带宽" src_s="https://image.p1nant0m.xyz/202407312258895.png" src_l="https://image.p1nant0m.xyz/202407312258895.png" >}}

该图表展示了使用 `tmpfs` 挂载时的原始I/O性能，这在 `runsc` 的情况下时沙箱内部的行为。通常，这些操作与在内存中复制数据的开销相似，我们并没有看到 VFS（虚拟文件系统）操作相关的带来的性能影响。

{{< image src="https://image.p1nant0m.xyz/202407312300889.png" caption="不同运行时下在 tmpfs 挂载时的原始 I/O 性能表现" src_s="https://image.p1nant0m.xyz/202407312300889.png" src_l="https://image.p1nant0m.xyz/202407312300889.png" >}}

VFS 操作的高成本可能在执行许多此类操作的基准测试中显现出来，尤其是在服务请求的热点路径上。该图表显示了使用 gVisor 提供小段静态内容的结果，可以预见结果较差。这个工作负载代表了 Apache 从一个容器镜像向运行 ApacheBench 的客户端提供单个**100k**大小的文件，测试基准设置了多个不同级别的并发数量。高性能开销主要来自于需要改进的 VFS 实现，其存在多个内部序列化点（因为所有请求都读取同一个文件），同时一些网络栈性能问题也影响了这个基准测试。

{{< image src="https://image.p1nant0m.xyz/202407312309703.png" caption="不同运行时下在不同并发级别对同一静态文件进行访问时的传输速率" src_s="https://image.p1nant0m.xyz/202407312309703.png" src_l="https://image.p1nant0m.xyz/202407312309703.png" >}}

对于那些受限于原始磁盘I/O以及计算和文件系统混合操作的基准测试来说，文件系统操作开销不那么成问题。该图表显示了一个 `ffmpeg` 容器启动、加载并转码一个 27MB 输入视频所需的总时间。

{{< image src="https://image.p1nant0m.xyz/202407312322221.png" caption="不同运行时启动、加载并转码一个 27MB 输入视频所需的总时间" src_s="https://image.p1nant0m.xyz/202407312322221.png" src_l="https://image.p1nant0m.xyz/202407312322221.png" >}}

下图展示了gVisor不同的沙箱化架构方案，以及性能与安全性之间的Trade-off（牺牲一定的安全性保证以获取性能的提高）。

{{< image src="https://image.p1nant0m.xyz/202407312323708.png" caption="Security & Performance Trade-off with different architecture" src_s="https://image.p1nant0m.xyz/202407312323708.png" src_l="https://image.p1nant0m.xyz/202407312323708.png" >}}


## 📦 沙箱 （gVisor） 使用场景

沙箱可以应用于恶意软件分析，多租场景下的敏感数据保护，LLMs 代码执行 Agent 运行环境，应用运行时监控/Enforcement 等应用场景。

### 容器化应用运行时（Runtime）行为限制（Enforcement）

例如，在对容器化应用进行基于策略的运行时限制中，可以对 gVisor 项目进行二次开发，向各系统调用入口处增加策略校验的逻辑，以达到对于容器运行时行为进行限制的目的。

{{< raw >}}<font color="#FF0000"><b>File System Restriction</b></font>{{< /raw >}}
- Define the access permission of file system (ABAC, RBAC)
- Following Policy
    - rootfs read-only: most of them can't be write to
    - writeablePaths: The dir/file can be write to
    - maskedPaths: The dir/file that can't be read by process

```json
{
  "file": {
    "writeablePaths": ["/tmp"],
    "maskedPaths": ["/mnt"]
  }
}
```

{{< raw >}}<font color="#FF0000"><b>Network Restriction</b></font>{{< /raw >}}
- Define the network action which can perform
- Following Policy
    - No networking at all
    - Limit outgoing IP/Port
    - Limit outgoing domain name
    - Limit local listen port

{{< raw >}}<font color="#FF0000"><b>Process Restriction</b></font>{{< /raw >}}
- In most of the situation only one sandboxed program is executed
- No reverse shell, no attack tool can be run
- Executable full path as policy

一个简单的硬编码规则对应用运行时行为进行限制的实现如下所示，

```go
func openat(t *kernel.Task, dirfd int32, pathAddr hostarch.Addr, flags uint32, mode uint) (uintptr, *kernel.SyscallControl, error) {
  // 自定义访问控制策略
  pathname, err := t.CopyInString(pathAddr, linux.PATH_MAX)
  if err != nil {
    return 0, nil, err
  }
  if pathname == "/etc/passwd" {
    return 0, nil, linuxerr.EACCES
  }

  path, err := copyInPath(t, pathAddr)
  if err != nil {
    return 0, nil, err
  }
  tpop, err := getTaskPathOperation(t, dirfd, path, disallowEmptyPath, shouldFollowFinalSymlink(flags&linux.O_NOFOLLOW == 0))
  if err != nil {
    return 0, nil, err
  }
  defer tpop.Release(t)

  file, err := t.Kernel().VFS().OpenAt(t, t.Credentials(), &tpop.pop, &vfs.OpenOptions{
    Flags: flags | linux.O_LARGEFILE,
    Mode:  linux.FileMode(mode & (0777 | linux.S_ISUID | linux.S_ISGID | linux.S_ISVTX) &^ t.FSContext().Umask()),
  })
  if err != nil {
    return 0, nil, err
  }
  defer file.DecRef(t)

  fd, err := t.NewFDFrom(0, file, kernel.FDFlags{
    CloseOnExec: flags&linux.O_CLOEXEC != 0,
  })
  return uintptr(fd), nil, err
}
```

在 `opeant` 系统调用的处理逻辑处添加如下代码片段，使得我们能够限制容器化应用打开 `/etc/passwd` 文件的行为。
```go
  // 自定义访问控制策略
  pathname, err := t.CopyInString(pathAddr, linux.PATH_MAX)
  if err != nil {
    return 0, nil, err
  }
  if pathname == "/etc/passwd" {
    return 0, nil, linuxerr.EACCES
  }

```

在更复杂的分布式场景下支持自定义、动态、可扩展、热加载特性并提供尽可能完备的规则语法可以让 gVisor 成为容器安全平/产品的技术基座，提供实现容器运行时防护与观测的能力。

### Malicious Binary Analysis

[Runtime Monitoring](https://gvisor.dev/docs/tutorials/falco/) + 异常行为检测引擎 + ATT&CK 知识库



