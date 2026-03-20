# SSH Resolution: 我和 AI 如何协作，跨三台设备打通 macOS -> VPS -> Windows

这不是一篇只给结果的教程，而是一份**可复刻的协作过程记录**。

目标读者：
- 你也有一台 Windows、一台 macOS、一台云服务器
- Windows 和 mac 在同一网络里，但 SSH 直连总是失败/不稳定
- 你想让 AI 协助你，同时操作多台机器，把链路搭起来

> 注意：本文会保留真实拓扑和真实命令风格，但**不会公开真实密码**。所有密码、密钥请用占位符替代。

---

## 1. 场景

我的环境里有三台设备：

1. **Windows**
   - 当前用户：`seqi`
   - OpenSSH Server 已安装
2. **macOS**
   - 用户：`seqi`
3. **腾讯云 VPS**
   - 用户：`ubuntu`
   - 公网 IP：`111.229.167.172`

需求不是让 Windows 连 mac，而是：

- **让 macOS 稳定登录 Windows**
- 因为 Windows 和 mac 即使在同一网络环境中，也始终无法正常 SSH 直连
- 所以决定借助 **VPS 中转**

最终目标是：

```text
macOS -> VPS -> Windows
```

---

## 2. 我和 AI 是怎么协作的

这次不是“AI 发一段教程，我自己慢慢试”，而是更接近**协同运维**：

### 我提供的信息

我把三端的关键信息给 AI：

- VPS 的 IP、用户名、密码
- mac 的 IP、用户名、密码
- 目标方向：**mac 连 Windows**

一开始我给过 AI 这些信息：

```text
VPS: 111.229.167.172
VPS user: ubuntu
mac IP: 10.16.207.63
mac user: seqi
```

密码也给了，但在公开项目里必须写成占位符：

```text
VPS password: <VPS_PASSWORD>
mac password: <MAC_PASSWORD>
```

### AI 做的事情

AI 不只是告诉我“你运行这个命令”。它实际做了这些工作：

1. 检查这台 Windows 的 SSH 状态
2. 尝试连接 VPS 和 mac
3. 在后台用脚本同时测试三端连通性
4. 自动发现配置差异
5. 自动修正方案
6. 最后把临时验证方案升级成长期稳定方案

### 协作关键点

这种协作模式要能跑通，通常需要满足：

1. AI 能操作当前机器
2. 用户愿意提供必要凭据
3. 用户允许 AI 在多台设备间做只读/写入配置
4. 用户理解：**真实密码不要进文档，不要进 Git 仓库**

---

## 3. 一开始的错误假设

最开始很容易以为这是个简单问题：

- 在 Windows 上开一个反向隧道到 VPS
- 在 mac 上 SSH 到 VPS 的某个端口
- 结束

但实际排查后，事情没那么直。

### 先踩到的坑

#### 坑 1：后台 PTY SSH 在当前环境不稳定

AI 先尝试直接用后台终端登录 VPS 和 mac，但遇到了：

```text
ssh_dispatch_run_fatal: Connection ... Invalid argument
```

这说明不是密码错，而是 **Windows OpenSSH + 后台 PTY** 这条路在当前环境里不稳定。

#### 坑 2：最初以为 Windows SSH 在 22 端口

后面实际读到 Windows 的配置文件后才发现：

```text
C:\ProgramData\ssh\sshd_config
```

关键内容是：

```text
Port 1022
PasswordAuthentication yes
```

也就是说：

- 这台 Windows 的 SSH **根本不在 22 端口**
- 而是在 **1022**

这一步非常关键。很多后续问题，都是因为目标端口先假设错了。

#### 坑 3：mac 用户名一开始给错了

最开始尝试的是：

```text
root@10.16.207.63
```

但实际不可用。后来修正成：

```text
seqi@10.16.207.63
```

才登录成功。

---

## 4. AI 是如何一步步排出来的

### 第一步：检查 Windows 本机 SSH 状态

先确认：

- 当前用户名
- sshd 是否运行
- SSH 客户端是否可用

Windows 上典型检查：

```powershell
whoami
Get-Service sshd | Format-List Name,Status,StartType
ssh -V
Get-Content C:\ProgramData\ssh\sshd_config
```

重点发现：

- `sshd` 在运行
- 监听端口是 `1022`
- 用户 `seqi` 属于管理员组

### 第二步：改用脚本化 SSH，而不是依赖交互终端

由于 PTY 路线不稳定，AI 改用 Python + Paramiko 做自动探测：

- 测试能否登录 VPS
- 测试能否登录 mac
- 测试 Windows 本机 SSH banner
- 测试 VPS 是否允许远程端口转发

这种方式的优势是：

- 可以直接在后台建立连接
- 可以自动收集错误信息
- 不需要用户盯着终端一步步输

### 第三步：确认 VPS 侧条件

AI 在 VPS 上探测到：

```text
gatewayports yes
allowtcpforwarding yes
```

理论上这说明：

- VPS 允许转发
- 允许外部访问反向转发端口

但实际运行时，**外部访问远程转发端口的行为仍然不稳定**。

### 第四步：不要强依赖“直接访问 VPS 反向端口”

这是这次排障里最重要的策略变化。

一开始的思路是：

```text
Windows --(ssh -R)-> VPS:42022
mac ------ssh to------> VPS:42022
```

理论上可行，但实测中：

- VPS 端口能开
- 连接也能打到 VPS
- 但最终连接经常被关闭

所以 AI 改成了一个更稳的结构：

```text
Windows --(反向隧道)-> VPS localhost:42022
mac --(本地转发)-> localhost:42024 -> VPS localhost:42022
mac ssh 到自己的 127.0.0.1:42024
```

也就是双隧道：

1. **Windows -> VPS**：反向隧道
2. **mac -> VPS**：本地转发
3. **mac -> 自己本地端口**：最终 SSH 入口

这个结构绕开了“VPS 对外暴露反向端口行为不稳定”的问题。

---

## 5. 另一个关键坑：Windows 管理员组的 authorized_keys 特殊路径

Windows OpenSSH 和 Linux/mac 不完全一样。

在这台机器上，`sshd_config` 里有：

```text
Match Group administrators
       AuthorizedKeysFile __PROGRAMDATA__/ssh/administrators_authorized_keys
```

这意味着：

如果用户 `seqi` 属于管理员组，那么公钥认证**不读**：

```text
C:\Users\seqi\.ssh\authorized_keys
```

而是读：

```text
C:\ProgramData\ssh\administrators_authorized_keys
```

所以 AI 之前把公钥加到用户目录里时，链路虽然通了，但认证仍然失败。

修正之后才真正成功。

---

## 6. 最终可复刻的方案

### 拓扑

```text
macOS localhost:42024
  -> SSH LocalForward 到 VPS localhost:42022
  -> SSH ReverseForward 到 Windows localhost:1022
  -> Windows OpenSSH Server
```

---

## 7. 复刻时需要准备什么

你至少要有：

### Windows

- Windows 用户名，如：`seqi`
- OpenSSH Server 已启用
- 知道 `sshd` 实际监听端口（不一定是 22）

### macOS

- mac 用户名，如：`seqi`
- 可 SSH 登录或允许 AI 在本机操作

### VPS

- 公网 IP 或域名
- SSH 用户名
- SSH 密码或密钥

### 凭据模板

文档里不要写真实密码，应该用：

```text
VPS_HOST=111.229.167.172
VPS_USER=ubuntu
VPS_PASSWORD=<VPS_PASSWORD>

MAC_HOST=10.16.207.63
MAC_USER=seqi
MAC_PASSWORD=<MAC_PASSWORD>

WINDOWS_USER=seqi
WINDOWS_SSH_PORT=1022
```

---

## 8. Windows 端配置

### 8.1 查看 sshd 配置

```powershell
Get-Content C:\ProgramData\ssh\sshd_config
```

确认类似：

```text
Port 1022
PasswordAuthentication yes

Match Group administrators
       AuthorizedKeysFile __PROGRAMDATA__/ssh/administrators_authorized_keys
```

### 8.2 放置桥接私钥

路径：

```text
C:\Users\<WINDOWS_USER>\.ssh\openclaw_bridge_key
```

### 8.3 将公钥写入正确位置

如果目标用户属于管理员组：

```text
C:\ProgramData\ssh\administrators_authorized_keys
```

### 8.4 反向隧道脚本

`C:\Users\<WINDOWS_USER>\.ssh\openclaw_reverse_tunnel.ps1`

```powershell
while ($true) {
  & ssh -i C:\Users\seqi\.ssh\openclaw_bridge_key -o StrictHostKeyChecking=accept-new -o ExitOnForwardFailure=yes -o ServerAliveInterval=30 -o ServerAliveCountMax=3 -N -R 42022:localhost:1022 ubuntu@111.229.167.172
  Start-Sleep -Seconds 5
}
```

### 8.5 开机/登录自动启动

```powershell
schtasks /Create /TN OpenClaw-Windows-Bridge /SC ONLOGON /RU seqi /TR "powershell -ExecutionPolicy Bypass -WindowStyle Hidden -File C:\Users\seqi\.ssh\openclaw_reverse_tunnel.ps1" /F
```

---

## 9. macOS 端配置

### 9.1 放置桥接私钥

```bash
~/.ssh/openclaw_bridge_key
chmod 600 ~/.ssh/openclaw_bridge_key
```

### 9.2 创建本地转发脚本

`~/.ssh/openclaw_mac_to_vps.sh`

```bash
#!/bin/bash
while true; do
  /usr/bin/ssh -f -i /Users/seqi/.ssh/openclaw_bridge_key -o StrictHostKeyChecking=no -o ExitOnForwardFailure=yes -o ServerAliveInterval=30 -o ServerAliveCountMax=3 -N -L 42024:localhost:42022 ubuntu@111.229.167.172 || true
  sleep 5
  if lsof -nP -iTCP:42024 -sTCP:LISTEN >/dev/null 2>&1; then
    sleep 30
  fi
done
```

```bash
chmod 700 ~/.ssh/openclaw_mac_to_vps.sh
```

### 9.3 配置 LaunchAgent

`~/Library/LaunchAgents/com.openclaw.windows-bridge.plist`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key><string>com.openclaw.windows-bridge</string>
  <key>ProgramArguments</key>
  <array>
    <string>/bin/bash</string>
    <string>/Users/seqi/.ssh/openclaw_mac_to_vps.sh</string>
  </array>
  <key>RunAtLoad</key><true/>
  <key>KeepAlive</key><true/>
  <key>StandardOutPath</key><string>/tmp/com.openclaw.windows-bridge.out</string>
  <key>StandardErrorPath</key><string>/tmp/com.openclaw.windows-bridge.err</string>
</dict>
</plist>
```

加载：

```bash
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.openclaw.windows-bridge.plist
launchctl kickstart -k gui/$(id -u)/com.openclaw.windows-bridge
```

### 9.4 配置快捷登录

在 `~/.ssh/config` 中加入：

```sshconfig
Host windows-via-vps
  HostName 127.0.0.1
  Port 42024
  User seqi
  IdentityFile /Users/seqi/.ssh/openclaw_bridge_key
  StrictHostKeyChecking no
  ServerAliveInterval 30
```

最终登录命令：

```bash
ssh windows-via-vps
```

---

## 10. 验证方式

### 在 mac 上验证本地端口监听

```bash
lsof -nP -iTCP:42024 -sTCP:LISTEN
```

### 验证 SSH 登录

```bash
ssh windows-via-vps
```

或：

```bash
ssh -vvv -p 42024 seqi@127.0.0.1 exit
```

成功标志：

- 看到远端 banner 是 `OpenSSH_for_Windows`
- 公钥认证成功
- `Exit status 0`

---

## 11. 这个协作流程里，AI 具体扮演什么角色

如果你也想复刻“人类 + AI + 三设备协同”这种方式，可以参考这次的角色分工：

### 用户负责

- 提供三端信息
- 提供必要凭据
- 授权 AI 做配置修改
- 最后确认长期方案是否接受

### AI 负责

- 自动发现错误假设
- 自动收集日志
- 自动测试多端链路
- 自动修改配置
- 把临时验证升级为长期方案
- 清理带密码的临时脚本
- 把最终方案和协作过程整理成文档

最有价值的部分，不是“AI 会敲命令”，而是：

- AI 会**不断修正自己的假设**
- AI 会根据日志把结构从单隧道改成双隧道
- AI 会把系统差异（例如 Windows 管理员组 authorized_keys）纳入最终方案

---

## 12. 风险与安全建议

### 当前风险

1. 同一把桥接私钥存在于 Windows 和 macOS
2. Windows 的 SSH (`1022`) 存在入站放行规则
3. Windows 还监听了其他服务，需要额外审视暴露面
4. 如果磁盘没加密，设备丢失风险更高

### 建议优化

1. **改成每一跳使用独立密钥**
2. 收紧 Windows 防火墙，仅允许必要来源访问 1022 / 3389
3. 检查 `3306` / `445` / 其他监听端口是否真的需要
4. 启用 BitLocker（如果业务允许）
5. 不要把真实密码、私钥、会话日志直接推到 GitHub

---

## 13. 回滚方法

### Windows

删除计划任务：

```powershell
schtasks /Delete /TN OpenClaw-Windows-Bridge /F
```

删除以下文件：

- `C:\Users\seqi\.ssh\openclaw_reverse_tunnel.ps1`
- `C:\Users\seqi\.ssh\openclaw_bridge_key`
- `C:\Users\seqi\.ssh\openclaw_bridge_key.pub`

并从以下文件中移除对应公钥：

- `C:\ProgramData\ssh\administrators_authorized_keys`

### macOS

卸载 LaunchAgent：

```bash
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/com.openclaw.windows-bridge.plist
```

删除文件：

- `~/.ssh/openclaw_mac_to_vps.sh`
- `~/.ssh/openclaw_bridge_key`
- `~/Library/LaunchAgents/com.openclaw.windows-bridge.plist`
- `~/.ssh/config` 里的 `windows-via-vps` 段

---

## 14. 总结

这次最值得复刻的不是某一条 SSH 命令，而是这套协作方法：

1. 先让 AI 拿到完整三端信息
2. 不要死盯最初方案
3. 用自动探测替代拍脑袋
4. 发现端口、账号、认证路径这些“隐藏差异”
5. 从临时打通，升级到长期稳定
6. 最后把过程写成别人能照抄的文档

最终成果是：

```bash
ssh windows-via-vps
```

别人要复刻，也可以沿着这篇文档，把自己的：

- Windows
- mac
- VPS
- 用户名
- 端口
- 密钥

替换进去完成部署。
