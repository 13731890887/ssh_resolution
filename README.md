# SSH Resolution

通过腾讯云 VPS 中转，让 macOS 稳定 SSH 到同一网络环境中的 Windows。

## 目标

实现：

- macOS -> VPS -> Windows
- 避开同网环境下直连失败的问题
- 支持长期稳定使用

## 最终使用方式

在 macOS 上执行：

```bash
ssh windows-via-vps
```

等价命令：

```bash
ssh -p 42024 seqi@127.0.0.1
```

## 拓扑

```text
macOS localhost:42024
  -> SSH LocalForward 到 VPS localhost:42022
  -> SSH ReverseForward 到 Windows localhost:1022
  -> Windows OpenSSH Server
```

## 关键发现

1. Windows 的 sshd 实际监听端口不是 22，而是 `1022`
2. Windows 的 `seqi` 属于管理员组，公钥认证需要写入：
   - `C:\ProgramData\ssh\administrators_authorized_keys`
3. 直接暴露 VPS 的远程转发端口访问行为不稳定时，可以改用双隧道：
   - Windows -> VPS 建反向隧道
   - mac -> VPS 建本地转发
   - mac 再连自己的 `127.0.0.1:42024`

## Windows 配置

### 1. 确认 sshd 配置

`C:\ProgramData\ssh\sshd_config`

关键项：

```text
Port 1022
PasswordAuthentication yes

Match Group administrators
       AuthorizedKeysFile __PROGRAMDATA__/ssh/administrators_authorized_keys
```

### 2. 放置桥接私钥

路径：

```text
C:\Users\seqi\.ssh\openclaw_bridge_key
```

### 3. 持续保持反向隧道

脚本：

`C:\Users\seqi\.ssh\openclaw_reverse_tunnel.ps1`

```powershell
while ($true) {
  & ssh -i C:\Users\seqi\.ssh\openclaw_bridge_key -o StrictHostKeyChecking=accept-new -o ExitOnForwardFailure=yes -o ServerAliveInterval=30 -o ServerAliveCountMax=3 -N -R 42022:localhost:1022 ubuntu@111.229.167.172
  Start-Sleep -Seconds 5
}
```

### 4. 设置开机/登录后自动启动

```powershell
schtasks /Create /TN OpenClaw-Windows-Bridge /SC ONLOGON /RU seqi /TR "powershell -ExecutionPolicy Bypass -WindowStyle Hidden -File C:\Users\seqi\.ssh\openclaw_reverse_tunnel.ps1" /F
```

## macOS 配置

### 1. 放置桥接私钥

路径：

```bash
~/.ssh/openclaw_bridge_key
chmod 600 ~/.ssh/openclaw_bridge_key
```

### 2. 创建本地转发脚本

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

### 3. 创建 LaunchAgent

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

### 4. SSH 别名

追加到 `~/.ssh/config`：

```sshconfig
Host windows-via-vps
  HostName 127.0.0.1
  Port 42024
  User seqi
  IdentityFile /Users/seqi/.ssh/openclaw_bridge_key
  StrictHostKeyChecking no
  ServerAliveInterval 30
```

## 验证方法

在 macOS 上运行：

```bash
ssh windows-via-vps
```

或：

```bash
ssh -vvv -p 42024 seqi@127.0.0.1 exit
```

成功标志：

- 能看到远端 banner 为 `OpenSSH_for_Windows`
- 公钥认证成功
- 退出码为 `0`

## 风险与建议

### 当前风险

1. 同一把桥接私钥存在于 Windows 和 macOS
2. Windows 的 SSH (`1022`) 对外监听，且存在入站规则
3. Windows 同时还开放了其他服务端口，需要额外审视暴露面
4. BitLocker 未开启时，设备丢失风险更高

### 建议改进

1. 改为每一跳使用独立密钥
2. 收紧 Windows 防火墙，仅允许必要来源访问 1022 / 3389
3. 检查 3306 / 445 / 其他监听端口是否真的需要
4. 评估启用 BitLocker

## 停用方法

### Windows

```powershell
schtasks /Delete /TN OpenClaw-Windows-Bridge /F
```

删除：

- `C:\Users\seqi\.ssh\openclaw_reverse_tunnel.ps1`
- `C:\Users\seqi\.ssh\openclaw_bridge_key*`
- `C:\ProgramData\ssh\administrators_authorized_keys` 中对应公钥

### macOS

```bash
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/com.openclaw.windows-bridge.plist
```

删除：

- `~/.ssh/openclaw_mac_to_vps.sh`
- `~/.ssh/openclaw_bridge_key`
- `~/Library/LaunchAgents/com.openclaw.windows-bridge.plist`
- `~/.ssh/config` 中的 `windows-via-vps` 条目
