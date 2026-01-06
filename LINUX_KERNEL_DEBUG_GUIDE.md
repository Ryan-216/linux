# Linux 内核编译、运行与调试指南

本文档详细记录了如何从零开始编译 Linux 内核，制作最小根文件系统，使用 QEMU 运行，并使用 GDB 进行调试的过程。本指南适合 Linux 内核开发的初学者。

## 1. 环境准备

在开始之前，需要安装一些必要的工具和依赖库。以 Ubuntu/Debian 为例：

```bash
sudo apt update
sudo apt install git build-essential libncurses-dev bison flex libssl-dev libelf-dev qemu-system-x86 busybox-static
```

*   `build-essential`: 包含 gcc, make 等基础编译工具。
*   `libncurses-dev`: 用于 `make menuconfig` 的文本图形界面。
*   `bison`, `flex`: 语法分析工具，编译内核必须。
*   `libssl-dev`, `libelf-dev`: 内核编译依赖。
*   `qemu-system-x86`: x86 架构的模拟器。
*   `busybox-static`: 静态链接的 BusyBox，用于制作根文件系统。

## 2. 编译内核

### 2.1 配置内核

我们将使用内核默认的配置 (`defconfig`)，它包含了所有标准驱动和功能，能确保在 QEMU 中顺利启动。

1.  **生成默认配置**：
    ```bash
    make defconfig
    ```

2.  **开启调试信息**：
    默认配置通常关闭了调试信息，为了后续使用 GDB 调试，我们需要手动开启它。

    执行以下命令修改配置：
    ```bash
    ./scripts/config --enable CONFIG_DEBUG_INFO
    ./scripts/config --enable CONFIG_DEBUG_INFO_DWARF_TOOLCHAIN_DEFAULT
    ./scripts/config --enable CONFIG_GDB_SCRIPTS
    make olddefconfig
    ```

### 2.2 编译

使用多核编译以加快速度（`$(nproc)` 自动获取 CPU 核心数）：

```bash
make -j$(nproc) bzImage
```

编译完成后，内核镜像位于 `arch/x86/boot/bzImage`。

## 3. 制作根文件系统 (RootFS)

内核启动后需要挂载根文件系统来运行用户空间程序（如 `init` 进程）。我们将使用 BusyBox 制作一个基于内存的初始文件系统 (initramfs)。

### 3.1 准备目录结构

创建一个工作目录 `my-rootfs` 并建立基础目录：

```bash
mkdir -p my-rootfs/{bin,sbin,etc,proc,sys,usr/bin,usr/sbin}
```

### 3.2 安装 BusyBox

将静态编译的 BusyBox 复制进去，并创建命令链接：

```bash
cp /bin/busybox my-rootfs/bin/
cd my-rootfs/bin
# 为 busybox 支持的所有命令创建符号链接
for cmd in $(./busybox --list); do ln -s busybox $cmd; done
cd ../..
```

### 3.3 编写 init 脚本

内核启动后会执行根目录下的 `/init` 脚本。创建 `my-rootfs/init`：

```bash
cat <<EOF > my-rootfs/init
#!/bin/busybox sh

# 挂载必要的伪文件系统
mount -t proc none /proc
mount -t sysfs none /sys
mount -t devtmpfs none /dev

# 启用热插拔 (如果内核支持)
if [ -e /proc/sys/kernel/hotplug ]; then
    echo /sbin/mdev > /proc/sys/kernel/hotplug
fi
mdev -s

# 打印欢迎信息
echo "Welcome to Linux Kernel!"

# 启动 Shell (尝试获取作业控制)
exec /bin/sh
EOF

chmod +x my-rootfs/init
```

### 3.4 打包 initramfs

将目录打包成 cpio 格式并压缩，这是 Linux 内核识别的 initramfs 格式：

```bash
cd my-rootfs
find . -print0 | cpio --null -ov --format=newc | gzip -9 > ../initramfs.cpio.gz
cd ..
```

## 4. 运行 QEMU

现在我们有了内核 (`bzImage`) 和文件系统 (`initramfs.cpio.gz`)，可以运行了。

```bash
qemu-system-x86_64 \
  -kernel arch/x86/boot/bzImage \
  -initrd initramfs.cpio.gz \
  -nographic \
  -append "console=ttyS0 earlyprintk=serial,ttyS0,115200"
```

*   `-kernel`: 指定内核镜像。
*   `-initrd`: 指定初始内存盘。
*   `-nographic`: 不弹出图形窗口，直接在当前终端输出。
*   `-append "..."`: 内核启动参数。
    *   `console=ttyS0`: 将日志输出到第一个串口。
    *   `earlyprintk=serial,ttyS0,115200`: 启用早期打印，防止内核在初始化控制台之前崩溃导致无输出。

**退出 QEMU**：按下 `Ctrl+A` 然后按 `X`。

## 5. GDB 调试

### 5.1 启动 QEMU 等待调试

添加 `-s -S` 参数：
*   `-s`: 开启 GDB 服务器，监听 TCP 1234 端口。
*   `-S`: 启动时冻结 CPU，等待 GDB 命令。

```bash
qemu-system-x86_64 \
  -kernel arch/x86/boot/bzImage \
  -initrd initramfs.cpio.gz \
  -nographic \
  -append "console=ttyS0" \
  -s -S
```

### 5.2 连接 GDB

在另一个终端中启动 GDB 并加载未压缩的内核镜像 `vmlinux`（包含符号表）：

```bash
gdb vmlinux
```

在 GDB 提示符下输入：

```gdb
target remote :1234    # 连接到 QEMU
break start_kernel     # 在内核入口函数设置断点
continue               # 开始运行
```

现在你可以使用 `next`, `step`, `print` 等命令调试内核了。

## 6. 常见错误排查

1.  **Kernel panic - not syncing: VFS: Unable to mount root fs**
    *   原因：内核找不到根文件系统。
    *   解决：检查是否指定了 `-initrd`，或者 initramfs 格式是否正确。

2.  **Failed to execute /init (error -8)**
    *   原因：Exec format error。内核不支持脚本或二进制格式。
    *   解决：确保开启了 `CONFIG_BINFMT_ELF` 和 `CONFIG_BINFMT_SCRIPT`（defconfig 默认已开启）。

3.  **Kernel panic - not syncing: No working init found**
    *   原因：`/init` 文件不存在，或者没有执行权限。
    *   解决：检查 `init` 脚本是否存在于 initramfs 根目录，且有 `chmod +x`。

4.  **/bin/sh: not found**
    *   原因：Shell 解释器路径错误或链接失效。
    *   解决：检查 `init` 脚本中的 shebang (`#!/bin/busybox sh`) 是否指向真实存在的文件。确保 `busybox` 是静态编译的。

5.  **启动后一片空白 (无输出)**
    *   原因：内核在初始化串口驱动之前就卡死或崩溃，或者未启用早期打印。
