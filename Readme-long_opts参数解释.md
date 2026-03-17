# long_opts参数解释

详细解释每个参数的用途。这些是 `reinstall.sh` 脚本支持的所有命令行参数。

---

## 参数详解

### 一、布尔开关参数（无需值，只要出现就生效）

#### 1. `--ci`
**用途：** 使用云镜像（Cloud Image）方式安装 Linux

**适用场景：**
- 默认情况下，脚本用 netboot 网络安装（下载包、现场编译）
- 加 `--ci` 后，直接下载预制的云镜像（qcow2 格式），速度更快

**示例：**
```bash
bash reinstall.sh debian 12 --ci
```

**效果：**
```bash
# 不加 --ci（传统 netboot）：
下载 vmlinuz + initrd → 分区 → debootstrap 在线安装 → 编译配置 → 30分钟

# 加 --ci（云镜像）：
下载 debian-12-genericcloud-amd64.qcow2 → 转 raw → dd 写入 → 5分钟
```

---

#### 2. `--installer`
**用途：** 强制使用官方 ISO 安装器（而不是 netboot）

**适用场景：**
- CentOS/AlmaLinux/Rocky/Fedora 默认用 netboot
- 加 `--installer` 后，下载完整 ISO，用图形化/半自动安装流程

**示例：**
```bash
bash reinstall.sh centos 9 --installer
```

**限制：**
- 需要内存 > 2GB
- 只支持动态 IP（DHCP）
- 密码固定为 `123@@@`
- SSH 端口固定为 `22`

---

#### 3. `--debug`
**用途：** 启用调试模式，输出详细日志

**效果：**
```bash
# 不加 --debug
***** DOWNLOAD VMLINUZ AND INITRD *****
http://mirror.nju.edu.cn/alpine/v3.22/...

# 加 --debug
+ curl -O http://mirror.nju.edu.cn/alpine/v3.22/...
+ sha256sum vmlinuz-lts
+ echo 'SHA256 verified'
+ ls -lh vmlinuz-lts
...（每一步命令都显示）
```

实际是在脚本开头添加 `set -x`。

---

#### 4. `--minimal`
**用途：** 安装最小化 Ubuntu（仅适用于 Ubuntu）

**示例：**
```bash
bash reinstall.sh ubuntu 24.04 --minimal
```

**效果：**
- 不安装 `ubuntu-standard`、`landscape-common` 等额外包
- 系统更精简，占用空间更小（减少约 500MB）
- 适合资源受限的小鸡

---

#### 5. `--allow-ping`
**用途：** 允许 Windows 被 Ping（仅 DD/安装 Windows 时有效）

**背景：**
Windows 默认防火墙阻止 ICMP Echo Request（Ping 请求）

**效果：**
脚本会在 Windows 首次启动时执行：
```cmd
netsh advfirewall firewall add rule name="Allow Ping" protocol=icmpv4:8,any dir=in action=allow
```

**示例：**
```bash
bash reinstall.sh dd --img "windows.vhd.xz" --allow-ping
```

---

#### 6. `--force-cn`
**用途：** 强制使用中国镜像源（即使不在中国）

**适用场景：**
- 在海外 VPS 上测试中国镜像源
- 海外机器但路由回国，用国内源更快

**示例：**
```bash
bash reinstall.sh debian 12 --force-cn
```

**效果：**
```bash
# 默认（海外）
alpine_repo=http://dl-cdn.alpinelinux.org/alpine/v3.22/main
debian_mirror=http://deb.debian.org/debian

# 加 --force-cn
alpine_repo=https://mirrors.tuna.tsinghua.edu.cn/alpine/v3.22/main
debian_mirror=https://mirrors.tuna.tsinghua.edu.cn/debian
```

---

#### 7. `--help`
**用途：** 显示帮助信息

```bash
bash reinstall.sh --help
```

输出完整的使用说明，然后退出。

---

### 二、需要值的参数（用 `:` 表示需要参数值）

#### 8. `--add-driver:`
**用途：** 为 Windows 添加额外驱动（仅安装 Windows 时有效）

**参数值：** 驱动 `.inf` 文件路径，或包含 `.inf` 的文件夹

**示例：**
```bash
# 单个驱动
bash reinstall.sh windows --image-name "Windows 11 Pro" \
    --add-driver /root/mydriver.inf

# 整个文件夹
bash reinstall.sh windows --image-name "Windows 11 Pro" \
    --add-driver /root/drivers/
```

**可多次使用：**
```bash
--add-driver /path/to/driver1.inf \
--add-driver /path/to/driver2.inf
```

**工作原理：**
脚本会把驱动复制到 `install.wim` 里的 `C:\Drivers\`，并修改注册表添加驱动搜索路径。

---

#### 9. `--hold:`
**用途：** 暂停安装流程，用于调试/手动操作

**参数值：**
- `--hold 1`：重启到 Alpine 环境后**不运行安装**，只启动 SSH，让你手动登录
- `--hold 2`：安装完成后**不重启**，让你 SSH 进去修改新系统

**示例 1：测试网络**
```bash
bash reinstall.sh alpine --hold 1 --ssh-port 12345
# 重启后 SSH 登录 Alpine 临时环境
ssh root@<IP> -p 12345
# 手动测试网络、磁盘等
# 确认没问题后，运行：
/trans.sh
```

**示例 2：安装后定制系统**
```bash
bash reinstall.sh debian 12 --hold 2
# Debian 装完后不重启，SSH 进去
ssh root@<IP> -p 22
# 新系统挂载在 /target
chroot /target /bin/bash
# 安装软件、修改配置
apt install nginx
# 完成后手动重启
reboot
```

---

#### 10. `--sleep:`
**用途：** 在关键步骤之间暂停指定秒数（调试用）

**参数值：** 秒数

**示例：**
```bash
bash reinstall.sh debian 12 --sleep 30
```

会在某些步骤（如修改 GRUB、写入 initramfs）后暂停 30 秒，方便你检查中间文件。

---

#### 11. `--iso:`
**用途：** 指定 Windows ISO 的下载链接（安装 Windows 时）

**参数值：** HTTP/HTTPS/磁力链接

**示例：**
```bash
# HTTP 链接
bash reinstall.sh windows \
    --image-name "Windows 11 Pro" \
    --iso "https://example.com/Win11.iso"

# 磁力链接
bash reinstall.sh windows \
    --image-name "Windows 10 LTSC 2021" \
    --iso "magnet:?xt=urn:btih:7352bd2d..."
```

**如果不指定：**
脚本会自动从 https://massgrave.dev 查找官方 ISO 下载链接。

---

#### 12. `--image-name:`
**用途：** 指定 Windows ISO 中的映像名称（安装 Windows 时）

**背景：**
一个 Windows ISO 通常包含多个版本（家庭版、专业版、企业版等），需要指定要安装哪个。

**参数值：** 映像名称（区分大小写）

**示例：**
```bash
bash reinstall.sh windows \
    --image-name "Windows 11 Pro" \
    --lang zh-cn
```

**常见映像名称：**
```
Windows 11 Pro
Windows 11 Enterprise
Windows 11 Enterprise LTSC 2024
Windows 10 Enterprise LTSC 2021
Windows Server 2022 SERVERDATACENTER
Windows Server 2025 SERVERDATACENTER
```

**如何查看 ISO 包含哪些映像：**
```bash
# 用 DISM（Windows）
dism /Get-ImageInfo /ImageFile:install.wim

# 用 wimlib（Linux）
wiminfo install.wim
```

---

#### 13. `--boot-wim:`
**用途：** 指定自定义的 `boot.wim` 文件路径（高级用法，极少用）

**参数值：** 本地文件路径或 HTTP 链接

通常不需要指定，脚本会从 ISO 自动提取。

---

#### 14. `--img:`
**用途：** 指定要 DD 的镜像文件 URL（DD 模式）

**参数值：** HTTP/HTTPS/磁力链接

**示例：**
```bash
bash reinstall.sh dd --img "https://example.com/debian.img.xz"
bash reinstall.sh dd --img "https://example.com/windows.vhd.gz"
```

**支持的格式：**
```
raw / raw.gz / raw.xz / raw.zst
vhd / vhd.gz / vhd.xz / vhd.zst
tar.gz / tar.xz / tar.zst (包含 raw 文件)
```

---

#### 15. `--lang:`
**用途：** 指定 Windows 的语言/区域（安装 Windows 时）

**参数值：** 语言代码（小写，用连字符）

**示例：**
```bash
bash reinstall.sh windows \
    --image-name "Windows 11 Pro" \
    --lang zh-cn  # 中文简体
```

**支持的语言：**
```
zh-cn  中文（中国）
zh-tw  中文（台湾）
zh-hk  中文（香港）
en-us  英语（美国）
en-gb  英语（英国）
ja-jp  日语
ko-kr  韩语
de-de  德语
fr-fr  法语
es-es  西班牙语
ru-ru  俄语
...（共 30+ 种）
```

完整列表见源码中的 `english()` 函数。

---

#### 16. `--passwd:` / `--password:`
**用途：** 设置系统密码（root 或 administrator）

**参数值：** 密码字符串

**示例：**
```bash
# Linux
bash reinstall.sh debian 12 --password "MySecurePass123"

# Windows
bash reinstall.sh windows --image-name "Windows 11 Pro" \
    --password "Admin@2024"

# DD 模式（仅用于 Alpine 临时环境的 SSH 密码）
bash reinstall.sh dd --img "debian.img.xz" \
    --password "TempPass"
```

**不指定时：**
- 脚本会提示输入密码
- 或生成随机密码并显示

**DD 模式的特殊说明：**
DD 镜像本身的密码不会被修改，`--password` 只用于 Alpine 临时环境的 SSH 登录。

---

#### 17. `--ssh-port:`
**用途：** 设置 SSH 端口

**双重作用：**
1. **Alpine 临时环境**的 SSH 端口（安装期间查看日志）
2. **最终系统**的 SSH 端口（写入 sshd_config）

**参数值：** 端口号（1-65535）

**示例：**
```bash
bash reinstall.sh debian 12 --ssh-port 12345
```

**效果：**
- 安装期间：`ssh root@<IP> -p 12345` 连接 Alpine 查看日志
- 安装完成：新系统的 sshd 监听 12345 端口

---

#### 18. `--ssh-key:` / `--public-key:`
**用途：** 设置 SSH 公钥登录（代替密码）

**参数值：** 可以是多种格式
- 公钥字符串：`"ssh-rsa AAAAB3NzaC1yc2EA..."`
- 本地文件路径：`/root/.ssh/id_rsa.pub`
- HTTP 链接：`https://example.com/my_key.pub`
- GitHub 用户：`github:username`
- GitLab 用户：`gitlab:username`

**示例：**
```bash
# 直接传公钥
bash reinstall.sh debian 12 \
    --ssh-key "ssh-rsa AAAAB3NzaC1yc2EAAAADAQAB..."

# 从本地文件读取
bash reinstall.sh debian 12 \
    --ssh-key /root/.ssh/id_rsa.pub

# 从 GitHub 拉取（会自动访问 https://github.com/username.keys）
bash reinstall.sh debian 12 \
    --ssh-key github:torvalds

# 从 URL 下载
bash reinstall.sh debian 12 \
    --ssh-key https://example.com/authorized_keys
```

**使用公钥时：**
- 密码会被置空（无密码登录）
- 只能用 SSH 密钥登录

---

#### 19. `--rdp-port:`
**用途：** 修改 Windows 远程桌面端口（仅 Windows）

**参数值：** 端口号（默认 3389）

**示例：**
```bash
bash reinstall.sh windows --image-name "Windows Server 2022 SERVERDATACENTER" \
    --rdp-port 8080
```

**工作原理：**
脚本会在 Windows 首次启动时修改注册表：
```powershell
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' -Name PortNumber -Value 8080
Restart-Service TermService -Force
```

---

#### 20. `--web-port:` / `--http-port:`
**用途：** 修改 Alpine 临时环境的 HTTP 日志查看端口

**参数值：** 端口号（默认 80）

**示例：**
```bash
bash reinstall.sh debian 12 --web-port 8080
```

**效果：**
安装期间可以通过浏览器访问：
```
http://<你的IP>:8080
```
实时查看安装日志（用 nginx 提供静态页面）。

**适用场景：**
- 80 端口被占用
- 防火墙只开放特定端口

---

#### 21. `--frpc-conf:` / `--frpc-config:` / `--frpc-toml:`
**用途：** 添加 frpc 内网穿透配置（用于无公网 IP 的机器）

**参数值：** 本地 `.toml` 文件路径或 HTTP 链接

**示例：**
```bash
bash reinstall.sh debian 12 \
    --frpc-toml /root/frpc.toml
```

**frpc.toml 示例：**
```toml
[common]
server_addr = frp.example.com
server_port = 7000
token = your_token

[ssh]
type = tcp
local_ip = 127.0.0.1
local_port = 22
remote_port = 12345
```

**工作原理：**
- 脚本会在 Alpine 临时环境启动 frpc，让你通过公网访问
- 新系统也会配置 frpc 开机自启
- 适合无公网 IP 的内网机器

---

#### 22. `--commit:`
**用途：** 指定使用旧版本的脚本（从 GitHub commit）

**参数值：** Git commit ID

**示例：**
```bash
bash reinstall.sh debian 12 --commit a1b2c3d4
```

**适用场景：**
- 最新版有 bug，想回退到稳定版
- 测试特定版本的功能

**原理：**
脚本会从 GitHub 下载指定 commit 的 `trans.sh` 等文件。

---

#### 23. `--force-boot-mode:`
**用途：** 强制指定引导模式（BIOS 或 EFI）

**参数值：** `bios` 或 `efi`

**示例：**
```bash
# 强制 BIOS 引导（MBR 分区表）
bash reinstall.sh windows --image-name "Windows 11 Pro" \
    --force-boot-mode bios

# 强制 EFI 引导（GPT 分区表）
bash reinstall.sh windows --image-name "Windows 11 Pro" \
    --force-boot-mode efi
```

**适用场景：**
- GCP 安装 Windows Server 2022 时需要 BIOS 模式（EFI 会反复重启）
- 某些老硬件只支持 BIOS

**不指定时：**
脚本自动检测机器的引导模式（检查 `/sys/firmware/efi` 是否存在）。

---

#### 24. `--force-old-windows-setup:`
**用途：** 强制使用老版 Windows 安装方式（兼容性模式）

**适用场景：**
- 某些老 ISO（Windows 7/Server 2008 R2）用新方法安装失败
- 测试向后兼容性

**效果：**
使用传统的 `setup.exe` 安装流程，而不是直接 `wimapply`。

---

## 完整示例

### 示例 1：安装 Debian 12（最常用）
```bash
bash reinstall.sh debian 12 \
    --password "MyPassword" \
    --ssh-port 22222
```

### 示例 2：DD Windows 镜像
```bash
bash reinstall.sh dd \
    --img "https://example.com/win11.vhd.xz" \
    --password "ViewLogs" \
    --rdp-port 3389 \
    --allow-ping
```

### 示例 3：安装 Windows Server（从 ISO）
```bash
bash reinstall.sh windows \
    --image-name "Windows Server 2025 SERVERDATACENTER" \
    --lang en-us \
    --password "Admin@2025" \
    --rdp-port 8080 \
    --allow-ping \
    --ssh-port 12345
```

### 示例 4：Ubuntu 云镜像快速安装
```bash
bash reinstall.sh ubuntu 24.04 --ci --minimal \
    --ssh-key github:yourusername
```

### 示例 5：调试模式（安装后不重启）
```bash
bash reinstall.sh debian 12 \
    --hold 2 \
    --debug \
    --ssh-port 12345
```

---

## 参数优先级

当多个参数冲突时：
1. `--ssh-key` 存在时，`--password` 无效（密码置空）
2. `--force-boot-mode` 覆盖自动检测
3. `--iso` 优先于自动查找
4. 后面的参数覆盖前面的（如果重复指定）

---

希望这份详解能帮你完全理解所有参数的用途！