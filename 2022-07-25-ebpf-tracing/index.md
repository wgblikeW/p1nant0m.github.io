# eBPF-tracing 使用 eBPF 编写监控类程序


![header_](https://image.p1nant0m.xyz/header_opensource.png)

### eBPF 程序的跟踪能力

{{< style "text-align:right; strong{color:#00b1ff;}" >}}
The ability to attach eBPF programs to trace points as well as kernel and user application probe points allows unprecedented visibility into the runtime behavior of applications and the system itself. By giving introspection abilities to both the application and system side, both views can be combined, allowing powerful and unique insights to troubleshoot system performance problems. Advanced statistical data structures allow to extract meaningful visibility data in an efficient manner, without requiring the export of vast amounts of sampling data as typically done by similar systems.
{{< /style >}}

![img](https://ebpf.io/static/intro_tracing-ffa5e3fa3407ecb445b1549f85f590f5.png)



### 如何找到你所需要的追踪点？

首先，我们需要明确自身的需求，考虑清楚我们需要获取哪一些数据信息，再从这些数据信息的类型出发找到其对应的追踪位置。在 eBPF 程序中有以下这几种常用的追踪点类型：

- [kprobe 内核探针](#kprobe)
- [uprobe 用户程序探针](#uprobe)
- tracepoint 内核静态跟踪点
- raw_tracepoint 原始跟踪点

知道了有哪些种类的跟踪点还不够，我们还需要根据自身的需求找到对应类型具体的跟踪点对象。

#### kprobe <a id="kprobe"></a>

对于 krpobe 类型内核探针来说，我们可以使用 `bpftrace` 工具查看支持的所有内核探针，我们只需要在 CLI 终端中键入 `bpftrace -l "kprobe:*"`  便可以查看所有支持的内核探针。

```
kprobe:zswap_free_entry
kprobe:zswap_frontswap_init
kprobe:zswap_frontswap_invalidate_area
kprobe:zswap_frontswap_invalidate_page
kprobe:zswap_frontswap_load
kprobe:zswap_frontswap_store
```

对于每一个内核探针来说，它都将被挂载在对应的内核函数之上，当该内核函数被调用后便会触发我们 Hook 在其之上的 eBPF 处理函数。如 `zswap_free_entry` 函数，在 https://elixir.bootlin.com/ 中，我们可以找到对应版本的 Linux kernel 查看相关内核函数的函数签名和实现，`zswap_free_entry` 函数的实现如下所示，

```c
/*
 * Carries out the common pattern of freeing and entry's zpool allocation,
 * freeing the entry itself, and decrementing the number of stored pages.
 */
static void zswap_free_entry(struct zswap_entry *entry)
{
	if (!entry->length)
		atomic_dec(&zswap_same_filled_pages);
	else {
		zpool_free(entry->pool->zpool, entry->handle);
		zswap_pool_put(entry->pool);
	}
	zswap_entry_cache_free(entry);
	atomic_dec(&zswap_stored_pages);
	zswap_update_total_size();
}
```

我们到这先停止以下，看看我们目前为止都做了哪一些工作：1）确定自身进行系统监控的需求；（2）根据需求找到需要进行跟踪的对象（在这里便是我们具体的 kprobe）；3）了解相关 kprobe 挂载的内核函数签名，**更进一步可以明确函数的调用链，将其与具体的事件相关联**。

{{< admonition tip "想到的话就得说 😀" false >}}
事件是触发一系列操作的因。

比如，我们使用 `cat foo.txt` 命令将 `foo.txt` 文件的内容输出至标准输出流中时就涉及到了操作系统的一系列操作（执行 `/usr/bin/cat` 二进制文件，内存申请与分配，打开文件，读写文件等），期间操作系统执行系统调用，由用户态陷入内核态，进而在内核空间中执行相关的逻辑。利用操作系统的能力为上层应用提供对应的服务，让应用无需关心复杂的资源分配与调度问题（CPU、内存、I/O）。

不过，了解内核函数何时被调用、被谁调用是一个较高的要求，它需要你对操作系统、对内核实现十分熟悉，对于非内核开发者来说无疑十分苦恼。

不过有了 eBPF 程序，我们可以通过函数调用栈找到上层的 caller，进而观察被 Hook 函数调用的全过程，实现对触发事件的溯源。通过在敏感函数上 Hook eBPF 程序，我们可以发现并记录系统中发生的异常行为，从而为事后的分析溯源以及拦截阻断提供充足的决断依据。
{{< /admonition >}}

接下来，我们就需要着手开始 eBPF 程序的编写了，我们先来看看一个 eBPF 程序的基本元素。

- **SEC Macro**: SEC 宏展开后用于创建 ELF Section ，帮助 BPF 加载器找到对应元素的位置，比如像 libbpf 库就可以通过 ELF 文件中相应的 Section 来找到并加载对应的 eBPF 程序。 
- **LICENSE**: eBPF 程序的授权许可证，通常使用 GPL 许可。

对于 kprobe 类型的 eBPF 程序（BPF_PROG_TYPE_KPROBE）来说，通常以 `SEC(kprobe/kernel_function_name)` 方式使用，比如 `SEC(kprobe/zswap_free_entry)` 。在其它类型的 eBPF 程序上也大同小异，通常为 `SEC(bpf_prog_type/instance_name)` ，例如 tracepoint 类型的 eBPF 程序使用 `SEC(tracepoint/tracepoint_name)` 对 eBPF 函数进行声明。

上述工作都做完后，我们就正式进入到 eBPF 程序的编写过程了，在编写 eBPF 程序时我们要确定对于 eBPF 程序类型函数的入参。在 kprobe 类型的 eBPF 函数中，我见到了十分多对传入指针不同的类型解释，我们都一起来看看各项目实现都是如何使用的，

在 [tracee](https://github.com/aquasecurity/tracee) 中使用了 libbpf 库提供的 BPF_KPROBE Macro 去定义它们的 kprobe 类型 eBPF 程序，该宏定义如下所示，

```c
/*
 * BPF_KPROBE serves the same purpose for kprobes as BPF_PROG for
 * tp_btf/fentry/fexit BPF programs. It hides the underlying platform-specific
 * low-level way of getting kprobe input arguments from struct pt_regs, and
 * provides a familiar typed and named function arguments syntax and
 * semantics of accessing kprobe input paremeters.
 *
 * Original struct pt_regs* context is preserved as 'ctx' argument. This might
 * be necessary when using BPF helpers like bpf_perf_event_output().
 */
#define BPF_KPROBE(name, args...)					    \
name(struct pt_regs *ctx);						    \
static __attribute__((always_inline)) typeof(name(0))			    \
____##name(struct pt_regs *ctx, ##args);				    \
typeof(name(0)) name(struct pt_regs *ctx)				    \
{									    \
	_Pragma("GCC diagnostic push")					    \
	_Pragma("GCC diagnostic ignored \"-Wint-conversion\"")		    \
	return ____##name(___bpf_kprobe_args(args));			    \
	_Pragma("GCC diagnostic pop")					    \
}									    \
static __attribute__((always_inline)) typeof(name(0))			    \
____##name(struct pt_regs *ctx, ##args)
```

通过注释以及宏定义我们可以看到，该宏的用途主要是消除了不同平台从 `struct *pt_regs` 中读取数据的差异，具体的入参还是 `struct *pt_regs` ，当然也可以通过 `args` 指定其它的参数。

通过 `struct *pt_regs` 我们能够获得调用被 Hook 函数时寄存器的状态，并能够获取其中的数据。对于不同的硬件平台，使用 `BPF_KPROBE` 宏能为我们抹除平台差异，让程序员能够以最熟悉的方式编写 eBPF 程序，`struct *pt_regs` 的定义如下所示，

```c
struct pt_regs {
	long unsigned int r15;
	long unsigned int r14;
	long unsigned int r13;
	long unsigned int r12;
	long unsigned int bp;
	long unsigned int bx;
	long unsigned int r11;
	long unsigned int r10;
	long unsigned int r9;
	long unsigned int r8;
	long unsigned int ax;
	long unsigned int cx;
	long unsigned int dx;
	long unsigned int si;
	long unsigned int di;
	long unsigned int orig_ax;
	long unsigned int ip;
	long unsigned int cs;
	long unsigned int flags;
	long unsigned int sp;
	long unsigned int ss;
};
```

通过一些辅助宏，我们可以从中获取到被 Hook 函数的入参，在 libbpf 库中便有如下的宏可供我们使用，

```c
#define PT_REGS_PARM1(x) (__PT_REGS_CAST(x)->__PT_PARM1_REG)
#define PT_REGS_PARM2(x) (__PT_REGS_CAST(x)->__PT_PARM2_REG)
#define PT_REGS_PARM3(x) (__PT_REGS_CAST(x)->__PT_PARM3_REG)
#define PT_REGS_PARM4(x) (__PT_REGS_CAST(x)->__PT_PARM4_REG)
#define PT_REGS_PARM5(x) (__PT_REGS_CAST(x)->__PT_PARM5_REG)
#define PT_REGS_RET(x) (__PT_REGS_CAST(x)->__PT_RET_REG)
#define PT_REGS_FP(x) (__PT_REGS_CAST(x)->__PT_FP_REG)
#define PT_REGS_RC(x) (__PT_REGS_CAST(x)->__PT_RC_REG)
#define PT_REGS_SP(x) (__PT_REGS_CAST(x)->__PT_SP_REG)
#define PT_REGS_IP(x) (__PT_REGS_CAST(x)->__PT_IP_REG)
```

根据不同的硬件平台，这些宏获取函数入参的方式也并不相同，在 x86 平台下，它们获取函数入参的方式如下，

```c
#define __PT_PARM1_REG di
#define __PT_PARM2_REG si
#define __PT_PARM3_REG dx
#define __PT_PARM4_REG cx
#define __PT_PARM5_REG r8
#define __PT_RET_REG sp
#define __PT_FP_REG bp
#define __PT_RC_REG ax
#define __PT_SP_REG sp
#define __PT_IP_REG ip
```

{{< admonition info "新知" false >}}

不同的编译语言使用不同的调用惯例 (calling convention)，用于规定调用方与被调用方对函数调用的共识。

这些共识有：函数参数、返回值使用何种方式进行传递以及参数的传递顺序、传递方式如何等。

函数参数既可以放在寄存器中进行传递，也可以放在堆栈中进行传递（Plan 9 ABI），由于 Linux Kernel 使用 C 语言编写，其调用惯例遵循 C 编译器的规则，使用寄存器进行函数调用参数及返回值的传递。

{{< /admonition >}}

使用 `PT_REGS_PARM1(ctx)` 便可以获取到函数的第一个入参，以此类推，我们可以使用其它相应的宏获取所有函数入参。

在这里我们已经知道函数的签名，使用类型转换便可以获取到对应数据类型的函数入参，方便我们进行后续的处理。比如在上述 `zswap_free_entry` 中，使用 `entry = (struct zswap_entry *) PT_REGS_PAMR1(ctx)` 便可以获得第一个函数入参。

> 在编写 eBPF 程序时引入 `vmlinux.h` 头可以一次性导入内核所有使用到的数据结构。
>
> 使用 `bpftool btf dump file /sys/kernel/btf/vmlinux format c > vmlinux.h` 命令可以导出 `vmlinux.h` 文件。

到这里，我们就已经可以使用 BPF Map 向用户侧运行的程序提交内核发生的事件信息，从而监控系统内部的运行状态了。不过，除了被 Hook 函数的入参以外，我们还想要获取其它更多的信息，比如执行函数的上下文环境信息，bpf_helpers 也为我们提供了获取这些信息的途径，常用的函数有，

- bpf_get_current_cgroup_id
- bpf_get_current_ancestor_cgroup_id
- bpf_get_current_comm
- bpf_get_current_pid_tgid
- bpf_get_current_task
- bpf_get_current_task_btf
- bpf_get_current_uid_gid
- bpf_get_ns_current_pid_tgid

这些函数的使用方式以及用途都可以在 bpf-helpers 中查看，例如 `bpf_get_current_task` 的文档，

```
       u64 bpf_get_current_task(void)

              Return A pointer to the current task struct.
```

在 eBPF 程序中，我们通常都需要从某一段内存空间中获取数据，这里的内存空间可能是位于用户态的也可能是位于内核态的，我们想要将这些数据从 eBPF 程序通过 BPF Maps 传递给用户态程序就需要拷贝这些内存空间中的数据，这时候就需要使用到 BPF 中的读系列函数。

通用情况（一般来说我们都推荐使用目标更加明确的函数）：

- bpf_probe_read

在用户态一侧：

- bpf_probe_read_user
- bpf_probe_read_user_str

在内核态一侧：

- bpf_probe_read_kernel
- bpf_probe_read_kernel_str

需要注意后缀带 `_str` 的函数，其用法示例为，

```c
bpf_probe_read_user_str(&event->args[event->args_size], ARGSIZE, argp);
```

这里并不会从 `argp` 指针指向的区域读取 `ARGSIZE` 字节数目的内容，而是读取从 `argp` 开始到字符串结束标识符 `\x00` 为止，也就是说 `\x00` 字符需要包含在最大读取字符长度 `ARGSIZE` 以内。

到这里，我们已经可以编写一个功能十分完善的 eBPF kprobe 程序了，但如果需要实现更为复杂的逻辑，联动其它的 eBPF 程序那就需要 BPF Map 的支持，充分发挥程序员的想象力了。



#### WIP 🐱

#### uprobe <a id="uprobe"></a>



#### tracepoint

在 `/sys/kernel/tracing/event` 路径下能够找到所有可以追踪的内核事件

#### raw_tracepoint

