# 启动方式 MBR 和 UEFI 区别

非常好的问题！EFI（UEFI）和 BIOS 是两种不同的**固件引导标准**，它们在启动方式、分区表、兼容性等方面有很大区别。

---

## 一、基本概念

### BIOS（Basic Input/Output System）

**年代：** 1975 年诞生，使用了 40+ 年的老标准

**特点：**
- 16 位实模式
- 使用 MBR 分区表
- 引导加载器存储在硬盘的第一个扇区（446 字节）
- 简单、成熟、兼容性好

---

### UEFI（Unified Extensible Firmware Interface）

**年代：** 2005 年发布，现代计算机的标准

**前身：** EFI（Extensible Firmware Interface），由 Intel 开发

**特点：**
- 32 位或 64 位模式
- 使用 GPT 分区表
- 引导加载器是独立的 `.efi` 文件，存储在 EFI 分区
- 功能强大、安全性更高

---

## 二、详细对比

### 1. 启动流程

#### BIOS 启动流程

```
开机
  ↓
POST（加电自检）
  ↓
BIOS 读取 MBR（硬盘第一个 512 字节扇区）
  ↓
执行 MBR 中的引导代码（前 446 字节）
  ↓
引导代码加载第二阶段引导程序（如 GRUB Stage 1.5）
  ↓
加载完整的引导加载器（如 GRUB Stage 2）
  ↓
引导加载器读取配置文件，显示菜单
  ↓
加载内核（vmlinuz）和 initrd
  ↓
启动操作系统
```

**特点：**
- 引导代码只有 446 字节，空间非常有限
- 需要多阶段引导（MBR → Stage 1.5 → Stage 2）
- 引导代码和分区表混在同一个扇区

---

#### UEFI 启动流程

```
开机
  ↓
POST（加电自检）
  ↓
UEFI 固件读取 GPT 分区表
  ↓
查找 EFI 系统分区（ESP，FAT32 格式）
  ↓
读取 NVRAM 中的启动项配置
  ↓
加载 EFI 引导加载器（如 /EFI/BOOT/BOOTX64.EFI）
  ↓
引导加载器显示菜单（如 GRUB）
  ↓
加载内核和 initrd
  ↓
启动操作系统
```

**特点：**
- 引导加载器是完整的文件（几百 KB），不受大小限制
- 一步到位，无需多阶段引导
- 引导配置存储在 NVRAM 中，更灵活

---

### 2. 分区表

| 特性 | MBR（BIOS） | GPT（UEFI） |
|------|------------|------------|
| **最大磁盘** | 2TB | 18EB（百亿 TB） |
| **最大分区数** | 4 个主分区（或 3+扩展） | 128 个（理论无限） |
| **分区表位置** | 第一个扇区（512 字节） | 硬盘开头和末尾（各一份备份） |
| **数据完整性** | 无校验 | CRC32 校验和 |
| **分区 ID** | 4 字节整数 | 128 位 GUID（全局唯一） |

**MBR 结构：**
```
┌────────────────────────────┐
│ 引导代码（446 字节）         │
├────────────────────────────┤
│ 分区表（64 字节，4 个分区） │
├────────────────────────────┤
│ 魔数 0x55AA（2 字节）       │
└────────────────────────────┘
```

**GPT 结构：**
```
┌────────────────────────────┐
│ 保护性 MBR（兼容旧系统）    │
├────────────────────────────┤
│ GPT 头（主）               │
├────────────────────────────┤
│ 分区表（128 个分区）        │
├────────────────────────────┤
│ ...分区数据...             │
├────────────────────────────┤
│ 分区表（备份）             │
├────────────────────────────┤
│ GPT 头（备份）             │
└────────────────────────────┘
```

---

### 3. EFI 系统分区（ESP）

**UEFI 特有的分区，BIOS 没有。**

```bash
# 查看 EFI 分区
lsblk -f

# 输出示例（UEFI 系统）
NAME   FSTYPE LABEL MOUNTPOINT
vda                
├─vda1 vfat   EFI   /boot/efi     ← EFI 系统分区（100-512MB）
└─vda2 ext4         /

# BIOS 系统（无 EFI 分区）
NAME   FSTYPE MOUNTPOINT
vda                
└─vda1 ext4   /
```

**EFI 分区内容：**
```
/boot/efi/
├── EFI/
│   ├── BOOT/
│   │   └── BOOTX64.EFI    ← 默认引导加载器（x64）
│   │   └── BOOTAA64.EFI   ← ARM64 版本
│   ├── debian/
│   │   └── grubx64.efi    ← Debian 的 GRUB
│   ├── ubuntu/
│   │   └── shimx64.efi    ← Ubuntu 的 Shim（安全启动）
│   └── Microsoft/
│       └── Boot/
│           └── bootmgfw.efi ← Windows 引导管理器
└── startup.nsh              ← UEFI Shell 脚本（可选）
```

---

### 4. 引导加载器

#### BIOS 引导（GRUB Legacy）

```bash
# GRUB 安装位置
/boot/grub/
├── stage1          # 第一阶段（安装到 MBR 的 446 字节）
├── stage2          # 第二阶段（完整引导程序）
├── menu.lst        # 配置文件（老版本）
└── grub.conf       # 配置文件（新版本）

# 安装 GRUB 到 MBR
grub-install /dev/sda
```

---

#### UEFI 引导（GRUB2）

```bash
# GRUB 安装位置
/boot/efi/EFI/debian/
└── grubx64.efi     # 完整的 GRUB 引导加载器（几百 KB）

/boot/grub/
└── grub.cfg        # 配置文件

# 安装 GRUB 到 EFI 分区
grub-install --target=x86_64-efi --efi-directory=/boot/efi
```

---

### 5. 启动菜单配置

#### BIOS（存储在硬盘）

```bash
# /boot/grub/grub.cfg
menuentry 'Debian GNU/Linux' {
    set root='hd0,msdos1'        # ← msdos1 表示 MBR 分区表的第一个分区
    linux /vmlinuz root=/dev/sda1
    initrd /initrd.img
}
```

---

#### UEFI（存储在 NVRAM + 硬盘）

```bash
# NVRAM 启动项（固件中）
efibootmgr -v
# BootCurrent: 0001
# Boot0000* debian    HD(...)/File(\EFI\debian\grubx64.efi)
# Boot0001* Windows   HD(...)/File(\EFI\Microsoft\Boot\bootmgfw.efi)

# /boot/grub/grub.cfg
menuentry 'Debian GNU/Linux' {
    set root='hd0,gpt2'          # ← gpt2 表示 GPT 分区表的第二个分区
    linux /vmlinuz root=/dev/vda2
    initrd /initrd.img
}
```

**关键区别：**
- BIOS：启动顺序只能在 BIOS 设置界面修改
- UEFI：可以用 `efibootmgr` 命令直接修改启动项

---

### 6. 安全启动（Secure Boot）

| 特性 | BIOS | UEFI |
|------|------|------|
| **安全启动** | ❌ 不支持 | ✅ 支持 |
| **数字签名验证** | ❌ 无 | ✅ 验证引导加载器签名 |
| **防恶意软件** | ❌ 无防护 | ✅ 阻止未签名的引导程序 |

**UEFI 安全启动流程：**
```
固件验证引导加载器的数字签名
  ↓
签名有效？
  ↓ 是
引导加载器验证内核签名
  ↓
签名有效？
  ↓ 是
启动系统

如果任何签名验证失败 → 拒绝启动
```

**问题：** 自己编译的内核或第三方引导程序可能无法通过验证，需要禁用安全启动或添加签名。

---

### 7. 功能对比表

| 功能 | BIOS | UEFI |
|------|------|------|
| **启动速度** | 慢 | 快（并行初始化） |
| **图形界面** | ❌ 仅文本 | ✅ 支持图形和鼠标 |
| **网络启动** | ⚠️ 复杂（需 PXE） | ✅ 原生支持（HTTP/FTP） |
| **多操作系统** | ⚠️ 麻烦（需手动配置） | ✅ 每个系统独立 .efi 文件 |
| **远程管理** | ❌ 无 | ✅ 支持（iLO/iDRAC） |
| **驱动加载** | ❌ 无 | ✅ 支持 EFI 驱动模块 |
| **Shell** | ❌ 无 | ✅ 内置 UEFI Shell |
| **磁盘大小** | 2TB | 无限制 |

---

## 三、如何判断当前系统是 BIOS 还是 UEFI？

### 方法 1：检查 `/sys/firmware/efi` 目录

```bash
ls /sys/firmware/efi
```

- **存在** → UEFI 模式
- **不存在** → BIOS 模式

---

### 方法 2：检查分区表类型

```bash
fdisk -l /dev/sda | grep "Disklabel type"
```

- **输出 `gpt`** → 通常是 UEFI
- **输出 `dos`（即 MBR）** → 通常是 BIOS

**注意：** GPT 也可以用于 BIOS（需要特殊分区），MBR 也可以用于 UEFI（需要 CSM）

---

### 方法 3：检查 EFI 分区

```bash
lsblk -f | grep vfat
```

- **有 `vfat` 类型分区挂载在 `/boot/efi`** → UEFI
- **无** → BIOS

---

### 方法 4：Windows 系统

```cmd
# 按 Win+R，输入 msinfo32，回车
# 查看 "BIOS 模式" 一栏

# 或用 PowerShell
bcdedit | findstr /i "path"
```

- **`\EFI\Microsoft\Boot\bootmgfw.efi`** → UEFI
- **`\Windows\system32\winload.exe`** → BIOS

---

## 四、在 `reinstall.sh` 中的影响

### 自动检测引导模式

```bash
detect_boot_mode() {
    if [ -d /sys/firmware/efi ]; then
        boot_mode="efi"
    else
        boot_mode="bios"
    fi
    echo "Boot mode: $boot_mode"
}
```

---

### 不同模式的分区方案

#### BIOS 模式

```bash
# MBR 分区表
parted /dev/vda mklabel msdos
parted /dev/vda mkpart primary ext4 1MiB 100%

# 格式化
mkfs.ext4 /dev/vda1

# 安装 GRUB
grub-install --target=i386-pc /dev/vda
```

---

#### UEFI 模式

```bash
# GPT 分区表
parted /dev/vda mklabel gpt

# EFI 分区（100MB，FAT32）
parted /dev/vda mkpart primary fat32 1MiB 101MiB
parted /dev/vda set 1 esp on

# 根分区
parted /dev/vda mkpart primary ext4 101MiB 100%

# 格式化
mkfs.vfat /dev/vda1
mkfs.ext4 /dev/vda2

# 挂载
mount /dev/vda2 /mnt
mkdir -p /mnt/boot/efi
mount /dev/vda1 /mnt/boot/efi

# 安装 GRUB
grub-install --target=x86_64-efi --efi-directory=/mnt/boot/efi --bootloader-id=debian
```

---

### `--force-boot-mode` 参数

```bash
bash reinstall.sh windows --image-name "Windows 11 Pro" \
    --force-boot-mode bios
```

**用途：**
- 某些云服务商（如 GCP）的 UEFI 模式安装 Windows 会反复重启
- 强制用 BIOS 模式可以解决
- 安装后可以用 MBR2GPT 工具转换成 UEFI

---

## 五、兼容性：CSM（Compatibility Support Module）

### CSM 是什么？

UEFI 固件中的**兼容模式**，可以模拟 BIOS 环境，让老系统（如 Windows 7）在 UEFI 主板上运行。

**启用 CSM 后：**
- UEFI 主板可以启动 MBR 分区的系统
- 引导流程和 BIOS 一样

**禁用 CSM 后：**
- 只能启动 UEFI 引导的系统（GPT + EFI 分区）

---

### CSM 的问题

```
UEFI + CSM + MBR 分区 = ❌ 混乱
```

**常见问题：**
- Windows 安装在 MBR 分区，Linux 安装在 GPT 分区 → 双系统无法共存
- 安装时用 UEFI，启动时用 BIOS → 找不到引导程序
- 安全启动失效（CSM 模式下不支持）

---

## 六、实际案例

### 案例 1：安装 Debian（自动检测）

```bash
bash reinstall.sh debian 12
```

**脚本行为：**
```
检测 /sys/firmware/efi
  ↓ 存在
使用 UEFI 模式
  ↓
创建 GPT 分区表
  ↓
创建 EFI 分区（100MB，FAT32）
创建根分区（剩余空间，ext4）
  ↓
安装 grub-efi-amd64
```

---

### 案例 2：GCP 安装 Windows（强制 BIOS）

```bash
bash reinstall.sh windows --image-name "Windows Server 2022" \
    --force-boot-mode bios
```

**原因：**
- GCP 的 UEFI 模式有 bug，Windows 安装后会反复重启
- 强制用 BIOS 模式可以正常安装

**脚本行为：**
```
忽略 /sys/firmware/efi
  ↓
使用 BIOS 模式
  ↓
创建 MBR 分区表
  ↓
安装 Windows 到 MBR 分区
  ↓
写入 MBR 引导代码
```

---

## 七、总结

| 维度 | BIOS | UEFI |
|------|------|------|
| **年代** | 1975 ~ 现在 | 2005 ~ 现在 |
| **位数** | 16 位 | 32/64 位 |
| **分区表** | MBR（2TB 限制） | GPT（无限制） |
| **引导位置** | MBR 前 446 字节 | EFI 分区的 .efi 文件 |
| **配置存储** | 硬盘 | NVRAM + 硬盘 |
| **安全启动** | ❌ | ✅ |
| **启动速度** | 慢 | 快 |
| **图形界面** | ❌ | ✅ |
| **适用场景** | 老设备、兼容性 | 新设备、安全性 |

**推荐：**
- 新机器、新系统 → 用 **UEFI**
- 老机器、老系统 → 可能只支持 **BIOS**
- VPS 重装系统 → 通常**自动检测**，特殊情况用 `--force-boot-mode`