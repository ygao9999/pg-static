# PostgreSQL 静态编译 for Android（脱离 Termux 包管理）

## 目标

- 工具链：复用 `ghcr.io/termux/package-builder` 镜像里的 aarch64 Android clang 交叉编译工具链
- 产物：纯命令行二进制（postgres, initdb, psql, pg_ctl 等），通过 adb shell / Termux 手动运行
- 静态范围：openssl, libxml2, icu, readline, libuuid(e2fsprogs), zlib 全部静态编译成 .a
- 链接方式：postgres 对这几个库用 `-Wl,-Bstatic ... -Wl,-Bdynamic` 部分静态链接
  （libc/libm/libdl 等系统库仍动态链接 Android bionic，因为 bionic 不支持完全静态）

## 当前阶段：Stage 1（探测环境）

由于本地开发环境（这个对话的沙盒）无法访问 ghcr.io / Docker Hub，也没有真实的
Termux/Android 环境，所以无法本地验证 termux-packages docker 镜像里的：

- aarch64 clang 交叉编译工具链的确切路径（例如 `aarch64-linux-android21-clang` 之类）
- sysroot 路径
- 是否有现成的 NDK 环境变量可用

`.github/workflows/build-pg-static.yml` 目前只做一件事：
在 `ghcr.io/termux/package-builder` 容器里跑一遍探测脚本，把：

1. 所有跟 android/ndk/clang/sysroot/prefix 相关的环境变量
2. 文件系统里能找到的 clang 交叉编译器路径
3. sysroot 目录路径
4. 一个最小的 hello world 交叉编译 smoke test

全部输出到 job log 里，并把产物（如果编译成功）作为 artifact 上传。

## 你需要做的事

1. 把这个仓库（或者只有 `.github/workflows/build-pg-static.yml` 这个文件）推到你自己的
   GitHub 仓库
2. 在 Actions 页面手动触发 `workflow_dispatch`
3. 跑完后把 job 的完整日志发给我（尤其是 "Probe toolchain environment" 和
   "Try a minimal clang cross-compile smoke test" 这两步的输出）

拿到真实路径后，我会据此写：

- `scripts/build-deps.sh`：按顺序编译 zlib → libuuid → readline → libxml2 → openssl → icu，
  每个都产出静态 `.a`，全部安装到统一的 `staging/` 目录
- `scripts/build-postgres.sh`：postgres configure 时指向 `staging/`，链接阶段对这 6 个库
  用 `-Wl,-Bstatic` 包裹，其余系统库保持动态

## 已知会踩的坑（供参考，后续脚本会处理）

- **openssl**：需要用 `Configure android-arm64 no-shared`，不能用默认 `./config`
- **icu**：官方 configure 对交叉编译支持较弱，通常需要先跑一次 host 版 ICU 生成
  `icu-config`/工具，再交叉编译 target 版（类似你们 postgres 里 host 编译 `zic` 复用的思路）
- **readline**：保留 GPL（你已确认自用，不对外分发，无需换 libedit）
- **libuuid**：来自 e2fsprogs 源码树的 `lib/uuid`，不是独立仓库，需要注意只编 uuid 子目录
- **bionic 完全静态限制**：`getpwnam`/`getaddrinfo`/NSS 相关调用在完全静态链接下可能异常，
  所以只对上述 6 个第三方库做静态，libc 本身保持动态（这点已经和 git/vim 的处理方式一致）
