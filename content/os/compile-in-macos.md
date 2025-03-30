+++
date = '2025-03-30T21:59:50+08:00'
draft = false
title = '在 Macos 上编译 linux 内核'
+++

之前在 macOS 上尝试编译 Linux 内核时，踩了不少坑。网上的教程大多不够详细，这里我参考了一篇[文章](https://mastermakrela.com/kernel/lkp/kernel-dev-on-macos)并用**GPT4.5**整理了一下自己需要的部分，供自己和大家参考。

---

## 准备工作

### 1. 克隆 Linux 内核源码

首先，我们需要特定版本的 Linux 内核源码（这里用的是 v6.5.7）：

```bash
git clone git://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git --depth 1 -b v6.5.7
```

> ⚠️ **注意**：  
> 一定要用 v6.5.7 版本，因为后续的补丁是专门针对这个版本的。如果用其他版本，可能会导致补丁失败。

另外，macOS 的文件系统 (APFS) 默认不区分大小写，可能导致一些奇怪的问题。不过在 v6.5.7 版本，这个问题不会影响编译过程，只是 `make clean` 可能会报错，暂时可以忽略。

---

### 2. 安装编译所需工具

我们需要用 Homebrew 安装一些必要工具：

```bash
brew install clang-format llvm make openssl
```

如果后面编译时提示缺少其他工具，再根据提示补装就好。

---

### 3. 应用 macOS 专用补丁

Linux 内核源码默认是为 Linux 环境准备的，直接在 macOS 上编译会缺少一些头文件（比如 `elf.h` 和 `endian.h`），还会有其他兼容性问题。因此我们需要打一个专门为 macOS 准备的补丁：

```bash
cd linux
curl -O https://github.com/mastermakrela/kernel-dev/raw/main/mac_patch_6-5-7.patch
patch < mac_patch_6-5-7.patch
```

补丁打好后，用下面命令确认一下：

```bash
git status
```

正常情况下，你会看到类似这样的输出：

``` bash
❯ git status
Not currently on any branch.
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
 modified:   Makefile
 modified:   arch/arm64/kernel/vdso32/Makefile
 new file:   arch/arm64/kernel/vdso32/elf_helper.h
 modified:   arch/arm64/kvm/hyp/nvhe/Makefile
 new file:   arch/arm64/kvm/hyp/nvhe/endian_helper.h
 new file:   elf.h
 new file:   endian.h
 modified:   scripts/mod/file2alias.c
 modified:   scripts/subarch.include
```

---

### 4. 编译 Linux 内核

接下来就可以编译内核了。我们需要明确告诉编译工具使用 Homebrew 安装的 LLVM 工具链（而不是 macOS 自带的），然后生成默认配置：

```bash
export PATH="$(brew --prefix llvm)/bin/:$PATH"
make LLVM=1 defconfig
```

如果你希望给编译的内核加一个自定义版本后缀（方便以后识别），可以运行：

```bash
make LLVM=1 menuconfig
```

在菜单里选择：

- General setup → Local version - append to kernel release  
输入你希望的版本后缀，比如 `-mykernel`。

然后正式开始编译：

```bash
time make LLVM=1 ARCH=arm64 -j $(sysctl -n hw.logicalcpu) HOSTCFLAGS="-I./"
```

参数含义：

- `LLVM=1`：使用 LLVM 工具链（clang）而非传统的 gcc。
- `ARCH=arm64`：为 arm64 架构编译。
- `-j $(sysctl -n hw.logicalcpu)`：使用所有 CPU 核心并行编译，加快速度。
- `HOSTCFLAGS="-I./"`：将当前目录添加到 include 路径。

## 准备驱动文件和运行虚拟机

我们刚刚在 macOS 上成功编译了 Linux 内核，但光有内核是不够的，还需要完整的 Linux 文件系统（驱动文件）才能启动运行。这里介绍一下如何获取驱动文件，并用 QEMU 虚拟机运行我们刚编译的内核。

---

### 1. 下载并准备驱动文件（Linux 文件系统镜像）

最简单的方法就是用现成的 Linux 系统镜像，这里推荐 Arch Linux ARM 镜像：

- 前往 [UTM 官方镜像库](https://mac.getutm.app/gallery/archlinux-arm) 下载 Arch Linux ARM 镜像（`.utm` 文件）。

下载好后，先用 UTM 打开这个镜像，进行一些简单的初始化设置：

```bash
pacman -Syu lsof neofetch strace
```

做完这些设置后，关闭虚拟机。

接下来，我们需要从 `.utm` 文件中提取磁盘镜像文件：

- 在 Finder 中找到你刚刚下载的 `.utm` 文件。
- 右键点击，选择「显示包内容」。
- 进入 `Data` 文件夹，找到里面的 `.qcow2` 文件（类似 `arch_aarch64.qcow2`）。
- 把这个文件复制到你喜欢的目录，后面我们会用到。

---

### 2. 安装 QEMU 虚拟机工具

我们使用 QEMU 来运行虚拟机，安装方法也很简单：

```bash
brew install qemu
```

---

### 3. 用 QEMU 启动虚拟机并加载自定义内核

现在我们手上有两个关键文件：

- 自己刚刚编译的 Linux 内核文件（位于 `linux/arch/arm64/boot/Image.gz`）。
- 刚刚提取的 Arch Linux ARM 磁盘镜像文件（比如 `arch_aarch64.qcow2`）。

我们用下面的命令启动虚拟机，并加载我们编译好的内核：

```bash
qemu-system-aarch64 \
    -machine virt \
    -cpu max \
    -m 2048 \
    -drive file=arch_aarch64.qcow2,format=qcow2 \
    -serial stdio \
    -kernel ./linux/arch/arm64/boot/Image.gz \
    -append "root=/dev/vda2 rw console=ttyAMA0"
```

参数解释：

- `qemu-system-aarch64`：表示启动基于 ARM 架构（arm64）的虚拟机。
- `-machine virt`：使用通用 ARM 虚拟机硬件平台。
- `-cpu max -m 2048`：分配最大 CPU 功能，并给虚拟机分配 2GB 内存。
- `-drive file=...`：指定磁盘镜像文件路径。
- `-serial stdio`：将虚拟机的控制台输出到终端，方便调试。
- `-kernel`：指定我们刚编译好的 Linux 内核文件路径。
- `-append`：传递内核启动参数，指定根文件系统位置为 `/dev/vda2`，并以可读写模式挂载，同时指定控制台设备。

虚拟机启动后，你应该会看到 Arch Linux 的启动过程，最后进入系统。

---

### 4. 检查虚拟机内核版本

启动成功后，登录虚拟机，然后执行：

```bash
uname -a
```

或者使用更直观的工具：

```bash
neofetch
```

如果之前在编译时设置了自定义的内核版本名（比如 `-mykernel`），你应该能在输出中看到：

```
Linux (your-hostname) 6.5.7-mykernel #1 SMP PREEMPT ...
```

看到自己的内核版本号，说明你已经成功地在 macOS 上编译并运行了 Linux 内核，恭喜 🎉！

---

## 总结

到这里，我们就完成了 macOS 上编译 Linux 内核，并使用 QEMU 虚拟机运行的全部过程。虽然过程稍微复杂了一些，但踩过一次坑后，以后再做就会顺利很多。
