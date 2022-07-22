---
title:      "XDP-tutorial 学习如何编写 eBPF XDP 程序"
subtitle:   "跟着 xdp-tutorial repo 以及 Linux repo bpf/samples 从零开始学习编写 eBPF XDP 程序"
date:       2022-07-18 07:00:00
author:     "p1nant0m"
header-img: "img/in-post/bg2.jpg"
lightgallery: true
tags:
    - eBPF
    - XDP
    - tutorial
---

## XDP-Tutorial

> ⭐ 本文章仍然在不断的更新中，因此文章结构以及内容可能会时常发生变化。同时本领域也是作者刚刚涉及的领域，难免在文本内容中出现错误，还望指正，大家一同学习共勉。

XDP-Tutorial 是一个 Github 上的 repo，皆在指导人们如何遵循最基本的步骤，高效地实现为内核中的 XDP 系统进行编程。我会随着这个课程的步伐与大家一同探寻 Linux 内核世界的奥秘以及如何使用高性能的 XDP 程序对网络数据报进行处理。随着学习的深入，我会列出理解每一节课程的编程实现所需要前置知识，查漏补缺，帮助我们大家更好地理解其中的知识脉络，为以后单独开发相关的系统提供思路支撑。



### XDP (eXpress Data Path) 简介

XDP 是 Linux 内核上游（Linux 内核原始版本非 Linux 分发版）的一部分，它为用户提供了将用户编写的包处理程序安装进入内核的通道，安装进入内核的包处理程序会在系统接收到包（还未对数据包进行任何处理）的时候触发执行，从而提供了一种高性能的方式允许用户自定义处理内核接收到数据包时的行为。


{{< image src="https://cdn.jsdelivr.net/gh/wgblikeW/blog-imgs/XDP_integration_with_linux_network_stack.png" caption="XDP 与内核协议栈的整合" src_s="https://cdn.jsdelivr.net/gh/wgblikeW/blog-imgs/XDP_integration_with_linux_network_stack.png" src_l="https://cdn.jsdelivr.net/gh/wgblikeW/blog-imgs/XDP_integration_with_linux_network_stack.png" >}}




### XDP 程序操作模式种类

原生模式  (Native XDP) 卸载模式 (Offloaded XDP) 通用模式 (Generic XDP)

> Native XDP：
>
> ​	该模式是默认模式。在这种模式下，XDP 的 BPF 程序 在网络驱动程序的早期接收路径之外直接运行。使用该模式需要硬件设备的支持。使用 `git grep -l XDP_SETUP_PROG drivers` 命令可以检查驱动程序是否支持此模式。
>
>
> Offloaded XDP：
>
> ​	在卸载模式下，XDP 的 BPF 程序直接卸载到网卡上，而不是在主机 CPU 上执行。因本就仅有相当低开销的 XDP 程序的执行从 CPU 上转移到网卡上，从而这种模式能够比原生 XDP 具有更高的性能。使用这种操作模式的 XDP 程序需要得到网卡的支持。使用 `git grep -l XDP_SETUP_PROG_HW drivers` 可以检查哪些网卡驱动程序支持 Offloaded 模式。
>
> Generic XDP：
>
> ​	对于没有提供 Offloaded 或 Native 模式支持的网卡设备来说，内核提供了一个使用 XDP 通用模式的选项。使用此模式不需要需要任何的驱动支持，因为其运行在一个相对于网络内核栈中相对靠后的位置。其存在主要的目的是为了开发人员在不支持上述两种模式的环境中进行 XDP 程序的开发与调试。在该模式下，XDP 程序的运行性能没有前两者那么高效，因此在生产环境中 



### XDP 程序使用场景

- 缓解 DDoS 攻击，防火墙

  利用 XDP BPF 的特性我们可以自行定义驱动丢弃恶意网络包的行为逻辑。通过在对数据包进行处理的早期阶段令数据包处理器向网络驱动抛出 `XDP_DROP` 丢弃恶意数据包，我们能够仅付出极小代价的情况下将数据包丢弃，维持系统资源处于一个健康可用的状态。另外，利用 `XDP_TX` `XDP_PASS` `XDP_REDIRECT` 我们还能够自定义逻辑对流量进行清洗、控制流量的走向，实现对主机的保护。

  借助 XDP，我们可以在网卡或其驱动程序中以完全可编程的方式获得相同的功能，而这相比于昂贵的防火墙硬件设备来的非常便宜且快速。**<u>并且我们可以通过远程调用 API 更改规则来控制映射，然后将映射中的规则集动态传递给每台特定计算机中加载的 XDP 程序，这样就能够动态的控制集群中网络拓扑结构</u>**。

- 负载均衡

  上面提到过数据包处理器可以通过向网络驱动抛出状态码影响网络驱动的对数据包的行为，在负载均衡的场景中，我们可以使用 `XDP_TX` 或者 `XDP_REDIRECT` ，将经过 XDP 程序处理后的数据包发往接收到该数据包的网卡或将其推至另外的网卡中传输。

  > 直接将该数据包传入到一类特殊的套接字家族 (AF_XDP) 中

- 先于内核协议栈的包过滤/处理

  在 Linux 内核协议栈正式处理抵达的数据包之前，XDP 程序可以根据用户自定义的策略对到达的数据包进行过滤，这可以通过 `XDP_DROP` 指示网络驱动丢弃掉不符合规则的数据包实现。

  > 例如，当我们确定某一节点上运行的服务流量只有 TCP 流量时，我们可以使用 XDP 程序丢弃所有使用其它四层协议（UDP、SCTP）的流量。

  我们也可以使用 XDP 程序去实现自有协议的封装与解封装，这对内核协议栈是透明的。不仅如此，我们还能够在接收到的数据包头部（非数据包区域）前增加一段元数据，这些元数据对于内核协议栈来说是透明的，但对于 TC 程序来说它可以获取到相关的元数据信息，从而控制其程序的行为逻辑，完成相关的数据包解析工作。

- 监控

  使用 XDP 程序对到达的网络数据流进行特征统计与流量分析，也可以对数据包进行一些复杂的分析（比如恶意特征提取与检测）。XDP 与以往的一些技术解决方案相比具有以下优势：1）XDP 允许在较早的阶段对网络数据包进行介入，能够截断或是将数据包中的载荷提取出来并通过 Linux 内核提供的 perf 基础设施（快速、无锁、每一个 CPU 都具有的内存环形缓冲区域）推送至用户空间的应用程序中进行下一步处理，这种数据通路的处理性能比以往的解决方案要更加动态快速。

  在云原生安全中，我们需要强调基础设施的可观测性，对于节点、容器、应用、服务等各层次等系统构成元素来说，我们需要监控其运行状态获取其性能指标，也需要对一个进入集群的请求进行追踪，对各组件产生的日记信息进行收集汇总，汇总各个维度的数据信息，为编排平台进行决策以及事件溯源提供充足的信息支撑。XDP 程序可以与其它组件以及数据源进行搭配组合，通过实时的监测指标以及运行状态数据统计，动态地改变集群内部的网络拓扑结构，最终实现服务集群的高可用。


### XDP Hook

使用 LLVM 对内核态的 XDP 程序进行编译后，我们可以使用两种方式将 ELF 文件中的 BPF 字节码加载进入内核并挂载至指定的网络设备当中。

- 编写 C 用户态程序，使用 BPF Loader 将编译后的 XDP ELF 文件加载进入内核之中

- 通过 `iproute2 ip` 挂载 （该方式所使用的 BPF Loader 并不是通过 Libbpf 库实现的，这意味着当我们开始使用 BPF Maps 的时候可能会出现不兼容的情况），下面将给出一段示例展示如何使用 `ip` 工具将编译后的 ELF 文件加载进入内核之中

  ```bash
  # 向 lo 网络设备上以 xdpgeneric 方式挂载 xdp_pass_kern.o ELF 文件中 section 为 xdp 的字节码
  ip link set dev lo xdpgeneric obj xdp_pass_kern.o sec xdp
  ```

  有了挂载的方式我们还需要有查看和卸载 XDP 程序的方式，下面将介绍使用该工具进行查看和卸载的方式，

  ```bash
  # 展示 lo 网络设备的相关信息
  ip link show dev lo
  # 方式 2
  bpftool net list dev lo
  
  # 从 lo 网络设备中卸载以 xdpgeneric 方式挂载的 XDP 程序
  ip link set dev lo xdpgeneric off
  ```

- Go 语言使用 libbpfgo 库加载 ELF 文件中的 XDP 程序字节码，这个方式是本人认为最优雅的方式

  ```go
  // 加载 eBPF 程序编译后的 ELF 文件 	
  bpfModule, err := bpf.NewModuleFromFile("main.bpf.o")
  	if err != nil {
  		fmt.Fprintln(os.Stderr, err)
  		os.Exit(-1)
  	}
  // 加载其中的 eBPF 对象
  err = bpfModule.BPFLoadObject()
  	if err != nil {
  		fmt.Fprintln(os.Stderr, err)
  		os.Exit(-1)
  	}
  // 提取其中的内核态程序 "target" 是自编写的 XDP 程序函数名
  xdpProg, err := bpfModule.GetProgram("target")
  	if xdpProg == nil {
  		fmt.Fprintln(os.Stderr, err)
  		os.Exit(-1)
  	}
  // 指定需要挂载的网络设备 lo
  _, err = xdpProg.AttachXDP("lo")
  	if err != nil {
  		fmt.Fprintln(os.Stderr, err)
  		os.Exit(-1)
  	}
  ```

在上述 Go 程序代码我们可以发现 `bpfModule.GetProgram("target")` 指定了我们想要从 ELF 文件中加载的 eBPF 程序对象，这意味着我们可以在一个 ELF 文件可以包含多个 XDP 程序，我们可以从中选择我们所需要的进行加载。这种能力是 libbpfgo 封装 libbpf 得来的，而 libbpf 是通过封装系统调用得到的。

在 libbpf 中内部使用 `struct bpf_object` `struct bpf_program` `struct bpf_map` 三类结构体去管理和抽象 eBPF 程序，用户需要使用 libbpf 对外暴露的 API 接口去操作相关的数据结构。



> XDP 程序操作模式种类：原生模式  (Native XDP) 卸载模式 (Offloaded XDP) 通用模式 (Generic XDP)
>
> Native XDP：
>
> ​	该模式是默认模式。在这种模式下，XDP 的 BPF 程序 在网络驱动程序的早期接收路径之外直接运行。使用该模式需要硬件设备的支持。使用 `git grep -l XDP_SETUP_PROG drivers` 命令可以检查驱动程序是否支持此模式。
>
> Offloaded XDP：
>
> ​	在卸载模式下，XDP 的 BPF 程序直接卸载到网卡上，而不是在主机 CPU 上执行。因本就仅有相当低开销的 XDP 程序的执行从 CPU 上转移到网卡上，从而这种模式能够比原生 XDP 具有更高的性能。使用这种操作模式的 XDP 程序需要得到网卡的支持。使用 `git grep -l XDP_SETUP_PROG_HW drivers` 可以检查哪些网卡驱动程序支持 Offloaded 模式。
>
> Generic XDP：
>
> ​	对于没有提供 Offloaded 或 Native 模式支持的网卡设备来说，内核提供了一个使用 XDP 通用模式的选项。使用此模式不需要需要任何的驱动支持，因为其运行在一个相对于网络内核栈中相对靠后的位置。其存在主要的目的是为了开发人员在不支持上述两种模式的环境中进行 XDP 程序的开发与调试。在该模式下，XDP 程序的运行性能没有前两者那么高效，因此在生产环境中推荐使用前两种模式。



### XDP 程序的编写

#### Basic 03 - counting with BPF maps

我们可以通过定义一个全局结构体 `bpf_map_def` 并带上 `SEC("maps")` 宏来创建一个 BPF map，

```c
struct {
	__uint(type, BPF_MAP_TYPE_ARRAY);
	__type(key, __u32);
	__type(value, struct datarec);
	__uint(max_entries, XDP_ACTION_MAX);
} xdp_stats_map SEC(".maps");

struct {
    __uint(type, BPF_MAP_TYPE_PERCPU_ARRAY);
    __type(key, __u32);
    __type(value, struct datarec);
    __uint(max_entries, MAX_ENTRIES);
} xdp_stats_map_percpu SEC(".maps");
```

BPF maps 是通用的键值对存储方式，在上述定义中 `__uint(type, BPF_MAP_TYPE_ARRAY)` 用于指定特定类型的 BPF maps（这里给出了 v5.4 内核版本中所有的 [BPF maps 类型](https://elixir.bootlin.com/linux/v5.4/source/include/uapi/linux/bpf.h#L115)），`__type(key, __u32)` 用于指定 Key 对应的数据类型，`__type(value, struct datarec)` 用于指定 Value 对应的数据类型，`__uint(max_entries, XDP_ACTION_MAX)` 用于指定可存放的最大元素个数。

使用 `bpf_object_find_map_by_name()` 函数可以通过 BPF maps 的名字找到其对应的 `bpf_map` 对象，通过 `bpf_map_fd()` 函数可以获得 map 的文件描述符，libbpf 库中提供了一个函数`bpf_object__find_map_fd_by_name()` 用于直接完成上述两个步骤。

{{< admonition type=tip title="eBPF Maps 特性" open=false >}}
所有处于内核态的 BPF 程序以及用户空间中的应用程序都能够访问 BPF Maps。
{{< /admonition >}}

在用户空间中，我们可以通过函数 `bpf_map_lookup_elem()` 在用户空间中读取 BPF Maps 中的内容。



在上述代码中，我们分别定义了两种不同类型的 BPF Maps，它们分别是 `BPF_MAP_TYPE_ARRAY` 与 `BPF_MAP_TYPE_PERCPU_ ARRAY` ，它们两者之间最大的区别【对持有数据的对象进行操作是否需要加锁】，对于前者来说多个数据操作主体共享一片内存空间，所以在这片内存空间之上进行操作需要持有锁（操作需要是原子的），而对于后者来说，由于各数据操作主体独有一片自用的内存空间，便不存在了临界区，我们可以直接在这片区域上进行读写。**不过，这时候我们在用户空间代码中的操作可能会有些许不同，这是由于数据被各 CPU 所持有，要想获取完整的数据需要遍历所有 CPU 持有的 Map 对象**。


{{< admonition tip "eBPF 程序编写指南" false >}}
在编写 BPF 程序时，遇到不熟悉的函数或者用法，可以查看 bpf-helpers。通过 `man bpf-helpers` 命令可以查看各函数相关入参、使用方法描述以及函数的返回值。
{{< /admonition >}}




我在本课程中使用 Go 语言编写对应的用户态代码，主要使用到了 `github.com/aquasecurity/libbpfgo` [这个库](https://github.com/aquasecurity/libbpfgo)，其中遇到了一些坑（也许是我不会用 😅）

我们可以通过 `bpfModule.GetMap("xdp_stats_map_percpu")` 获取 BPF Maps 对象，此后便可以将其卸载到指定的 NIC 上，监听该网卡上接收到的数据包。接着使用 ` bpfMap.GetValu e(unsafe.Pointer(&XDP_PASS))` 我们便可以获取 Map 对应 Key 所存储的数据。但我尝试从 `BPF_MAP_TYPE_PERCPU_ARRAY` 类型的 Map 中获取数据时却没有得到预期中的结果。我随即查阅了 `(*BPFMap) GetValue(unsafe.Pointer)` 函数的相关实现，

```go
// GetValue takes a pointer to the key which is stored in the map.
// It returns the associated value as a slice of bytes.
// All basic types, and structs are supported as keys.
//
// NOTE: Slices and arrays are also supported but special care
// should be taken as to take a reference to the first element
// in the slice or array instead of the slice/array itself, as to
// avoid undefined behavior.
func (b *BPFMap) GetValue(key unsafe.Pointer) ([]byte, error) {
	value := make([]byte, b.ValueSize())
	valuePtr := unsafe.Pointer(&value[0])

	ret, errC := C.bpf_map_lookup_elem(b.fd, key, valuePtr)
	if ret != 0 {
		return nil, fmt.Errorf("failed to lookup value %v in map %s: %w", key, b.name, errC)
	}
	return value, nil
}
```

可以看到其中 value 分配的内存空间只有 `b.ValueSize()` 也即单个 Value 数据结构的空间，但当我们使用 `BPF_MAP_TYPE_PERCPU_ARRAY` Map 类型时，`bpf_map_lookup_elem` 返回的是所有 CPU 各自持有的 Map 对象中的数据，

```c
	struct datarec values[nr_cpus]; // sizeof(struct datarec) * nr_cpus
	int i;

	if ((bpf_map_lookup_elem(fd, &key, values)) != 0) {
		fprintf(stderr,
			"ERR: bpf_map_lookup_elem failed key:0x%X\n", key);
		return;
	}

	for (i = 0; i < nr_cpus; i++) {
		sum_pkts  += values[i].rx_packets;
		sum_bytes += values[i].rx_bytes;
	}
```

`bpf_map_*` 系列函数都是通过 `bpf()` 系统调用实现的，具体实现[参考此处](https://elixir.bootlin.com/linux/v5.4/source/tools/lib/bpf/bpf.c#L371)，也可以参考 `bpf` 系统调用手册。

<center>    <img style="border-radius: 0.3125em;    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08); zoom:67%;"     src="https://cdn.jsdelivr.net/gh/wgblikeW/blog-imgs/bpf_basic03.png">    <br>    <div style="color:orange; border-bottom: 1px solid #d9d9d9;    display: inline-block;    color: #999;    padding: 2px;">示例代码运行输出</div> </center>

#### packet01-parsing



### TCP 协议报文结构

```bash
                    0                   1                   2                   3
                    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
                   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                   |          Source Port          |       Destination Port        |
                   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                   |                        Sequence Number                        |
                   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                   |                    Acknowledgment Number                      |
                   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                   |  Data |           |U|A|P|R|S|F|                               |
                   | Offset| Reserved  |R|C|S|S|Y|I|            Window             |
                   |       |           |G|K|H|T|N|N|                               |
                   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                   |           Checksum            |         Urgent Pointer        |
                   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                   |                    Options                    |    Padding    |
                   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                   |                             data                              |
                   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

                                            TCP Header Format
```

相关字段含义网络上已经有非常多详细的说明，在本文中便不再赘述。



### IP 协议报文结构

```bash
                    0                   1                   2                   3
                    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
                   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                   |Version|  IHL  |Type of Service|          Total Length         |
                   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                   |         Identification        |Flags|      Fragment Offset    |
                   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                   |  Time to Live |    Protocol   |         Header Checksum       |
                   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                   |                       Source Address                          |
                   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                   |                    Destination Address                        |
                   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                   |                    Options                    |    Padding    |
                   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

                                    Example Internet Datagram Header
```

相关字段含义网络上已经有非常多详细的说明，在本文中便不再赘述。

### References

1. [Github repo](https://github.com/xdp-project/xdp-tutorial) for Learning XDP Programming 
2. [Cilium BPF reference guide](https://cilium.readthedocs.io/en/latest/bpf/) for building industrial application with BPF
3. General Introduction to XDP in [the academic paper](https://github.com/xdp-project/xdp-paper/blob/master/xdp-the-express-data-path.pdf) or [the presentation](https://github.com/xdp-project/xdp-paper/blob/master/xdp-presentation.pdf).  
4. [Linux 内核观测技术 BPF](https://item.jd.com/12939760.html)
5. 极客时间课程，倪鹏飞老师的 eBPF 核心技术与实战
