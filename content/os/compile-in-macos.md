+++
date = '2025-03-30T21:59:50+08:00'
draft = true
title = '在 Macos 上编译 linux 内核'
+++

### 1. 克隆内核源码

使用以下命令克隆特定版本的内核源码：
```bash
git clone git://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git --depth 1 -b v6.5.7
```
> **注意**：必须使用 `v6.5.7` 版本，否则补丁可能无法应用。  
> 此版本可以忽略 APFS 文件系统不区分大小写的问题，但 `make clean` 可能无法正常工作。

---

### 2. 安装必要工具

通过 Homebrew 安装以下工具：
```bash
brew install clang-format llvm make openssl
```
> 如果编译过程中提示缺少其他工具，可以根据提示安装。

---

### 3. 应用补丁

由于 macOS 缺少一些必要文件（如 `elf.h` 和 `endian.h`），需要应用补丁以解决这些问题。

执行以下命令：
```bash
cd linux
curl -O https://github.com/mastermakrela/kernel-dev/blob/main/mac_patch_6-5-7.patch
patch < mac_patch_6-5-7.patch
```

应用补丁后，可通过以下命令检查：
```bash
git status
```

示例输出：
```
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

### 4. 编译内核

执行以下命令编译内核：
```bash
PATH="$(brew --prefix llvm)/bin/:$PATH" make LLVM=1 defconfig
make LLVM=1 menuconfig  # 可选：在 General setup 中设置内核版本
time make LLVM=1 ARCH=arm64 -j $(sysctl -n hw.logicalcpu) HOSTCFLAGS="-I./"
```

#### 参数说明：
- `PATH="$(brew --prefix llvm)/bin/:$PATH"`：确保使用 Homebrew 安装的 `clang`，避免使用 Xcode 自带版本。
- `LLVM=1`：使用 `clang` 而非 `gcc`。
- `ARCH=arm64`：为 arm64 架

