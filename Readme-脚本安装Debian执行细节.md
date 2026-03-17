# DD重装脚本 (Debian → Debian) 完整执行流程详解

这个脚本的核心思路非常精妙：**它不能直接在运行中的系统上 dd 自身所在的硬盘（刀割自己），因此它的解决方案是：先借一个"中间操作系统"起来，再由那个中间系统执行真正的 dd 操作**。

整个过程分为两大阶段：

---

### 阶段一：在原 Debian 上运行 `reinstall.sh`（准备环境）

**你执行的命令类似：**
```bash
bash reinstall.sh dd --img "https://example.com/debian.img.xz"
```

#### 1. 环境检测与合法性校验

脚本首先做一系列检查：
- 确认以 `bash` 运行（不是 `sh`），否则自动切换
- 检测是否在容器（OpenVZ/LXC）里运行，如果是则直接报错退出（容器无法操控宿主硬盘）
- 检测当前地理位置（通过访问 `qualcomm.cn`），以便国内用户自动选择 CNB 镜像源

#### 2. 测试镜像 URL 可达性并识别文件类型

脚本用 `curl` 拉取镜像的前 1MB 头部数据，然后用 `file` 命令递归分析压缩层级，识别真实格式（`raw.xz`、`raw.gz`、`raw.zst`、`vhd.gz` 等）。这样在后面 Alpine 环境里就知道用什么命令解压。

#### 3. 下载 Alpine Linux 的 netboot 内核和 initrd

脚本的日志会显示类似这样的步骤：
```
***** DOWNLOAD VMLNUZ AND INITRD *****
http://mirror.nju.edu.cn/alpine/v3.22/releases/x86_64/netboot/vmlinuz-lts
http://mirror.nju.edu.cn/alpine/v3.22/releases/x86_64/netboot/initramfs-lts
```

这两个文件被保存到当前系统的 `/boot/` 目录（或某个固定路径，如 `/reinstall-vmlinuz` 和 `/reinstall-initramfs`）。

**为什么用 Alpine？** Alpine 的 netboot initrd 极小（约几十 MB），只需要 256MB 内存即可运行，而且是完整可操作的 Linux 环境，可以用来运行 dd/wget。

#### 4. 修改 Alpine initrd，注入 trans.sh

日志中会看到 `***** MOD ALPINE INITRD *****` 的步骤，然后是下载 `trans.sh` 和 `initrd-network.sh` 等脚本。

这一步极为关键：

- 将 Alpine 的 `initramfs` 解压（它本质是 cpio 格式）
- 将 `trans.sh`（真正的安装/DD脚本）、`initrd-network.sh`（网络配置脚本）等文件注入进去
- 将所有安装参数（如 `finalos_distro=dd`、镜像 URL、磁盘信息等）打包成内核命令行参数
- 重新打包 initramfs

这样，Alpine 启动后，其 init 脚本会在开机时自动调用 `trans.sh`，无需人工干预。

#### 5. 修改 GRUB，添加引导菜单项

这是脚本在原系统上做的最后一步，也是最重要的一步。

脚本会向 `/boot/grub/custom.cfg` 写入类似这样的内容：
```
set timeout_style=menu
set timeout=5
menuentry "reinstall (debian)" --unrestricted {
    insmod lvm
    insmod all_video
    search --no-floppy --file --set=root /reinstall-vmlinuz
    linux /reinstall-vmlinuz alpine_repo=http://dl-cdn.alpinelinux.org/alpine/v3.22/main \
        modloop=http://...netboot/modloop-lts \
        console=ttyS0,115200n8 console=tty0 \
        finalos_distro=dd \
        finalos_img='https://example.com/debian.img.xz' \
        extra_main_disk=<磁盘UUID>
    initrd /reinstall-initramfs
}
```

注意几个细节：
- **使用文件路径搜索而不是硬编码设备名**（`search --no-floppy --file --set=root /reinstall-vmlinuz`）：因为每次启动硬盘名字可能变化（`sda`/`vda`/`nvme0n1`），用 GRUB 搜索文件的方式更稳定
- **`extra_main_disk=<UUID>`**：脚本全程用分区表 UUID 来识别目标硬盘，防止 dd 写错盘
- `finalos_distro=dd` 告诉 trans.sh 这次是 DD 模式

#### 6. 重启机器

```bash
reboot
```

GRUB 会显示菜单，5秒内自动选择 "reinstall" 条目，引导进入 Alpine。

---

### 阶段二：在 Alpine initrd 环境中运行 trans.sh（执行真正的 DD）

机器重启后，GRUB 加载了我们注入的 `vmlinuz` + 修改过的 `initramfs`，Alpine 内核开始初始化。

#### 7. Alpine init 启动，运行 trans.sh

Alpine 的 initrd 启动后，其 `/init` 脚本会在网络就绪后执行我们注入的 `trans.sh`。

从日志可见，Alpine 启动后的第一条输出就是 `***** START TRANS *****`，此时的内核命令行参数完整保留了所有安装信息：
```
BOOT_IMAGE=/reinstall-vmlinuz alpine_repo=http://... modloop=http://... finalos_distro=dd finalos_img=... extra_main_disk=<UUID>
```

#### 8. 启动 SSH 服务器（供监控用）

trans.sh 会在 Alpine 中启动一个 SSH 服务，让你可以通过 SSH 连进来查看安装日志（`tail -fn+1 /reinstall.log`）。同时也会用 nginx 在 80 端口提供日志的 Web 访问界面。

#### 9. 加载 Alpine modloop（内核模块）

从内核命令行的 `modloop=...` 参数，脚本从网络下载 Alpine 的内核模块包（`.tar.gz`），挂载为 loop 设备并加载必要的模块（磁盘驱动、网络驱动等）。这就是有时会看到 `ERROR: modloop failed to start` 的地方——如果网络不通或签名验证失败就会在这里卡住。

#### 10. 配置网络

`initrd-network.sh` 从原系统传递过来的网络参数（IP、网关、DNS）自动配置好网卡，保证 Alpine 在这个纯 RAM 环境里也能上网。

#### 11. 识别目标磁盘

通过内核命令行中的 `extra_main_disk=<分区表UUID>`，脚本用 `lsblk` 或 `/dev/disk/by-id/` 找到对应的物理磁盘设备（如 `/dev/sda`），而非用名字去猜。

#### 12. 执行 DD 操作（核心步骤）

```bash
# 伪代码表示实际逻辑
wget -O- "https://example.com/debian.img.xz" | xz -dc | dd of=/dev/sda bs=4M status=progress
```

或者对于已经识别的压缩格式，用对应的解压工具直接流式写入，不需要先下载完整文件，节省内存和时间。

**关键：此时整个原系统已经不在运行了**（它只是存在于磁盘上，此时 Alpine 跑在纯 RAM 里），所以可以安全地向 `/dev/sda` 写入数据，不会出现"锯掉自己坐的树枝"的问题。

对于 Debian DD 到 Debian 的场景，脚本注释明确写道：**DD Linux 镜像时，不会修改镜像的任何内容**。镜像按原样写入，包括分区表、文件系统、所有配置文件，完全保留镜像原貌。

#### 13. 完成后重启

DD 写入完成，脚本执行 `reboot`。机器再次启动，这次 GRUB 找到的是新写入磁盘里的 Debian 系统，正常进入新系统。

---

### 整体流程图

```
原 Debian 系统运行 reinstall.sh
         │
         ├─ 检测环境、验证URL
         ├─ 下载 Alpine vmlinuz + initramfs
         ├─ 修改 initramfs（注入 trans.sh + 参数）
         ├─ 修改 GRUB（写入 custom.cfg）
         └─ reboot
                │
         GRUB 引导 Alpine netboot
                │
         Alpine initrd 启动（纯内存）
                │
         trans.sh 运行
         ├─ 启动 SSH/HTTP 服务（供监控）
         ├─ 加载 modloop（驱动模块）
         ├─ 配置网络
         ├─ 定位目标磁盘（用UUID）
         ├─ 下载镜像并流式 dd 写入 /dev/sdX
         └─ reboot
                │
         新 Debian 系统启动 ✓
```

---

### 设计亮点总结

这个脚本最巧妙的地方在于，它完全**不依赖任何外部工具（IPMI、VNC、启动介质）**，仅靠修改 GRUB 菜单和 initrd，就实现了在运行中的系统上自我替换的效果。它把"安装过程"外包给了一个纯内存的 Alpine 临时系统，这个临时系统在对磁盘操作时，原系统已经完全不再使用磁盘，因此 dd 操作是完全安全的。

# 使用在线镜像
好的，我来结合你的具体命令，把整个执行过程详细讲解一下。

---

## 你的命令：`bash reinstall.sh debian 12 --ssh-port 12345`

这是**功能1：安装Linux**，使用 Debian 官方 netboot 安装器安装 Debian 12，而不是 DD 镜像。流程和 DD 模式有所不同，我来逐步分解。

---

## 阶段一：在原系统上运行 reinstall.sh

### 1. 参数解析

脚本解析命令行参数，得到：

```
distro=debian
releasever=12
ssh_port=12345
```

`--ssh-port` 有双重作用：**安装过程中 Alpine 临时系统开放的 SSH 端口**，以及**写入最终 Debian 12 系统的 SSH 配置**（会被写入 preseed/cloud-init 配置中，让装好的系统 sshd 也监听 12345）。

### 2. 环境检测

- 确认以 bash 运行
- 检测是否在 OpenVZ/LXC 容器中（是则报错退出）
- 检测是否是虚拟机（影响后续是否需要加载固件）
- 探测地理位置（访问 `qualcomm.cn`），国内自动切换 CNB 镜像源

### 3. 探测硬盘信息

脚本通过解析 `/proc/mounts` 找到根分区所在的磁盘设备，例如 `/dev/vda`，并记录其分区表 UUID（不是分区 UUID，而是整块磁盘的 disk UUID），后续全程用这个 UUID 识别目标盘，防止多块盘时 dd 写错。

### 4. 探测网络配置

从当前运行的系统里读取网络信息：

- IP 地址、子网掩码、网关
- DNS 服务器
- 网卡 MAC 地址
- 是 DHCP 还是静态 IP

这些信息会被传递给 Alpine 临时环境，让 Alpine 起来后能立刻配好网络，不需要 DHCP。这对纯 IPv6 机器、`/32` 掩码、网关不在子网范围内等特殊网络环境尤为重要。

### 5. 下载 Alpine netboot 文件

根据是否在国内，从对应的镜像站下载两个文件：

```
/reinstall-vmlinuz    ← Alpine 内核
/reinstall-initramfs  ← Alpine 初始内存盘
```

Debian 12 是 amd64 的话，就下载 Alpine x86_64 的 LTS netboot。

### 6. 修改 Alpine initramfs，注入脚本

这是整个流程最精妙的步骤：

```
解压 initramfs（cpio 格式）
  ↓
注入文件：
  - trans.sh        ← 安装 Debian 的主逻辑
  - initrd-network.sh ← 网络配置脚本
  - debian.cfg      ← Debian preseed 自动应答文件模板
重新打包 initramfs
```

同时，所有安装参数被编码进**内核命令行**（cmdline），例如：

```
finalos_distro=debian
finalos_ver=12
finalos_ssh_port=12345
finalos_password=<你输入的密码或随机密码>
ip=<静态IP> gw=<网关> dns=<DNS>
extra_main_disk=<磁盘UUID>
```

### 7. 写入 GRUB 菜单

向 `/boot/grub/custom.cfg`（或 EFI 对应路径）写入引导条目：

```
menuentry "reinstall (debian)" --unrestricted {
    search --no-floppy --file --set=root /reinstall-vmlinuz
    linux  /reinstall-vmlinuz <上面所有 cmdline 参数>
    initrd /reinstall-initramfs
}
```

设置默认引导这个条目，超时 5 秒后自动启动，防止无人值守的服务器重启后卡在菜单。

### 8. 执行 reboot

原系统完成使命，重启。

---

## 阶段二：Alpine 临时环境运行（内存中）

机器重启，GRUB 加载 Alpine 内核和修改过的 initramfs，整个 Alpine 系统运行在纯 RAM 里，磁盘此时闲置。

### 9. Alpine init 启动，执行 trans.sh

Alpine 的 `/init` 在完成基础初始化后，读取内核命令行参数，调用注入的 `trans.sh`。

### 10. 启动监控服务

```
sshd  监听 12345 端口  ← 就是你传入的 --ssh-port
nginx 监听 80 端口     ← 提供 Web 日志查看界面
```

此时你就可以用 `ssh root@<你的IP> -p 12345` 连进来，实时看安装日志，即使装到一半出问题也能手动救砖。

### 11. 加载驱动模块（modloop）

从网络下载 Alpine 的 modloop（内核模块包），挂载后加载磁盘控制器、虚拟化半虚拟化驱动（virtio）等模块，确保能识别到目标磁盘。

### 12. 配置网络

`initrd-network.sh` 利用从 cmdline 传入的 IP/网关/DNS，用 `ip` 命令配置好网卡，这样 Alpine 就可以访问外网，后续下载 Debian 安装器。

### 13. 分区磁盘

trans.sh 通过分区表 UUID 找到目标磁盘（比如 `/dev/vda`），然后清除原有分区，重新分区：

```
根据引导方式不同：
  BIOS 机器  → MBR 分区表
              /dev/vda1  ext4  挂载 /
  EFI 机器   → GPT 分区表
              /dev/vda1  vfat  挂载 /boot/efi  (EFI 分区)
              /dev/vda2  ext4  挂载 /
```

注意：**不创建 swap 分区、不创建独立 /boot 分区**，最大化利用磁盘空间。

### 14. 下载并部署 Debian 安装器

这里有两条路（具体走哪条取决于内存大小和是否是虚拟机）：

**路径A：netboot 安装（默认，内存 ≥ 256MB）**

```
下载 Debian 12 的 vmlinuz 和 initrd.gz（netboot 版）
     ↓
将其放入刚分好区的磁盘分区里
     ↓
配置 GRUB，让下次重启进入 Debian 安装器的 vmlinuz
```

**路径B：云镜像安装（`--ci` 参数，或内存不足时）**

```
下载 Debian 12 cloud image（qcow2 格式）
     ↓
用 qemu-img 转成 raw，或直接 dd 写入磁盘
     ↓
resize 文件系统到整个磁盘大小
```

### 15. 写入 preseed 自动应答文件

trans.sh 根据模板生成 `debian.cfg`（preseed 文件），内容包括：

```
d-i passwd/root-password-crypted  password <加密后的密码>
d-i openssh-server/port           string 12345   ← 你的端口
d-i mirror/country                string manual
d-i mirror/http/hostname          string deb.debian.org
d-i partman/...                   # 跳过手动分区（已经分好了）
d-i grub-installer/only_debian    boolean true
```

这个文件会被传递给 Debian 安装器，让整个安装过程完全无人值守。

### 16. 重启进入 Debian 安装器

```
reboot
```

GRUB 这次引导的是写入磁盘的 Debian netboot 安装器内核。

---

## 阶段三：Debian 官方安装器无人值守安装

### 17. Debian installer 启动

从网络（`deb.debian.org` 或国内镜像）拉取所有需要的包，按照 preseed 文件全自动完成：

```
语言/地区/时区设置
  ↓
磁盘分区（跳过，Alpine 已做好）
  ↓
基础系统安装（debootstrap）
  ↓
安装引导器（grub-install 到 /dev/vda）
  ↓
安装 openssh-server，配置端口 12345
  ↓
设置 root 密码
  ↓
清理，卸载分区，重启
```

### 18. 最终重启，进入全新 Debian 12

GRUB 引导磁盘上的新系统，干净的 Debian 12 启动，SSH 监听你指定的 12345 端口。

---

## 完整流程图

```
bash reinstall.sh debian 12 --ssh-port 12345
│
├─ 参数解析 / 环境检测 / 网络信息采集
├─ 下载 Alpine vmlinuz + initramfs
├─ 修改 initramfs（注入 trans.sh + debian.cfg 模板 + 网络参数）
├─ 修改 GRUB → 写入 Alpine 引导项
└─ reboot ①
        │
   GRUB → Alpine netboot（纯内存）
        │
   trans.sh 运行
   ├─ SSH(12345) + HTTP(80) 监控服务启动
   ├─ 驱动加载（modloop）
   ├─ 网络配置（静态IP，原样照抄）
   ├─ 磁盘分区（GPT/MBR，无swap，无独立/boot）
   ├─ 下载 Debian 12 netboot 内核
   ├─ 生成 preseed（含 ssh-port=12345、密码）
   ├─ 配置 GRUB → 写入 Debian installer 引导项
   └─ reboot ②
        │
   GRUB → Debian 官方 installer（netboot，无人值守）
        │
   按 preseed 全自动安装
   ├─ debootstrap 拉包
   ├─ 安装 grub / openssh-server
   ├─ SSH 端口写入 sshd_config → 12345
   └─ reboot ③
        │
   全新 Debian 12 启动 ✓
   ssh root@<IP> -p 12345
```

---

## 关于 `--ssh-port 12345` 的两层作用

| 阶段 | 端口作用 |
|------|---------|
| Alpine 临时环境（阶段二）| `sshd` 监听 `12345`，供你 SSH 进来看安装日志 |
| 最终 Debian 12（阶段三）| preseed 写入 `sshd_config`，装好的系统也监听 `12345` |

这是这个脚本设计得非常贴心的地方——一个参数，两处生效，装完直接就能连。

# 自制镜像或启动安装环境
是的，也不完全是。你有几种思路：

---

## 思路一：自制镜像（你说的 img 方式）

在本地或另一台机器上装好 Debian 12 + LNMP，配置好一切，然后：

```bash
dd if=/dev/sda | xz -c > debian-lnmp.img.xz
```

上传到某个 HTTP 服务器，之后用：

```bash
bash reinstall.sh dd --img "https://yourserver.com/debian-lnmp.img.xz"
```

**优点：** 完全一致，秒级部署，适合大量机器  
**缺点：** 镜像制作麻烦，镜像体积大，迁移到不同规格磁盘时要注意分区大小问题

---

## 思路二：用 `--hold 2` 参数，装完后、重启前手动干预

这是这个脚本专门提供的"救砖/定制"钩子，非常实用：

```bash
bash reinstall.sh debian 12 --ssh-port 12345 --hold 2
```

`--hold 2` 的效果是：**Debian 安装完成后，不立即重启，让你 SSH 进去修改新系统的内容**。此时新系统挂载在 `/target` 目录下。

你可以：

```bash
# SSH 进入 Alpine 临时环境
ssh root@<IP> -p 12345

# 新系统的根目录在 /target
chroot /target /bin/bash

# 然后在 chroot 环境里做任何事
apt update
apt install -y nginx mysql-server php-fpm
# ... 配置文件、初始化数据库等

# 做完后退出，手动重启
exit
reboot
```

**优点：** 不需要自制镜像，基于干净系统按需定制，灵活  
**缺点：** 需要人工介入，不适合批量自动化

---

## 思路三：cloud-init（最优雅的方式）

这个脚本本身支持在新系统首次启动时执行 cloud-init，你可以写一个 `user-data` 脚本，装好系统后**第一次开机自动执行**：

不过更直接的做法是，装好干净 Debian 后，在 `sshd_config` 里配置一个 **首次登录自动执行的脚本**，或者用 systemd 的 `oneshot` 服务：

```ini
# /etc/systemd/system/first-boot-setup.service
[Unit]
Description=First boot setup
After=network-online.target
ConditionPathExists=/etc/first-boot-pending

[Service]
Type=oneshot
ExecStart=/root/setup-lnmp.sh
ExecStartPost=/bin/rm /etc/first-boot-pending

[Install]
WantedBy=multi-user.target
```

结合 `--hold 2` 把这个 service 和 `setup-lnmp.sh` 预置进去，重启后自动完成安装。

---

## 总结对比

| 方式 | 自动化程度 | 灵活性 | 复杂度 |
|------|-----------|--------|--------|
| 自制 img | ⭐⭐⭐⭐⭐ | ⭐⭐ | 高（制作镜像麻烦） |
| `--hold 2` 手动 chroot | ⭐⭐ | ⭐⭐⭐⭐⭐ | 低 |
| 首次开机自动脚本 | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | 中 |

**实际场景建议：**
- 只装一台 → 用 `--hold 2` chroot 进去手动搞
- 要批量部署同一环境 → 做一次自制镜像
- 每台配置略有差异 → 首次开机脚本（参数化），或者 Ansible/Cloud-init

# Grub启动参数中链接的作用
这些链接在 GRUB 引导 Alpine 临时系统时，通过**内核命令行参数**传递给 Alpine，用于不同的启动阶段。我来详细拆解每个参数的作用：

---

## GRUB 配置完整示例

```bash
menuentry "reinstall (debian)" --unrestricted {
    search --no-floppy --file --set=root /reinstall-vmlinuz
    linux /reinstall-vmlinuz \
        alpine_repo=http://dl-cdn.alpinelinux.org/alpine/v3.22/main \
        modloop=http://dl-cdn.alpinelinux.org/alpine/v3.22/releases/x86_64/netboot/modloop-lts \
        console=ttyS0,115200n8 console=tty0 \
        finalos_distro=dd \
        finalos_img='https://example.com/debian.img.xz' \
        extra_main_disk=<UUID>
    initrd /reinstall-initramfs
}
```

---

## 各参数详解

### 1. `alpine_repo=http://dl-cdn.alpinelinux.org/alpine/v3.22/main`

**作用：** 指定 Alpine Linux 的软件包仓库地址

**使用时机：** Alpine 启动后，如果需要安装额外软件包（比如 `ntfs-3g`、`parted`、`curl`、`wget` 等），会从这个仓库下载。

**实际执行：**
```bash
# Alpine init 脚本读取这个参数后，会配置 /etc/apk/repositories
echo "http://dl-cdn.alpinelinux.org/alpine/v3.22/main" > /etc/apk/repositories

# 然后就可以安装软件
apk add ntfs-3g curl
```

**为什么需要传递：**
- Alpine netboot initramfs 是最小化的，只包含核心工具
- DD Windows 需要 `ntfs-3g` 挂载 NTFS 分区
- 下载镜像需要 `curl` 或 `wget`
- 分区操作需要 `parted`、`gdisk` 等

**国内镜像替换：**
```bash
# 如果脚本检测到在中国，会自动替换为国内镜像
alpine_repo=https://mirrors.tuna.tsinghua.edu.cn/alpine/v3.22/main
```

---

### 2. `modloop=http://dl-cdn.alpinelinux.org/.../modloop-lts`

**作用：** 指定内核模块包的下载地址

**什么是 modloop：**
Alpine netboot 的内核是精简版，不包含所有驱动模块（为了减小 vmlinuz 体积）。`modloop` 是一个压缩的 squashfs 文件系统，包含了：
```
驱动模块：
- 磁盘控制器驱动（SATA、NVMe、virtio-blk）
- 网卡驱动（e1000、virtio-net、ixgbe）
- 文件系统驱动（ext4、xfs、btrfs、ntfs）
- 虚拟化驱动（virtio、vmware、xen）
```

**使用流程：**
```bash
# Alpine init 启动后执行：
1. 从 modloop URL 下载 modloop-lts 文件（约 40-60MB）
2. 挂载为 loop 设备
   mount -t squashfs modloop-lts /lib/modules/
3. 根据硬件自动加载需要的模块
   modprobe virtio_blk
   modprobe virtio_net
   modprobe nvme
```

**为什么不打包进 initramfs：**
- 如果打包所有驱动，initramfs 会从 50MB 膨胀到 150MB+
- 网络下载 modloop 只需要一次，所有机器共享同一个文件（可以 CDN 缓存）

**常见错误：**
```
ERROR: modloop not found
```
这表示网络不通，无法下载 modloop，导致硬盘驱动无法加载，看不到 `/dev/sda`。

---

### 3. `finalos_distro=dd`

**作用：** 告诉 `trans.sh` 这次是 DD 模式，而不是安装 Debian/Ubuntu/Windows ISO

**trans.sh 中的逻辑：**
```bash
case "$finalos_distro" in
    debian|ubuntu|alpine|...)
        # 调用网络安装流程
        install_from_netboot
        ;;
    dd)
        # 调用 DD 流程
        download_and_dd_image
        ;;
    windows)
        # 调用 Windows ISO 安装流程
        install_windows_from_iso
        ;;
esac
```

**其他可能的值：**
```
finalos_distro=debian      → 网络安装 Debian
finalos_distro=alpine      → 网络安装 Alpine
finalos_distro=dd          → DD 镜像模式
finalos_distro=netboot.xyz → 引导到 netboot.xyz
```

---

### 4. `finalos_img='https://example.com/debian.img.xz'`

**作用：** 指定要 DD 的镜像文件 URL

**使用时机：** 在 `trans.sh` 执行 DD 操作时：

```bash
# trans.sh 读取这个参数
img_url="$finalos_img"

# 根据文件类型（从前面识别的 gzip.raw、xz.raw 等）选择解压工具
case "$img_type" in
    xz.raw|xz.vhd)
        curl -L "$img_url" | xz -dc | dd of=/dev/vda bs=4M status=progress
        ;;
    gzip.raw|gzip.vhd)
        curl -L "$img_url" | gzip -dc | dd of=/dev/vda bs=4M status=progress
        ;;
    zstd.tar.raw)
        curl -L "$img_url" | zstd -dc | tar xOf - | dd of=/dev/vda bs=4M
        ;;
esac
```

**为什么用单引号包裹：**
```bash
finalos_img='https://example.com/debian.img.xz'
            ↑                                  ↑
            防止 URL 中的特殊字符（& ? =）被 shell 解析
```

如果 URL 包含特殊字符而不用引号，比如：
```
https://example.com/file.xz?token=abc&expire=123
```
Shell 会把 `&` 理解为后台运行，导致参数解析错误。

---

### 5. `extra_main_disk=<UUID>`

**作用：** 指定目标硬盘的分区表 UUID（不是分区 UUID，是整个磁盘的 ID）

**为什么不用设备名：**
```bash
# 设备名可能变化：
/dev/sda  → 有时变成 /dev/vda
/dev/vda  → 有时变成 /dev/nvme0n1
/dev/hda  → 老设备名

# 分区表 UUID 是唯一且不变的
# 可以用 blkid 查看
blkid -s PTUUID -o value /dev/sda
# 输出类似：e8c9f3a2-1234-5678-90ab-cdef12345678
```

**trans.sh 中如何使用：**
```bash
# 通过 UUID 找到对应的磁盘
target_disk=$(lsblk -o NAME,PTUUID | grep "$extra_main_disk" | awk '{print "/dev/"$1}')

# 或者遍历 /dev/disk/by-id/
target_disk=$(find /dev/disk/by-id/ -type l -exec readlink -f {} \; | \
              while read disk; do 
                  if [ "$(blkid -s PTUUID -o value $disk)" = "$extra_main_disk" ]; then
                      echo $disk
                      break
                  fi
              done)

# 确保找到的是块设备，不是分区
if [ -b "$target_disk" ]; then
    echo "Target disk: $target_disk"
    dd if=... of="$target_disk" bs=4M
fi
```

**防止写错盘：**
假设服务器有多块硬盘：
```
/dev/sda  ← 系统盘（30GB），PTUUID=aaa-bbb-ccc
/dev/sdb  ← 数据盘（500GB），PTUUID=ddd-eee-fff
```

如果脚本只用设备名 `/dev/sda`，在某些情况下磁盘顺序可能变化，导致 DD 写到了数据盘，系统盘没动。

用 UUID 可以 100% 确保写入正确的磁盘。

---

### 6. `console=ttyS0,115200n8 console=tty0`

**作用：** 配置内核的控制台输出

**参数解析：**
```bash
console=ttyS0,115200n8  # 第一个控制台：串行口（COM1）
        ↑     ↑    ↑ ↑
        |     |    | └── 8 数据位
        |     |    └──── 无校验位
        |     └───────── 波特率 115200
        └─────────────── 设备 ttyS0（COM1）

console=tty0            # 第二个控制台：VGA 终端
```

**为什么需要两个 console：**

1. **VPS 通常没有 VGA 显示器**，只有串行控制台（Serial Console）
   - 阿里云、腾讯云、AWS EC2 的 "获取实例截图" 就是从串行口读取的
   - IPMI 的 SOL (Serial Over LAN) 也是接串行口

2. **物理机/本地虚拟机有显示器**，用 `tty0`（VGA 输出）

同时指定两个，内核会向两个设备都输出日志，兼容不同环境。

**实际效果：**
```bash
# VPS 用户通过商家控制台查看：
[    0.000000] Linux version 6.6.x-alpine
[    0.123456] Command line: ... finalos_img=...
# ↑ 这些启动日志会同时输出到串行口和 VGA

# 如果没有 console=ttyS0，VPS 控制台会一片空白
```

---

## 完整流程中的链接使用时序

```
┌─────────────────────────────────────────────┐
│ GRUB 读取 custom.cfg                        │
│ 解析内核命令行参数（所有参数都在这里）    │
└─────────────────┬───────────────────────────┘
                  │
                  ↓
┌─────────────────────────────────────────────┐
│ 加载 /reinstall-vmlinuz (Alpine 内核)      │
│ 加载 /reinstall-initramfs (包含 trans.sh)  │
└─────────────────┬───────────────────────────┘
                  │
                  ↓
┌─────────────────────────────────────────────┐
│ Alpine 内核启动                             │
│ 读取 cmdline: alpine_repo=... modloop=...  │
└─────────────────┬───────────────────────────┘
                  │
                  ↓
┌─────────────────────────────────────────────┐
│ 第一阶段：配置 Alpine 环境                  │
│ ① wget modloop (从 modloop URL)            │
│ ② 挂载并加载驱动模块                       │
│ ③ 配置 apk 仓库 (用 alpine_repo)           │
└─────────────────┬───────────────────────────┘
                  │
                  ↓
┌─────────────────────────────────────────────┐
│ 第二阶段：trans.sh 执行                     │
│ ① 读取 finalos_distro (判断是 DD 模式)     │
│ ② 读取 extra_main_disk (找到目标磁盘)      │
│ ③ 读取 finalos_img (下载并 DD)             │
│   curl finalos_img | xz -dc | dd of=...    │
└─────────────────┬───────────────────────────┘
                  │
                  ↓
┌─────────────────────────────────────────────┐
│ 第三阶段：DD 完成，重启                     │
│ 新系统从磁盘启动                            │
└─────────────────────────────────────────────┘
```

---

## 总结：每个链接的角色

| 参数 | 下载时机 | 下载者 | 用途 |
|------|---------|--------|------|
| `alpine_repo` | Alpine 启动后 | apk 包管理器 | 安装工具软件 |
| `modloop` | Alpine init 早期 | Alpine init 脚本 | 加载硬件驱动 |
| `finalos_img` | trans.sh 执行时 | curl/wget | DD 到磁盘 |
| `extra_main_disk` | - | - | 不是链接，是 UUID（防写错盘） |

这种设计的巧妙之处在于：**一次性把所有需要的信息通过内核命令行传递进 Alpine**，Alpine 启动后不需要再回读原系统的任何配置文件，完全独立运行，从而可以安全地覆盖整个磁盘。


