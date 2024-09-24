## aya

### 安装
- https://aya-rs.dev/book/start/development/#the-lifecycle-of-an-ebpf-program

## start a new project

### cargo generate https://github.com/aya-rs/aya-template报错，
  原因是git开启 http proxy， aya 不能支持模板下载

  解决方法将https://github.com/aya-rs/aya-template clone 到本地
  cargo generate -p aya-template

### xdp program

- RUST_LOG=info cargo xtask run -- --iface wlp112s0
