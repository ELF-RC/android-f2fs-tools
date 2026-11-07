# android-f2fs-tools

在 GitHub Actions 上构建 [f2fs-tools](https://git.kernel.org/pub/scm/linux/kernel/git/jaegeuk/f2fs-tools.git)
与 [ImgKit-Scuti](https://github.com/Kindness-Kismet/ImgKit-Scuti) 的 Linux 二进制包。

- f2fs-tools 使用 kernel.org 真上游；GitHub 上的 `jaegeuk/f2fs-tools` 已废弃，
  AOSP 的 `platform/external/f2fs-tools` 也指向这个上游。
- ImgKit 使用 `Kindness-Kismet/ImgKit-Scuti`（最新 tag 为 `v1.2.7`，Rust 实现）。

本仓库只保存工作流，源码全部由工作流在运行时拉取。

## 工作流

`.github/workflows/build.yml`，构建矩阵：

| arch | runner | 说明 |
| --- | --- | --- |
| `amd64` | `ubuntu-22.04` | 原生 x86_64 |
| `aarch64` | `ubuntu-22.04-arm` | 原生 ARM64（非 QEMU） |

触发方式：

- push 到 `main`
- push 任意 tag
- pull request
- `workflow_dispatch`（可指定 `f2fs_ref` / `imgkit_ref`）

## 构建步骤

1. 设置时区为 `Asia/Shanghai`（同时设置 `TZ`）
2. `apt` 安装编译依赖
3. 克隆 f2fs-tools（默认 `master`）与 ImgKit-Scuti（默认 `main`）
4. 安装 Rust stable，构建 ImgKit-Scuti（`cargo build --release`）
5. 构建 f2fs-tools（`autogen.sh` → `configure` → `make` → `make install`）
6. 上传全部二进制产物，收集全部日志

f2fs-tools 采用 `--enable-static --disable-shared`：`libf2fs` 静态链接进工具，
系统库（libuuid / libselinux / libblkid / liblz4 / liblzo2）保持动态链接。

## 产物

每个 arch 各上传两个 artifact，均由 GitHub Actions 自动压成 zip：

| artifact | 实际下载文件名 | 内容 |
| --- | --- | --- |
| `f2fs-tools-{arch}-{日期}-{tag}` | `f2fs-tools-{arch}-{日期}-{tag}.zip` | 根目录下 13 个二进制：`mkfs.f2fs`、`fsck.f2fs`、`dump.f2fs`、`defrag.f2fs`、`resize.f2fs`、`sload.f2fs`、`f2fslabel`、`inject.f2fs`、`fibmap.f2fs`、`parse.f2fs`、`f2fscrypt`、`f2fs_io`、`imgkit_scuti` |
| `build-logs-{arch}` | `build-logs-{arch}.zip` | 全流程日志（任何一步失败也会上传） |

`{日期}` 为上海时区的 `YYYYMMDD`；`{tag}` 在 tag 触发时使用仓库 tag，
否则使用 f2fs-tools 的 `git describe --tags --always`，例如
`v1.16.0-123-g83fd39a`。

> GitHub artifact 的 zip 不保留执行位，解压后需要：`chmod +x *`。

`dump.f2fs`、`defrag.f2fs`、`resize.f2fs`、`sload.f2fs`、`f2fslabel`、
`inject.f2fs` 在 f2fs-tools 里通过 `argv[0]` 分派，包内按独立二进制输出
（内容与 `fsck.f2fs` 相同）。

## 本地等价构建

```bash
# 依赖（Ubuntu 22.04）
sudo apt-get install -y build-essential autoconf automake libtool libtool-bin pkg-config m4 \
  uuid-dev libselinux1-dev libblkid-dev liblzo2-dev liblz4-dev \
  cmake nasm file git

# f2fs-tools
git clone https://git.kernel.org/pub/scm/linux/kernel/git/jaegeuk/f2fs-tools.git
cd f2fs-tools
sh ./autogen.sh
./configure --prefix=/usr --enable-static --disable-shared \
  --with-selinux --with-blkid --with-lz4 --with-lzo2
make -j"$(nproc)"
make install DESTDIR="$PWD/stage"

# ImgKit-Scuti
git clone https://github.com/Kindness-Kismet/ImgKit-Scuti.git
cd ImgKit-Scuti && cargo build --release
```

## 注意

- `ubuntu-22.04-arm` 需要仓库为公开仓库，或账户具备 ARM64 runner 权限。
  不可用时改为 `ubuntu-22.04` + QEMU，或交叉编译（参考 ImgKit-Scuti 官方 CI）。
- tag 触发时 `{tag}` 取仓库 tag；普通分支运行取 f2fs-tools 的 describe 结果。
- 产物为 Ubuntu 22.04（glibc 2.35）构建，不适用于 Android bionic 环境。
