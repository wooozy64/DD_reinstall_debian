# SSH配置

## SSH_CONFIG
### StrictHostKeyChecking
`StrictHostKeyChecking` 控制的是：**当连接一台服务器时，如何处理它的主机指纹验证。**
#### 可选值
**`ask`（默认值）**
- 新主机：提示 `yes/no/fingerprint`，手动确认后保存
- 指纹变了：警告并拒绝连接，需要手动处理
**`accept-new`**
- 新主机：自动接受并保存，不提示
- 指纹变了：拒绝连接（比ask更安全）
**`yes`**
- 新主机：直接拒绝，不提示，不保存
- 指纹变了：拒绝连接
- 必须提前把指纹加入 `known_hosts` 才能连
**`no`**
- 新主机：自动接受，不保存
- 指纹变了：自动接受，不警告
- 最不安全，完全不验证
**`off`**
- 和 `no` 基本相同
#### 安全级别排序
```
yes > accept-new > ask > no/off
```
日常推荐 `accept-new`，生产自动化脚本用 `yes`（配合提前导入指纹），测试环境图方便用 `no`。

### BatchMode 
作用：控制客户端是否允许交互式输入。
`yes`：禁止一切交互式提示；需要密码时直接失败，不等待输入；适合脚本、自动化任务；
`no`（默认）：允许弹出密码提示；允许弹出指纹确认；适合人工操作；


## SSHD_CONFIG
### PermitOpen
```bash
PermitOpen none
```
这会 **明确禁止任何 `direct-tcpip` 转发**，所以即使 `AllowTcpForwarding yes` 也会被挡住，结果就是 `Administratively prohibited`。
**改法二选一**：
1. 放开所有目标（最简单）  
```sshconfig
PermitOpen any
```
2. 只放开特定目标（更安全）  
```sshconfig
PermitOpen 目标IP:目标端口
```
改完重启 `sshd`。
另外确认两点：
- 有没有 `Match User` / `Match Address` 覆盖了 `PermitOpen`  
- 你连接的那个“网关”用户是否命中更严格的 Match 段

### MaxSessions
它的核心作用是**限制单个 TCP 连接（Multiplexing）中所能打开的并行会话（Session）数量**。

需要明确的是，它**不限制**你同时开启多少个独立的 SSH 命令行窗口（那是`MaxStartups`的活儿），而是限制“连接复用”下的并发数。
**举例**：如果设置为1，MobaXterm连接进入服务器后，SFTP服务就无法使用。

### AuthenticationMethods
控制允许的认证方式和组合逻辑。
目前配置是`AuthenticationMethods publickey password`，如果设置成`AuthenticationMethods publickey，password`，就无法登陆服务器，AI都回复空格和逗号都是AND关系，但是实际测试下来，空格是OR关系，逗号是AND关系。
#### 基本认证方式

| 值 | 说明 |
|---|---|
| `password` | 密码认证 |
| `publickey` | 公钥认证 |
| `keyboard-interactive` | 键盘交互（PAM等） |
| `gssapi-with-mic` | Kerberos认证 |
| `hostbased` | 主机信任认证 |

#### 组合语法
**逗号 = AND（必须依次全部通过）**
```
AuthenticationMethods publickey,password
# 先公钥，再密码，两步都要
```
**空格 = OR（满足其一即可）**
```
AuthenticationMethods publickey password
# 公钥或密码，任一通过即可
```
**混合使用**
```
AuthenticationMethods publickey,password publickey
# 方案1：公钥+密码 OR 方案2：只用公钥
```
#### 特殊值
**`any`（默认值）**
- 不设置 `AuthenticationMethods` 时的默认行为
- 按照各自开关（`PubkeyAuthentication`、`PasswordAuthentication` 等）决定允许哪些方式
#### 注意
`AuthenticationMethods` 里列出的方式，对应的开关必须也是开启的，否则冲突会导致行为异常，就是你之前遇到的情况（`PubkeyAuthentication no` 但却列了 `publickey`）。



