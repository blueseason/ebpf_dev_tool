## examples

1. list all probe

`bpftrace -l 'tracepoint:syscalls:sys_enter_*`

2. helloworld
```
bpftrace sudo bpftrace -e 'BEGIN { printf("hello world\n"); }'

Attaching 1 probe...
ERROR: Could not resolve symbol: /proc/self/exe:BEGIN_trigger

```
运行版本0.14, 错误原因 https://github.com/bpftrace/bpftrace/issues/2292

3. list 打开文件
``` 
bpftrace -e 'tracepoint:syscalls:sys_enter_openat { printf("%s %s\n", comm, str(args.filename)); }'

stdin:1:64-73: ERROR: Can not access field 'filename' on type '(ctx) struct _tracepoint_syscalls_sys_enter_openat *'. Try dereferencing it first, or using '->'
tracepoint:syscalls:sys_enter_openat { printf("%s %s\n", comm, str(args.filename)); }
                                                               ~~~~~~~~~
stdin:1:64-82: ERROR: str() expects an integer or a pointer type as first argument (none provided)
tracepoint:syscalls:sys_enter_openat { printf("%s %s\n", comm, str(args.filename)); }
                                                               ~~~~~~~~~~~~~~~~~~
#arg->filename 可以正确执行
bpftrace -e 'tracepoint:syscalls:sys_enter_openat { printf("%s %s\n", comm, str(args->filename)); }'

```
- comm是内建变量，代表当前进程的名字。其它类似的变量还有pid和tid，分别表示进程标识和线程标识。
- args是一个包含所有tracepoint参数的结构。这个结构是由bpftrace根据tracepoint信息自动生成的。
  这个结构的成员可以通过命令bpftrace -vl tracepoint:syscalls:sys_enter_openat

4. 进程级系统调用计数
```
bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[comm] = count(); }'
```
- @: 表示一种特殊的变量类型，称为map，可以以不同的方式来存储和描述数据。你可以在@后添加可选的变量名(如@num)，用来增加可读性或者区分不同的map。
- count(): 这是一个map函数 - 记录被调用次数。因为调用次数根据comm保存在map里，输出结果是进程执行系统调用的次数统计

5. read()返回值分布统计
```
bpftrace -e 'tracepoint:syscalls:sys_exit_read /pid == 18644/ { @bytes = hist(args->ret); }'
```

6. 内核动态跟踪read()返回的字节数
- kretprobe:vfs_read: 这是kretprobe类型(动态跟踪内核函数返回值)的探针，跟踪vfs_read内核函数。
- kprobe和kretprobe 是"不稳定"的探针类型

7. read()调用的时间
```
bpftrace -e 'kprobe:vfs_read { @start[tid] = nsecs; } kretprobe:vfs_read /@start[tid]/ { @ns[comm] = hist(nsecs - @start[tid]); delete(@start[tid]); }'
```
- @start[tid]: 使用线程ID作为key。每个调用记录一个起始时间戳
- nsecs: 自系统启动到现在的纳秒数。这是一个高精度时间戳，可以用来对事件计时。
- /@start[tid]/: 该过滤条件检查起始时间戳是否被记录。程序可能在某次read调用中途被启动，如果没有这个过滤条件，这个调用的时间会被统计为now-zero，而不是now-start。
- delete(@start[tid]): 释放变量。

8. 统计进程级别的事件
```
bpftrace -e 'tracepoint:sched:sched* { @[probe] = count(); } interval:s:5 { exit(); }'
```
- sched: sched探针可以探测调度器的高级事件和进程事件如fork, exec和上下文切换。
- probe: 探针的完整名称。
- interval:s:5: 这是一个每5秒在每个CPU上触发一次的探针，它用来创建脚本级别的间隔或超时时间。
- exit(): 退出bpftrace

9. 打印内核实时函数栈
```
bpftrace -e 'profile:hz:99 { @[kstack] = count(); }'
```
- profile:hz:99: 这里所有cpu都以99赫兹的频率采样分析内核栈。为什么是99而不是100或者1000？我们想要抓取足够详细的内核执行时内核栈信息，但是频率太大影响性能。100赫兹足够了，但是我们不想用正好100赫兹，这样采样频率可能与其他定时事件步调一致，所以99赫兹是一个理想的选择。
- kstack: 返回内核调用栈。这里作为map的关键字，可以跟踪次数。这些输出信息可以使用火焰图可视化。此外ustack用来分析用户级堆栈
10. 调度器跟踪
```
bpftrace -e 'tracepoint:sched:sched_switch { @[kstack] = count(); }'
```
- sched: 跟踪调度类别的调度器事件:sched_switch, sched_wakeup, sched_migrate_task等。
- sched_switch: 当线程释放cpu资源，当前不运行时触发。这里可能的阻塞事件:如等待I/O，定时器，分页/交换，锁等。
- kstack: 内核堆栈跟踪，打印调用栈。
- sched_switch在线程切换的时候触发，打印的调用栈是被切换出cpu的那个线程。像你使用其他探针一样，注意这里的上下文，例如comm, pid, kstack等等，并不一定反映了探针的目标的状态。

11. block IO
```
bpftrace -e 'tracepoint:block:block_rq_issue { @ = hist(args->bytes); }'
```
- tracepoint:block: 块类别的跟踪点跟踪块级I/O事件。
- block_rq_issue: 当I/O提交到块设备时触发。
- args.bytes: 跟踪点block_rq_issue的参数成员bytes，表示提交I/O请求的字节数。
该探针的上下文是非常重要的: 它在I/O请求被提交给块设备时触发。这通常发生在进程上下文，此时通过内核的comm可以得到进程名；也可能发生在内核上下文，(如readahead)，此时不能显示预期的进程号和进程名信息。

12. kernel structure 
```
# cat path.bt
#ifndef BPFTRACE_HAVE_BTF
#include <linux/path.h>
#include <linux/dcache.h>
#endif

kprobe:vfs_open
{
    printf("open path: %s\n", str(((struct path *)arg0)->dentry->d_name.name));
}

bpftrace path.bt
```
- kprobe: 如前面所述，这是内核动态跟踪kprobe探针类型，跟踪内核函数的调用(kretprobe探针类型跟踪内核函数返回值)。
- arg0 是一个内建变量，表示探针的第一个参数，其含义由探针类型决定。对于kprobe类型探针，它表示函数的第一个参数。其它参数使用arg1,...,argN访问。
- ((struct path *)arg0)->dentry->d_name.name: 这里arg0作为struct path *并引用dentry。
