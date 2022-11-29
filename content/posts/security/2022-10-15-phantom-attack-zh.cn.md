---
title: "Phantom Attack: Evading System Call Monitoring (TOCTOU Problem)"
subtitle:   "DEFCON29 分享学习 TOCTOU 绕过 Falco Policy Enforcement"
date:       2022-10-14 11:46:09
author:     "p1nant0m"
lightgallery: true
tags:
    - eBPF
    - TOCTOU
    - Clound Native Security
    - Malicious Behaviour Monitoring
    - DEFCON 29
    - Conference Sharing
---

### 背景知识

在第 29 届的 **DEFCON** 会议上，来自 Lacework 的 Rex Guo 与 LinkedIn 的 Junyuan Zeng 分享了以他们关于利用内核从用户态拷贝数据到内核态的过程中，由于某些 Runtime Security 产品选择了不合适的 Probes 插入位置，存在 TOCTOU 问题，进而能够欺骗探针，绕过 Runtime Security 产品的 Policy Enforcement，进而达成攻击者进行恶意活动的目的。

另外，分享者还介绍了 **Semantic confusion**，一种利用内核与跟踪程序对数据语义解释的歧义实现的攻击方式，这与在 Usenix Security'22 会议上获得最佳论文奖的 [Identity Confusion in WebView-based Mobile App-in-app Ecosystems](https://www.usenix.org/conference/usenixsecurity22/presentation/zhang-lei) 有异曲同工之妙，有兴趣的读者可以自行查阅欣赏。

在这次分享中，他们提到了两个具有漏洞的开源项目，一个是云原生 Runtime Security 项目 [Falco](https://falco.org/)，它同时也是 CNCF 的孵化项目，另一个则是我们之前有介绍过的 [Tracee](https://aquasecurity.github.io/tracee/latest)。

这两个项目实现都不同程度的受到 TOCTOU 问题的影响，存在漏洞被利用的风险。在介绍具体的漏洞利用细节前，我们先来看看什么是 TOCTOU，以及这两个项目最主要的功能实现。

#### Time-Of-Check Time-Of-Use (TOCTOU)

在[维基百科](https://en.wikipedia.org/wiki/Time-of-check_to_time-of-use#Examples)中对 TOCTOU 问题的定义如下，

{{< admonition type=quote title="TOCTOU" open=false >}}
In software development, time-of-check to time-of-use (TOCTOU, TOCTTOU or TOC/TOU) is a class of software bugs caused by a **race condition** involving the checking of the state of a part of a system (such as a security credential) and the use of the results of that check.
{{< /admonition >}}

简单来说 TOCTOU 问题利用了当前 Multi-threads 或 Multi-cores 机器上指令执行序不确定的现象外加不正确的编码所引起的 Race condition，从而当对软件系统的某一部分的状态进行检查并通过后，操作系统可能并不会马上让当前进程继续执行后续的指令流程（使用经过检查的对象），而是调度其它的任务去干一些其它的事情（当然，这种情况也可能不发生，原先的任务可以一直执行下去，直至分配给其的时间片被消耗殆尽）。当操作系统调度我们具有 TOCTOU 问题的进程在 CPU 上运行时，先前通过检查的对象已经被悄悄的修改。从全局上看，安全上下文语义已经发生了变化，操作系统拿着这个被“掉包”的对象进行后续的操作，从而可能会引入一定的安全风险。

我们可以发现在对系统状态进行检查与使用该经过检查的对象之间存在一定的**时间窗口**，这个时间窗口的长度决定了我们进行 TOCTOU 攻击时成功的概率。我们需要有办法让当前进程在执行完检查系统状态的指令后让出 CPU （时间窗口的左端点），并且引入另外一个线程对相关的对象进行修改，最终在回到原指令序列时以被替换后的对象执行后续的任务（时间窗口的右端点）。

{{< image src="https://image.p1nant0m.com/202210151601004.png" caption="TOCTOU Overview" src_s="https://image.p1nant0m.com/202210151601004.png" src_l="https://image.p1nant0m.com/202210151601004.png" >}}

到这里，我们可以发现要想成功利用一次 TOCTOU， 需要集齐以下要素，

- 需要有办法让执行当前任务的进程挂起
- 需要有一个协助线程能够访问目标对象的内存空间，改变目标对象的状态
- 需要让 TOCTOU 窗口足够大以确保我们能够在进程恢复前完成修改目标对象的任务（进程调度是操作系统的任务，用户一般无法干预/预测一个进程在什么时间点执行，我们能够知道的是这个进程在就绪状态下会被执行，至于具体的细节由操作系统决定，这也导致 TOCTOU 的利用不一定是 100% 成功的，当然在有利条件下也是能够达成 100% 成功）

#### Runtime Security Products

Falco 和 Tracee 一样，都可以归属与 Runtime Security 产品类别，主要实现的功能为：1）在运行时环境中、内核层面解析 Linux 系统调用，将其转换为具有安全上下文语义的结构化数据；2）使用规则检测引擎根据用户自定义的规则在捕获器产生的事件流中发现违背规则的事件，并产生告警；

Falco 和 Tracee 实现上述功能的原理都是通过在关键内核函数或跟踪点上 Hook 住探针来达到对进程执行在操作系统内核层面行为的监测，其需要与内核相关函数以及相关事件执行流程进行关联来确定安全上下文语义，进而对进程执行流程进行画像，从而找到异常的行为事件，确保运行时安全。

有关 Tracee 的介绍可以查看本博客的[这篇文章](https://p1nant0m.com/2022-09-05-tracee-%E6%BC%AB%E6%B8%B8%E6%8C%87%E5%8C%97/)。

{{< admonition type=tip title="差异" open=false >}}
Runtime Security 产品的差异可以在几个维度上给予考虑：1）用户自定义规则上的语义丰富程度；2）规则检测引擎的性能；3）可提供安全事件的覆盖范围以及安全上下文语义的丰富程度；4）与其它安全产品或平台的联动支持程度；

需要注意的是，他们都只是对相关的异常行为进行告警，并不会有什么主动的动作，也就是说上述两者并没有阻断进程异常行为的能力，在本篇文章的末尾我将会简要探讨一下 eBPF LSM 在弥补这一方面不足的能力。
{{< /admonition >}}



### 漏洞描述

该漏洞被编号为 [CVE-2021-33505
](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-33505)，官方对其描述如下所示，

{{< admonition type=note title="Description" open=false >}}
A local malicious user can circumvent the Falco detection engine through 0.28.1 by running a program that alters arguments of system calls being executed.
{{< /admonition >}}

该漏洞可以欺骗 Falco 规则引擎的检测（实际上，在数据采集阶段获取的信息在全局的视角来看已经是不准确的了，内核执行的相关操作已与采集器捕获到的数据无关，也就不能表征进程的行为，给出有用的安全建议了），执行规则本不允许的操作而不产生告警信息。

#### TOCTOU 在内核观测中

在两位研究者的分享中，以 `openat` 系统调用为例，介绍了在有内核探针的环境下 TOCTOU 的利用。

{{< image src="https://image.p1nant0m.com/202210151918144.png" caption="openat 系统调用入口" src_s="https://image.p1nant0m.com/202210151918144.png" src_l="https://image.p1nant0m.com/202210151918144.png" >}}

当应用程序需要调用 `openat` 系统调用完成其工作时，将会陷入内核完成相应的逻辑处理流程，首先便是 `trace_sys_enter` 入口，内核探针可以 Attach 到 `raw_syscalls/sys_enter` 处，每当操作系统陷入内核进行系统调用时都会触发相应的探针程序，完成相应的用户处理逻辑。在本文中，Runtime Security 可以在此获得相关系统调用的入参以及进程上下文环境。

接着，根据具体执行的系统调用和硬件平台查找 [Syscall Table](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl)（在此处是 `openat`）调用相应的内核处理函数（在此处是 `do_sys_open`）完成相应的任务，最后调用 `trace_sys_exit` 函数结束系统调用，最后返回应用程序空间。若有任何内核探针被挂载到 `trace_sys_exit` 上，其也会在 `trace_sys_exit` 函数被调用时执行。

利用 eBPF 编写 Tracing 类程序可以简单参考本博客中的[这篇文章](http://localhost:1313/2022-07-25-ebpf-tracing/)。

{{< image src="https://image.p1nant0m.com/202210152022831.png" caption="在不同位置引入 Probe" src_s="https://image.p1nant0m.com/202210152022831.png" src_l="https://image.p1nant0m.com/202210152022831.png" >}}

上图考虑的是 Tracing Program 在不同的内核位置 Hook 探针，主要考虑在（1）`trace_sys_enter` raw_tracepoints 入口处，以及 （2）在 `do_sys_open` 执行具体 `openat` 系统调用处。

当内核执行到这些被 Hook 住的函数时，会触发用户挂载的自定义处理程序（比如 eBPF 程序），执行相应的处理逻辑。比如，在 Tracce 中定义了两种 Probes，

```go
SysEnter: &traceProbe{
            eventName: "raw_syscalls:sys_enter", 
            probeType: rawTracepoint, 
            programName: "trace_sys_enter"},

SyscallEnter__Internal: &traceProbe{
            eventName: "raw_syscalls:sys_enter", 
            probeType: rawTracepoint, 
            programName: "tracepoint__raw_syscalls__sys_enter"},
```

它们对应的内核态 eBPF 程序如下所示，

```go
SEC("raw_tracepoint/trace_sys_enter")
int trace_sys_enter(struct bpf_raw_tracepoint_args *ctx)
{
    event_data_t data = {};
    if (!init_event_data(&data, ctx))
        return 0;
    if (!should_trace(&data))
        return 0;

    // always submit since this won't be attached otherwise
    int id = ctx->args[1];
    struct task_struct *task = (struct task_struct *) bpf_get_current_task();
    if (is_compat(task)) {
        // Translate 32bit syscalls to 64bit syscalls, so we can send to the correct handler
        u32 *id_64 = bpf_map_lookup_elem(&sys_32_to_64_map, &id);
        if (id_64 == 0)
            return 0;

        id = *id_64;
    }
    save_to_submit_buf(&data, (void *) &id, sizeof(int), 0);
    events_perf_submit(&data, RAW_SYS_ENTER, 0);
    return 0;
}

SEC("raw_tracepoint/sys_enter")
int tracepoint__raw_syscalls__sys_enter(struct bpf_raw_tracepoint_args *ctx)
{
    struct task_struct *task = (struct task_struct *) bpf_get_current_task();
    int id = ctx->args[1];
    if (is_compat(task)) {
        // Translate 32bit syscalls to 64bit syscalls, so we can send to the correct handler
        u32 *id_64 = bpf_map_lookup_elem(&sys_32_to_64_map, &id);
        if (id_64 == 0)
            return 0;

        id = *id_64;
    }
    bpf_tail_call(ctx, &sys_enter_init_tail, id);
    return 0;
}
```

不过，`trace_sys_enter` 在通常情况下并不会默认挂载到指定的 tracepoint 上，只有当显式指定 **Tracing Arguments** 中包含 `sys_enter` 事件时才会被挂载。在 Tracee 中使用 `tracepoint__raw_syscalls__sys_enter` 跟踪系统调用，并通过 tail call 完成事件的解析与上报。

回到上图跟踪 `openat` 系统调用的例子，为了更好的说明，我们需要知道 `openat` 系统调用的函数签名，
```c
int openat(int dirfd, const char *pathname, int flags, mode_t mode);
```
其中，第二个参数是一个指向用户空间内存的指针，其中存放的是进行系统调用时，应用传入的想要打开文件的路径。在 `do_sys_open` 函数中，内核通过 `getname` 函数将用户空间的数据复制到内核缓冲区中，并在后续调用 `do_filp_open` 传入相关待打开文件的路径参数，完成 `openat` 系统调用的主要逻辑。

我们可以以 `getname` 函数为界，将该代码段分为 CP-1 和 CP-2 两部分。在 CP-1 中，`filename` 数据存放于用户空间内存，位于用户空间的攻击者可以修改存放 `filename` 数据的内存，进而在内核拷贝/使用相关数据时（TOU）传入的是攻击者篡改的恶意路径。

我们若考虑在 CP-1 函数中 Hook 探针有这么两个选择：1）使用 **raw_tracepoints** 在 `trace_sys_enter` 上 Hook 探针，2）使用 **kprobe** 在 `do_sys_open` 上 Hook 探针；由于这两个函数的入参指向的都是位于用户空间的内存，攻击者能够在通过探针的检查后，在程序控制流执行到 `getname` 函数前，修改相关内存位置的 `filename` 数据，欺骗 **Tracing Program**，达到进行恶意行为，避免产生安全告警的目的。这两个 Hook Points 也就是我们所说的 TOC 部分，其中探针获取到的数据在全局来看是不可靠的。

在 CP-2 中，由于位于用户空间侧的数据已经被拷贝进入了内核缓冲区，其能够避免位于用户空间侧攻击者对数据的篡改。若我们此时选择 `do_filp_open` 函数作为 **Tracing Program** 的 Hook 点，我们就能够确保内核执行文件打开逻辑所使用的参数与我观测程序所“看到”的参数是一致的。（我们在这做的就是将观测点尽可能的靠近内核进行主要操作逻辑的部分）

需要注意的是，若 Hook 点选择在 `trace_sys_exit` 上，其仍然会受到 TOCTOU 问题的影响，因为其传入的参数与 `trace_sys_enter` 中的参数是一致的。

#### Phantom v1 Exploit Plan

研究者给出了他们利用 TOCTOU 问题对 Falco 进行 Exploit 的计划，具体可以划分为以下几个环节，

- 使用恶意参数触发目标系统调用
- 让内核从用户空间中读取恶意参数并且执行攻击者预想的恶意行为
- 使用合法的参数覆写用户内存空间指针指向的数据结构
- 在调用 `sys_exit` 准备结束一次系统调用的时候，**tracing program** 读取用户空间指针指向的数据结构并且通过规则引擎的检测

为了实现上述这些目标需要面临如下挑战，

- 内核线程什么时候会读取用户传入的参数？
- 我们如何同步覆写与内核线程读行为？（确保覆写完成后，内核线程读取到的是我们覆写的合法内容）
- 竞态窗口对于每一个系统调用来说是否足够大？
- 如何确保 **tracing program** 读取到的是我们覆写的内容？

##### Userfaultfd Syscall

研究者介绍了一个在 TOCTOU 利用中常见的 Trick —— `Userfaultfd` 系统调用，它允许用户注册自己的缺页处理机制，在发生缺页错误时若通过 `userfaultfd` 注册了相应的处理函数，将会导致内核线程暂停执行，等待用户空间的用户处理代码相应，更多细节可以[参考这篇 `userfaultfd` 系统在 CTF 比赛中的应用](https://ctf-wiki.org/pwn/linux/kernel-mode/exploitation/userfaultfd/)。

##### Interrupts and Scheduling

在 Exploit 的背景介绍中研究者还铺垫了有关中断以及进程调度的相关内容，在这里我就直接将原文贴出了，

- An interrupt notifies the processor with an event that requires immediate attention
- An interrupt diverts the program control flow to an interrupt handler
- Interrupt can be triggered indirectly from system calls
  - Hardware interrupts (networking, e.g., connect)
  - Interprocessor interrupts (IPIs) (e.g., mprotect)

需要注意的是 `mprotect` 场景下的中断处理，由于 `mprotect` 会改变映射内存页的权限，所有缓存了该页的处理器缓存都会发生失效，需要重新加载并缓存具有正确权限的内存页面。下图展示了研究者在 Exploit 中会使用到的攻击模型简易示例，

{{< image src="https://image.p1nant0m.com/202210161424770.png" caption="攻击模型简要示例" src_s="https://image.p1nant0m.com/202210161424770.png" src_l="https://image.p1nant0m.com/202210161424770.png" >}}

可以看到在 Core0 上运行着用户程序 TaskA，Core1 上运行着用于触发中断的辅助程序，当 TaskA 触发系统调用陷入内核处理时，Core1 可以通过向 Core0 发送硬中断或 IPI 中断，打断 Core0 上的内核线程执行，使其转向执行注册的中断处理程序。在执行完中断处理程序后，再恢复当前进程上下文，继续执行原先的内核线程系统调用。

{{< admonition type=note title="Core1 中断 Core0" open=false >}}
关于 Core1 触发的中断为何要让 Core0 进行处理，在 HW interrupt 的情况中，有专门的 CPU 核心用于处理此类中断类型；在 IPI interrupts 中，比如使用 `mprotect` 系统调用触发中断会导致内存页面权限发生变化，使用该内存页面的 CPU 需要更新缓存以获得具有正确权限的内存页，这将会导致其进行中断处理。
{{< /admonition >}}

进一步，研究者介绍了他们在本次 Exploit 中将会使用到的基本攻击原语: **sched_setscheduler()** 以及 **sched_setaffinity()**.

**sched_setscheduler()** 在 Exploit 中是一个可选项，当 TOCTOU Window 足够大时，比如进行与网络相关的系统调用可以不需要使用这个方法。但在向处理打开文件这样的系统调用需要使用该方法来改变进程运行的优先级来增加 TOCTOU Window 的大小，以确保 Exploit 能够以一个较高的概率成功。有关 **sched_setscheduler** 更进一步的介绍可以[点击这里](https://man7.org/linux/man-pages/man2/sched_setscheduler.2.html)，感兴趣的读者可以自行查阅。

**sched_setaffinity()** 用于将一个线程固定到一个指定的 CPU 核心上。

##### Phantom v1 Attack - An Openat Examples

整一个攻击流程可以用下图展示，
{{< image src="https://image.p1nant0m.com/202210161447837.png" caption="Phantom v1 Attack" src_s="https://image.p1nant0m.com/202210161447837.png" src_l="https://image.p1nant0m.com/202210161447837.png" >}}

可以看到整一个攻击链由三个主要的线程共同协作完成，它们分别是 **main thread**, **userfaultfd thread** 和 **overwrite thread**。其中 **main thread** 给固定在了 CPU3 上，而 **overwrite thread** 固定到了 CPU2 上。研究者进行 Exploit 演示的硬件平台运行在四核 CPU 环境上， CPU3 正是用于处理 Network Interrupt 的默认 CPU，因此若使用 Networking Interrupt 触发 Core3 执行中断处理程序，那么我们就需要将其 PIN 在 Core3 上，而若使用 IPIs 作为手段，**main thread** 可以运行在任意 CPU 上。


在 **main thread** 上调用 `mmap` 创建 pageA 内存映射，注意到此时该页还未被分配加载。同时，注册 `userfaultfd` 线程去处理 pageA 的缺页中断事件。**main thread** 还会作为调用 `openat` 的入口来演示相关的 TOCTOU 问题。

**overwrite thread** 负责在 **tracing program** 检查前向 pageA 中写入合法的文件名，其在刚启动时会被条件锁给阻塞。

在了解了整个 Exploit 的基本设置后，我们结合流程图在 Exploit 代码的视角来看看整个攻击的流程，

```c
    ...
  page_size = sysconf(_SC_PAGE_SIZE);
  page = mmap(NULL, page_size, PROT_READ | PROT_WRITE,
              MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
    ...
  int myfd = open(page, O_CREAT|O_RDWR|O_DIRECT, 0640);
  if (myfd < 0) 
    handle_error_en(myfd, "open failure");
```

通过调用 `open` 函数触发一次系统调用，此时 pageA 指向的内存位置还未被分配，这将触发一次缺页中断。**main thread** 的控制流由 **kernel thread** 转向由 `userfaultfd` 注册的用于自定义缺页处理函数，我们在这省略一些初始化逻辑以及用于调试的输出信息，
```c
    ...
  /* Create a page that will be copied into the faulting region */
  if (page == NULL) {
    page = mmap(NULL, page_size, PROT_READ | PROT_WRITE,
                MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
    if (page == MAP_FAILED)
      handle_error_en(page, "mmap");
  }
    ...

      uffdio_copy.src = (unsigned long) page;
      /* We need to handle page faults in units of pages(!).
         So, round faulting address down to page boundary */
      uffdio_copy.dst = (unsigned long) msg.arg.pagefault.address &
                                           ~(page_size - 1);
      uffdio_copy.len = page_size;
      uffdio_copy.mode = 0;
      uffdio_copy.copy = 0;

      /* 将 Malicious filename 复制到 page 指向内存空间的起始位置 */
      strncpy(page, filename, sizeof(filename));
      flush(page);
      /* 释放互斥锁 */
      sender(); 

      printf("Before ioctl\n");
      if (ioctl(uffd, UFFDIO_COPY, &uffdio_copy) == -1)
        handle_error_en(-1, "ioctl-UFFDIO_COPY");

      printf("        (uffdio_copy.copy returned %lld)\n",
                uffdio_copy.copy);
    ...  
```

我们在 `userfaultfd` 注册相应事件的处理逻辑，在这里主要就关注 UFFD_EVENT_PAGEFAULT. 此时，我们已经“偷偷地”改写了 `filename`，并释放了用于同步 **overwrite thread** 的锁，最终通过 IOCTL 返回到内核执行流。

内核线程从恶意 `filename` 指向的用户内存空间中复制要打开文件的路径数据到内核缓冲区，并以此为参数调用内部处理文件打开的函数，这是 TOU 阶段。

与此同时，在另外一个 CPU Core 上运行着的 **overwrite thread** 正准备向 pageA 中写入合法的路径数据，
```c
  receiver() /* 该线程会在这暂停执行，直到锁释放 */

  asm volatile("mfence");
  // 写入合法的路径数据
  write_char(fakename, sizeof(fakename));
  asm volatile("mfence");
  flush(page);

    /* trigger interrupt */
#ifdef TLBSHOOTDOWN_MPROTECT
  int s = mprotect((void *)page, 4096, PROT_READ | PROT_WRITE | PROT_EXEC);
  if (s != 0) {
    handle_error_en(s, "mprotect");
  }
  return NULL;
#endif 
```

在上述代码中，我们使用了 IPIs 来触发 CPU Core3 上运行着的 **kernel thread**
的中断，这将会为我们的覆写带来充足的时间窗口。

最终，当控制流又重新回到 **kernel thread** 时，Hook 在 `sys_exit` 处的 **tracing program** 读取到被修改后的 `filename`，最终实现 TOCTOU 问题的利用，欺骗 **Runtime Security** 产品，访问规则不允许的文件而不产生告警，相关的 demo 可以在 github 上找到，[链接](https://github.com/rexguowork/phantom-attack/blob/main/phantom_v1/attack_openat.c#L385-L392)放在这儿。

### 🤔进一步研究

在介绍整个 Phantom Attack v1 的过程中，我们不难发现整个攻击链路的关键点在于设计的 **tracing program** 选择了什么函数作为跟踪的 Hook 点 ，若选择的 Hook 点读取的数据来自于用户空间侧，其就存在被篡改的风险，若选择的 Hook 点使用的是已经位于内核侧的数据作为入参，其存在被篡改的可能性就小，**tracing program** 获得到的数据也就更加可信。在内核中有以 security_xxxx 命名的函数，这些函数所处的路径可以认为是安全的，hook 到这些函数的 kprobe 可以信任它们从中获得的数据。

并且我们发现这些 **Runtime Security** 产品并没有能够对异常行为进行阻断的能力，它们能够做的事情是：发生正在进行的异常行为，并将与该行为相关的上下文信息尽可能多的展示给安全运维人员，用作事后审计或有人员实时对相关异常行为进行研判处理。

为了让运行时安全产品获得阻断异常行为的能力，可供选择的解决方案之中赫然屹立着 **LSM BPF** 这种令人耳目一新的解决方案，后续随着进一步的学习，笔者也会写写这方面有关的学习笔记。

### References
[DEF CON 29 - Rex Guo, Junyuan Zeng - Phantom Attack: Evading System Call Monitoring](https://www.youtube.com/watch?v=yaAdM8pWKG8&t=2030s)
[LSM BPF Change Everything - Leonardo Di Donato, Elastic & KP Singh, Google](https://www.youtube.com/watch?v=l8jZ-8uLdVU)