# 基本概念
## ebpf是什么？
eBPF（扩展伯克利包过滤器）是一种革命性的 Linux 内核技术，允许用户在不修改内核源代码或加载内核模块的情况下，安全、高效地运行自定义程序。
它通过在内核中运行一个受限制的虚拟机（eBPF VM），提供了一种动态注入代码的能力，广泛应用于网络、监控、安全、性能分析，调度器优化，存储加速等领域。
## BPF程序类型和Helper类型
BPF程序类型定义了义了程序可以附加到内核的哪些钩子（hook），并限制其允许的操作，决定了eBPF 能做什么。
helper函数是 eBPF 程序与内核交互的接口，允许安全地访问内核数据或执行受限操作
doc: https://docs.kernel.org/bpf/helpers.html
全部类型参考 https://elixir.bootlin.com/linux/v6.13.7/source/include/uapi/linux/bpf.h
https://elixir.bootlin.com/linux/v6.13.7/source/tools/lib/bpf/bpf_helpers.h
### 程序类型
1. 网络类程序
- BPF_PROG_TYPE_SOCKET_FILTER
用途：过滤套接字接收的数据包（类似 tcpdump）。
钩子点：socket 的 SO_ATTACH_BPF 选项。
- BPF_PROG_TYPE_XDP
用途：在网络驱动层（早于内核协议栈）处理数据包，用于高性能网络过滤、转发或 DDoS 防护。
钩子点：XDP（eXpress Data Path）挂载到网卡驱动。
- BPF_PROG_TYPE_SCHED_CLS
用途：流量分类（Traffic Control 子系统），替代 tc 的 classifier。
钩子点：Linux 流量控制层（sch_clsact）。
- BPF_PROG_TYPE_SCHED_ACT
用途：流量动作执行（如丢弃、重定向数据包）。
钩子点：流量控制层的 action 阶段。
- BPF_PROG_TYPE_CGROUP_SKB
用途：控制 cgroup 内进程的网络流量（允许/拒绝入站或出站数据包）。
钩子点：cgroup 的网络流量入口/出口。
- BPF_PROG_TYPE_SK_LOOKUP
用途：自定义套接字查找逻辑（如负载均衡）。
钩子点：内核执行套接字查找时触发。

2. 跟踪与观测类程序
- BPF_PROG_TYPE_KPROBE
用途：动态跟踪内核函数的调用或返回（通过 kprobe/kretprobe）。
钩子点：内核函数入口或返回地址。
- BPF_PROG_TYPE_TRACEPOINT
用途：静态跟踪预定义的 tracepoint（如系统调用、文件操作）。
钩子点：内核预埋的静态跟踪点（/sys/kernel/tracing/events）。
- BPF_PROG_TYPE_PERF_EVENT
用途：响应硬件或软件性能事件（如 CPU 缓存未命中、定时采样）。
钩子点：Perf 事件（perf_event_open）。
- BPF_PROG_TYPE_RAW_TRACEPOINT
用途：直接访问原始 tracepoint 参数（无参数格式转换，性能更高）。
- BPF_PROG_TYPE_FENTRY/FEXIT
用途：跟踪内核函数入口/退出（替代 kprobe，依赖 BTF 类型信息）。
钩子点：通过 BTF（BPF Type Format）定位函数边界。

3. 安全类程序
- BPF_PROG_TYPE_LSM
用途：实现 Linux 安全模块（LSM）策略（如动态权限检查）。
钩子点：LSM 钩子（如文件访问、进程创建）。
- BPF_PROG_TYPE_CGROUP_DEVICE
用途：控制 cgroup 内进程的设备访问权限。
4. 其他类型
- BPF_PROG_TYPE_SYSCALL
用途：跟踪或过滤系统调用。
- BPF_PROG_TYPE_STRUCT_OPS
用途：动态替换内核子系统（如 TCP 拥塞控制算法）。

### helper 函数
1. 通用助手函数
- bpf_map_lookup_elem / bpf_map_update_elem
操作 eBPF 映射（查询、插入键值对）。
- bpf_trace_printk
调试输出（类似 printk，但功能有限）。
- bpf_get_current_pid_tgid
获取当前进程的 PID 和 TGID。
- bpf_ktime_get_ns
获取系统单调时间（纳秒精度）。

2. 网络相关助手
- bpf_skb_load_bytes / bpf_skb_store_bytes
读取或修改网络数据包内容。
- bpf_xdp_adjust_head
调整 XDP 数据包的头部指针（用于封装/解封装）。
- bpf_redirect_map
将数据包重定向到其他网卡或 CPU（用于负载均衡）。

3. 跟踪与上下文访问
- bpf_probe_read_user / bpf_probe_read_kernel
安全读取用户或内核空间内存。
- bpf_get_current_comm
获取当前进程的进程名。
- bpf_get_stackid
捕获用户或内核栈跟踪。

4. 系统调用与安全
- bpf_send_signal
向进程发送信号（如强制终止恶意进程）。
- bpf_override_return
修改内核函数的返回值（需特权级权限）。

5. 辅助数据结构
- bpf_spin_lock / bpf_spin_unlock
对映射中的数据进行同步（防止竞态条件）。
- bpf_ringbuf_output
向环形缓冲区写入数据（高性能事件传递）。

# devtools for epbf
## bpftrace
![ bpftrace_arch](./bpftrace.png)
## ubprobe
- https://lwn.net/Articles/499190/
- uretprobe https://lwn.net/Articles/543924/
## how USDT works
https://leezhenghui.github.io/linux/2019/03/05/exploring-usdt-on-linux.html#heading-linux-tracing-technical-stack
https://docs.ebpf.io/linux/concepts/usdt/
- In authoring time, Using macro DTRACE_PROBE() to delcare a USDT trace point at appropriate souce code location
- During compilation, the source code with USDT trace point will be translated into a nop instruction, in the meanwhile, the USDT metadata will be stored in the ELF's .note.stapstd section.
- When register a probe, USDT tool(usually implemented based on uprobe under the hood) will read the ELF .note.stapstd section, and instrument the instruction from nop to breakpoint(int3 on x86). In such way, whenever control reaches the marker, the interrupt handler for int3 is called, and by turn the uprobe and attached eBPF program get called in kernel to process the events. If the USDT probe associated with semaphores, the front-ends need to incrementing the semaphore’s location via poking /proc/$PID/mem to enable the probe.
- After deregister the probe, USDT will instrument the instruction from breakpoint back to nop, no event get generated anymore, in the meanwhile, decrementing the semaphore's location to detach the current probe.
- A semaphore is a number which is incremented when a probe is attached and decremented when detached. This
allows a program to see if it being traced. 
- bpf_program__attach_usdt()
``` C
/* Copyright (c) 2022 Meta Platforms, Inc. and affiliates. */
SEC("usdt/./urandom_read:urand:read_without_sema")
int BPF_USDT(urand_read_without_sema, int iter_num, int iter_cnt, int buf_sz)
{

}

// SEC can be used to tell libbpf where this program should be auto-attached.
// Starting with usdt for the program type
// ./urandom_read for the path to the binary
// urand for the provider
// read_without_sema for the tracepoint name
```

## BPF CO-RE (Compile Once – Run Everywhere)

### protability problem
1. BPF programs do not control memory layout of a surrounding kernel environment.
2. kernel data structure changes

__sk_buf field 下划线版本 提供了一个稳定的view版本

BCC 采用内置的CLang和本机内置的header file 即时编译，这样解决了内存布局问题，
并且可以用#ifdef/#else做条件编译，但是有一些问题
1.Clang/LLVM combo is a big library, 编译后二进制文件过大
2.Clang/LLVM combo 编译占用资源比较大，增大prod主机的负载
3.Target主机头文件不存在
4.测试和开发过程痛苦

### High-level BPF CO-RE mechanics
- BTF type information, which allows to capture crucial pieces of information about kernel and BPF program types and code, enabling all the other parts of BPF CO-RE puzzle;

- compiler (Clang) provides means for BPF program C code to express the intent and record relocation information; BTF relocations 

- BPF loader (libbpf) ties BTFs from kernel and BPF program together to adjust compiled BPF code to specific kernel on target hosts;

- kernel, while staying completely BPF CO-RE-agnostic, provides advanced BPF features to enable some of the more advanced scenarios.

### BCC to Libbpf
https://www.pingcap.com/blog/tips-and-tricks-for-writing-linux-bpf-applications-with-libbpf/

### profile 程序
- blazesym_symbolize() 函数将栈中的地址解析为符号名和源代码位置

### 统计TCP延时
- tcpconnlat https://eunomia.dev/zh/tutorials/13-tcpconnlat/#_1
- 几个主要的跟踪点：tcp_v4_connect, tcp_v6_connect 和 tcp_rcv_state_process。
- tcp fast open https://juejin.cn/post/7152904835342270472

### TCP State
sock/inet_sock_set_state这个内核 tracepoint 上。每当 TCP 连接状态发生变化时，这个 tracepoint 就会被触发，然后执行handle_set_state函数

### TCP RTT
SEC("fentry/tcp_rcv_established")
- 根据过滤条件对 TCP 连接进行过滤。
- 在hists map 中查找或者初始化对应的 histogram。
- 读取 TCP 连接的srtt_us字段，并将其转换为对数形式，存储到 histogram 中。
- 如果设置了show_ext参数，将 RTT 值和计数器累加到 histogram 的latency和cnt字段中。

### libbpf-bootstrap
https://nakryiko.com/posts/libbpf-bootstrap/

### java gc
- usdt
```
   skel->links.handle_mem_pool_gc_start = bpf_program__attach_usdt(skel->progs.handle_gc_start, env.pid,
                                    binary_path, "hotspot", "mem__pool__gc__begin", NULL);
    if (!skel->links.handle_mem_pool_gc_start) {
        err = errno;
        fprintf(stderr, "attach usdt mem__pool__gc__begin failed: %s\n", strerror(err));
        goto cleanup;
    }

    skel->links.handle_mem_pool_gc_end = bpf_program__attach_usdt(skel->progs.handle_gc_end, env.pid,
                                binary_path, "hotspot", "mem__pool__gc__end", NULL);
    if (!skel->links.handle_mem_pool_gc_end) {
        err = errno;
        fprintf(stderr, "attach usdt mem__pool__gc__end failed: %s\n", strerror(err));
        goto cleanup;
    }

```
### LSM 安全
- SEC("lsm/socket_connect") 宏指出该程序期望的挂载点

### tc流量控制
-  SEC("tc")

### XDP
- SEC("xdp")

### HTTP协议追踪
两种思路
#### Socket Filter
- SEC("socket")
- 用户态代码的主要目的是创建一个原始套接字（raw socket），然后将先前在内核中定义的eBPF程序附加到该套接字上，从而允许eBPF程序捕获和处理从该套接字接收到的网络数据包,例如：
```
/* Create raw socket for localhost interface */
    sock = open_raw_sock(interface);
    if (sock < 0) {
        err = -2;
        fprintf(stderr, "Failed to open raw socket\n");
        goto cleanup;
    }

    /* Attach BPF program to raw socket */
    prog_fd = bpf_program__fd(skel->progs.socket_handler);
    if (setsockopt(sock, SOL_SOCKET, SO_ATTACH_BPF, &prog_fd, sizeof(prog_fd))) {
        err = -3;
        fprintf(stderr, "Failed to attach to raw socket\n");
        goto cleanup;
    }
```
#### Syscall 
- SEC("tracepoint/syscalls/sys_enter_accept")
- SEC("tracepoint/syscalls/sys_exit_accept")
- SEC("tracepoint/syscalls/sys_enter_read")
- SEC("tracepoint/syscalls/sys_exit_read")
sys_enter_accept：定义在accept系统调用的入口处，用于捕获accept系统调用的参数，并将它们存储在哈希映射中。
sys_exit_accept：定义在accept系统调用的退出处，用于处理accept系统调用的结果，包括获取和存储新的套接字文件描述符以及建立连接的相关信息。
sys_enter_read：定义在read系统调用的入口处，用于捕获read系统调用的参数，并将它们存储在哈希映射中。
sys_exit_read：定义在read系统调用的退出处，用于处理read系统调用的结果，包括检查读取的数据是否为HTTP流量，如果是，则发送事件。

在sys_exit_accept和sys_exit_read中，还涉及一些数据处理和事件发送的逻辑，例如检查数据是否为HTTP连接，组装事件数据，并使用bpf_perf_event_output将事件发送到用户空间供进一步处理。


### sockops 加速网络请求转发
- https://preliminary.istio.io/latest/zh/blog/2022/merbridge/
- bpf_contrack.bpf.c 中的 BPF 代码定义了一个套接字操作（sockops）程序，它的功能主要是当本机（使用 localhost）上的任意 TCP 连接被创建时，根据这个新连接的五元组（源地址，目标地址，源端口，目标端口，协议），在 sock_ops_map 这个 BPF MAP 中创建一个条目。这个 BPF MAP 被定义为 BPF_MAP_TYPE_SOCKHASH 类型，可以存储套接字和对应的五元组。这样使得每当本地 TCP 连接被创建的时候，这个连接的五元组信息也能够在 BPF MAP 中找到。

- bpf_redirect.bpf.c 中的 BPF 代码定义了一个网络消息 (sk_msg) 处理程序，当本地套接字上有消息到达时会调用这个程序。然后这个 sk_msg 程序检查该消息是否来自本地地址，如果是，根据获取的五元组信息（源地址，目标地址，源端口，目标端口，协议）在 sock_ops_map 查找相应的套接字，并将该消息重定向到在 sock_ops_map 中找到的套接字上，这样就实现了绕过内核网络栈。

### uprobe 捕获多种库的 SSL/TLS 明文数据
- SEC("uretprobe/SSL_read")
- SEC("uretprobe/SSL_write")
- SEC("uprobe/do_handshake")
- SEC("uretprobe/do_handshake")

### 测量函数延迟
- 使用 kprobes 和 kretprobes（用于内核函数）或 uprobes 和 uretprobes（用于用户空间函数）
