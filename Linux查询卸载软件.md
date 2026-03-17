# Linux查询卸载软件

## 查询安装的软件
```bash
dpkg -l | grep exim
```

``` bash
dpkg -l | grep -E 'telnet|rsh|rlogin|ftp|tftp|xinetd|exim4|avahi|cups|bluez|modemmanager|ppp|wpasupplicant|snapd'
```

### 基本上用不上的软件
- cups*  -- 打印系统
- bluez  -- 蓝牙
- avahi-daemon -- 局域网广播服务
- rpcbind  -- NFS 用的
- exim4*     -- 邮件系统
- postfix*   -- 邮件系统
- bsd-mailx   -- 邮件系统
- snapd   -- Snap
- vsftpd  -- ftp 服务
- proftpd -- ftp 服务

## 查看当前“手动安装”的包
```bash
apt-mark showmanual
```

## 卸载软件
``` bash
apt purge exim4*
```

## 删除孤立依赖
``` bash
sudo apt autoremove --purge
```

## 查正在运行的服务
```bash
systemctl list-unit-files --type=service --state=enabled
```

## 不要乱删的核心组件

这些别动：
``` bash
systemd
bash
coreutils
libc6
openssh-server
apt
```
删了可能直接崩。