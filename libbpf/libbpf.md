## libbpf

### install
```
apt install clang libelf1 libelf-dev zlib1g-dev
git clone --recurse-submodules https://github.com/libbpf/libbpf-bootstrap
```

### Step
https://facebookmicrosites.github.io/bpf/blog/2020/02/20/bcc-to-libbpf-howto-guide.html
Building libbpf-based BPF application using BPF CO-RE consists of few steps:
1. generating vmlinux.h header file with all kernel types;
2. compiling your BPF program source code using recent Clang (version 10 or newer) into .o object file;
3. generating BPF skeleton header file from compiled BPF object file;
4. including generated BPF skeleton header to use from user-space code;
5. then, at last, compiling user-space code, which will get BPF object code embedded in it, so that you don’t have to distribute extra files with your application.

