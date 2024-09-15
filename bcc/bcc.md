# BCC example

## install
`
sudo apt-get install bpfcc-tools linux-headers-$(uname -r)
`


## 可观测性

1. helloworld
```
# 默认python环境有问题，用conda创建新的环境
conda active pythondev

sudo python3 hellowrld.py
```

```
from bcc import BPF
BPF(text='int kprobe__sys_clone(void *ctx) { bpf_trace_printk("Hello, World!\\n"); return 0; }').trace_print()
```
- text='...': This defines a BPF program inline. The program is written in C.
- kprobe__sys_clone(): This is a short-cut for kernel dynamic tracing via kprobes. If the C function begins with kprobe__, the rest is treated as a kernel function name to instrument, in this case, sys_clone().
-void *ctx: ctx has arguments, but since we aren't using them here, we'll just cast it to void *.
-bpf_trace_printk(): A simple kernel facility for printf() to the common trace_pipe (/sys/kernel/debug/tracing/trace_pipe). This is ok for some quick examples, but has limitations: 3 args max, 1 %s only, and trace_pipe is globally shared, so concurrent programs will have clashing output. A better interface is via BPF_PERF_OUTPUT(), covered later.
-return 0;: Necessary formality (if you want to know why, see #139).
-.trace_print(): A bcc routine that reads trace_pipe and prints the output.

2. hello_fields
- b.attach_kprobe(event=b.get_syscall_fnname("clone"), fn_name="hello"): 

- b.trace_fields(): Returns a fixed set of fields from trace_pipe. Similar to trace_print(), this is handy for hacking, but for real tooling we should switch to BPF_PERF_OUTPUT().

3. sync_time

- BPF_HASH(last): Creates a BPF map object that is a hash (associative array), called "last". We didn't specify any further arguments, so it defaults to key and value types of u64.
- key = 0: We'll only store one key/value pair in this hash, where the key is hardwired to zero.
- last.lookup(&key): Lookup the key in the hash, and return a pointer to its value if it exists, else NULL. We pass the key in as an address to a pointer.

4. disk snoop
- use req pointer as the key
- req->__data_len: We're dereferencing members of struct request. See its definition in the kernel source for what members are there. bcc actually rewrites these expressions to be a series of bpf_probe_read_kernel() calls. Sometimes bcc can't handle a complex dereference, and you need to call bpf_probe_read_kernel() directly.

5. hello perf output
```
from bcc import BPF

# define BPF program
prog = """
#include <linux/sched.h>

// define output data structure in C
struct data_t {
    u32 pid;
    u64 ts;
    char comm[TASK_COMM_LEN];
};
BPF_PERF_OUTPUT(events);

int hello(struct pt_regs *ctx) {
    struct data_t data = {};

    data.pid = bpf_get_current_pid_tgid();
    data.ts = bpf_ktime_get_ns();
    bpf_get_current_comm(&data.comm, sizeof(data.comm));

    events.perf_submit(ctx, &data, sizeof(data));

    return 0;
}
"""

```
- use self defined struct to output
- events.perf_submit(): Submit the event for user space to read via a perf ring buffer.

6. bitehist.py
- BPF_HISTOGRAM(dist);

7. urandomread.py
- tracepoint
8. strlen_count.py
- bpf_probe_read_user(),This attempts to safely read size bytes from user address space to the BPF stack, so that BPF can later operate on it. 
- b.attach_uprobe(name="c", sym="strlen", fn_name="count"): Attach to library "c" (if this is the main program, use its pathname), instrument the user-level function strlen(), and on execution call our C function count().
9. nodejs_http_server.py
- instruments a user statically-defined tracing (USDT) probe
- bpf_usdt_readarg(6, ctx, &addr): Read the address of argument 6 from the USDT probe into addr.
- u = USDT(pid=int(pid)): Initialize USDT tracing for the given PID.
- u.enable_probe(probe="http__server__request", fn_name="do_trace"): Attach our do_trace() BPF C function to the Node.js http__server__request USDT probe.
- b = BPF(text=bpf_text, usdt_contexts=[u]): Need to pass in our USDT object, u, to BPF object creation.

10. mysql uprobe
RecordMysqlQuery.c

```
const std::string ALLOC_QUERY_FUNC = "_Z11alloc_queryP3THDPKcj";

bpf.attach_uprobe(mysql_path, ALLOC_QUERY_FUNC, "probe_mysql_query");


```

```
sudo bpftrace -e '
uprobe:/home/openxs/dbs/maria10.5/bin/mariadbd:_Z16dispatch_com
mand19enum_server_commandP3THDPcjbb { @sql[tid] = str(arg2);
@start[tid] = nsecs; }
uretprobe:/home/openxs/dbs/maria10.5/bin/mariadbd:_Z16dispatch_
command19enum_server_commandP3THDPcjbb /@start[tid] != 0/ {
printf("%s : %u %u ms\n", @sql[tid], tid, (nsecs -
@start[tid])/1000000); } 
```
