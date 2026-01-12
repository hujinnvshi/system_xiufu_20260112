# SystemD 和 D-Bus 服务故障排查指南

**问题日期**: 2026-01-12
**系统**: Ubuntu 18.04.6 LTS (ARM64)
**问题主机**: 172.16.196.203

---

## 问题现象

```bash
# systemctl 命令完全失效
$ systemctl status
Failed to connect to bus: Connection refused

$ systemctl --failed
Failed to connect to bus: Connection refused

$ systemctl list-units
Failed to connect to bus: Connection refused
```

**影响范围**:
- 无法通过 systemctl 管理服务
- SSH 登录时出现 pam_systemd 错误
- 所有依赖 systemd 的服务管理功能失效

---

## 问题根因分析

### 1. 初步诊断流程

#### 步骤 1: 检查 systemd 进程状态
```bash
# PID 1 是否为 systemd
$ ps -p 1 -f
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 Jan09 ?        00:00:04 /sbin/init

$ readlink -f /sbin/init
/lib/systemd/systemd  # ✅ 确认是 systemd

$ cat /proc/1/comm
systemd  # ✅ 进程名正确
```

**关键发现**: systemd 进程存在但功能异常

#### 步骤 2: 检查 D-Bus 连接
```bash
$ ps aux | grep dbus-daemon
# 空 - dbus-daemon 没有运行！

$ ls -la /run/dbus/
total 0
srw-rw-rw-  1 root root   0 Jan  9 06:42 system_bus_socket
# 套接字存在但没有进程监听
```

**关键发现**: D-Bus 守护进程未运行

#### 步骤 3: 尝试启动 D-Bus
```bash
$ mkdir -p /var/run/dbus && dbus-daemon --system --fork
$ ps aux | grep dbus-daemon
message+ 17804  0.0  0.0   6728  2444 ?        Ss   03:39   0:00 dbus-daemon --system --fork
# ✅ dbus-daemon 启动成功
```

#### 步骤 4: 测试 systemctl
```bash
$ systemctl status
Failed to read server status: The permission of the setuid helper is not correct
```

**新的错误**: "The permission of the setuid helper is not correct"

---

### 2. 深度诊断

#### 检查 dbus-daemon-launch-helper 权限
```bash
$ ls -la /usr/lib/dbus-1.0/dbus-daemon-launch-helper
-rwxrwxr-x 1 root root 42848 Dec 30 17:53 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
# ❌ 错误：权限是 0777，应该是 4755
```

**正确权限应该是**:
```bash
-rwsr-xr-x 1 root root 42848 Dec 30 17:53 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
4755 root root /usr/lib/dbus-1.0/dbus-daemon-launch-helper
```

#### 检查 systemd 文件描述符
```bash
$ ls -la /proc/1/fd
total 0
dr-x------ 2 root root  0 Jan  9 06:42 .
dr-xr-xr-x 9 root root  0 Jan  9 06:42 ..
lrwx------ 1 root root 64 Jan  9 06:42 0 -> /dev/null
lrwx------ 1 root root 64 Jan  9 06:42 1 -> /dev/null
lrwx------ 1 root root 64 Jan  9 06:42 2 -> /dev/null
# ❌ 只有 stdin/stdout/stderr，没有套接字
```

**正常情况下应该看到**:
- fd 3: `/run/systemd/private` (systemd 私有套接字)
- fd 4: D-Bus 系统总线连接
- 其他通信套接字

#### 检查内核日志
```bash
$ dmesg | grep -i systemd | tail -10
systemd-journald[2246]: Failed to send stream file descriptor to service manager: Transport endpoint is not connected
```

**关键发现**: systemd-journald 无法连接到服务管理器

---

## 解决方案

### 修复步骤

#### 1. 修复 dbus-daemon-launch-helper 权限
```bash
chmod 4755 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
chown root:root /usr/lib/dbus-1.0/dbus-daemon-launch-helper

# 验证
$ ls -la /usr/lib/dbus-1.0/dbus-daemon-launch-helper
-rwsr-xr-x 1 root root 42848 Dec 30 17:53 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
```

#### 2. 启动 D-Bus 守护进程
```bash
# 清理旧的 pid 文件
rm -f /var/run/dbus/pid

# 启动 dbus-daemon
dbus-daemon --system --fork

# 验证
$ ps aux | grep dbus-daemon
message+ 18836  0.1  0.0   6728  2576 ?        Ss   03:42   0:00 dbus-daemon --system --fork
```

#### 3. 重启系统（关键步骤）
```bash
# 由于 systemd (PID 1) 没有正常初始化，必须重启
# systemctl reboot 不可用，使用 SysRq 强制重启

echo 1 > /proc/sys/kernel/sysrq
echo b > /proc/sysrq-trigger

# 或者直接向 init 发送信号（如果可用）
kill -TERM 1
```

**为什么需要重启**:
- systemd (PID 1) 在启动时未能正确初始化 D-Bus 套接字
- 这是系统启动时的根本性错误，无法在运行时修复
- 重启后 systemd 会重新初始化，自动建立 D-Bus 连接

#### 4. 重启后验证
```bash
# 等待系统启动（约 30-60 秒）
$ systemctl --version
systemd 237

$ systemctl status
● AllstarOS
    State: starting
     Jobs: 0 queued
   Failed: 0 units

$ systemctl list-units --type=service --state=running | wc -l
31
```

---

## 常见问题处理

### 问题 1: systemd-tmpfiles-setup.service 失败

**现象**:
```bash
$ systemctl --failed
● systemd-tmpfiles-setup.service       loaded failed failed Create Volatile Files and Directories
```

**日志**:
```
systemd-tmpfiles[4611]: Unsafe symlinks encountered in /var/log/journal, refusing.
```

**根因**: `/var/log` 是符号链接 → `/preserve/logs/var_log`

**解决方案**:
```bash
# 1. 创建实际目录的正确结构
mkdir -p /preserve/logs/var_log/journal/<machine-id>
chown root:systemd-journal /preserve/logs/var_log/journal
chmod 2755 /preserve/logs/var_log/journal

chown root:root /preserve/logs/var_log/journal/<machine-id>
chmod 0755 /preserve/logs/var_log/journal/<machine-id>

# 2. 创建 tmpfiles 配置
cat > /etc/tmpfiles.d/journal.conf << 'EOF'
# Explicitly allow /var/log/journal with the symlink to /preserve
L+ /var/log/journal - - - - /preserve/logs/var_log/journal
EOF

# 3. 测试
systemd-tmpfiles --create --exclude-prefix=/dev

# 4. 重置失败状态
systemctl reset-failed systemd-tmpfiles-setup.service
```

---

## 经验总结

### 关键经验

#### 1. 诊断思路
```
systemctl 失败
  ↓
检查 D-Bus 连接
  ↓
检查 dbus-daemon 是否运行
  ↓
检查 dbus-daemon-launch-helper 权限
  ↓
检查 systemd (PID 1) 文件描述符
  ↓
确认需要重启系统
```

#### 2. 重要检查点

**检查清单**:
- [ ] PID 1 是否为 systemd (`ps -p 1`)
- [ ] D-Bus 守护进程是否运行 (`ps aux | grep dbus-daemon`)
- [ ] dbus-daemon-launch-helper 权限是否为 4755
- [ ] systemd (PID 1) 是否有打开的套接字 (`ls -la /proc/1/fd`)
- [ ] `/run/systemd/private` 是否存在且有进程监听
- [ ] 系统日志是否有错误 (`journalctl -p err -b`)

#### 3. 命令速查表

```bash
# 检查 systemd 版本
systemctl --version

# 检查 systemd 进程
ps -p 1 -f
cat /proc/1/comm
cat /proc/1/cmdline

# 检查 D-Bus
ps aux | grep dbus-daemon
ls -la /run/dbus/
dbus-send --system --dest=org.freedesktop.systemd1 ...

# 检查权限
ls -la /usr/lib/dbus-1.0/dbus-daemon-launch-helper
stat -c '%a %U %G %n' /usr/lib/dbus-1.0/dbus-daemon-launch-helper

# 检查文件描述符
ls -la /proc/1/fd
lsof | grep /run/systemd/private

# 检查系统日志
journalctl -p err -b
dmesg | grep -i systemd

# 强制重启（当 systemctl 不可用时）
echo 1 > /proc/sys/kernel/sysrq
echo b > /proc/sysrq-trigger
```

#### 4. 故障类型判断

| 症状 | 根因 | 解决方案 |
|------|------|----------|
| `Failed to connect to bus: Connection refused` | dbus-daemon 未运行 | 启动 dbus-daemon |
| `The permission of the setuid helper is not correct` | 权限错误 | chmod 4755 |
| PID 1 只有 3 个 fd | systemd 未正常初始化 | 重启系统 |
| `Unsafe symlinks encountered` | 符号链接问题 | 创建 tmpfiles 配置 |

---

## 预防措施

### 1. 权限监控

建议定期检查关键文件的权限：
```bash
# 添加到 cron 或监控脚本
check_file_permissions() {
    local file="/usr/lib/dbus-1.0/dbus-daemon-launch-helper"
    local expected_perms="4755"

    local actual_perms=$(stat -c '%a' "$file")
    if [ "$actual_perms" != "$expected_perms" ]; then
        echo "WARNING: $file has incorrect permissions: $actual_perms"
        # 发送告警
    fi
}
```

### 2. 系统启动验证

在系统启动后自动验证关键服务：
```bash
# /etc/systemd/system/system-health-check.service
[Unit]
Description=System Health Check
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/system-health-check.sh

[Install]
WantedBy=multi-user.target
```

### 3. 日志监控

监控关键错误日志：
```bash
# 监控 journal 和 syslog
journalctl -p err -b -f | grep -E '(systemd|dbus|systemctl)'
```

---

## 相关资源

### 手册页
```bash
man systemd
man dbus-daemon
man systemctl
man systemd-tmpfiles
man tmpfiles.d
```

### 配置文件
- `/etc/systemd/system.conf` - systemd 配置
- `/etc/dbus-1/system.conf` - D-Bus 系统配置
- `/usr/lib/tmpfiles.d/*.conf` - tmpfiles 配置
- `/etc/tmpfiles.d/*.conf` - 本地 tmpfiles 覆盖配置

### 系统目录
- `/run/systemd/` - systemd 运行时目录
- `/run/dbus/` - D-Bus 运行时目录
- `/var/log/journal/` - systemd 日志目录

---

## 附录

### A. 完整诊断脚本

```bash
#!/bin/bash
# systemd-dbus-diagnostic.sh

echo "=== Systemd/D-Bus Diagnostic Tool ==="
echo

# 1. Check PID 1
echo "1. Checking PID 1..."
pid1_comm=$(cat /proc/1/comm)
echo "   PID 1 process: $pid1_comm"
if [ "$pid1_comm" != "systemd" ]; then
    echo "   WARNING: PID 1 is not systemd!"
fi
echo

# 2. Check D-Bus
echo "2. Checking D-Bus daemon..."
dbus_count=$(ps aux | grep -c 'dbus-daemon.*--system' || true)
echo "   D-Bus daemon processes: $dbus_count"
if [ "$dbus_count" -eq 0 ]; then
    echo "   WARNING: dbus-daemon is not running!"
fi
echo

# 3. Check helper permissions
echo "3. Checking dbus-daemon-launch-helper permissions..."
helper_perms=$(stat -c '%a' /usr/lib/dbus-1.0/dbus-daemon-launch-helper 2>/dev/null)
echo "   Current permissions: $helper_perms"
if [ "$helper_perms" != "4755" ]; then
    echo "   WARNING: Incorrect permissions! Should be 4755"
fi
echo

# 4. Check systemd file descriptors
echo "4. Checking systemd (PID 1) file descriptors..."
fd_count=$(ls /proc/1/fd 2>/dev/null | wc -l)
echo "   Open file descriptors: $fd_count"
if [ "$fd_count" -le 3 ]; then
    echo "   WARNING: systemd has too few file descriptors!"
fi
echo

# 5. Test systemctl
echo "5. Testing systemctl..."
if systemctl --version >/dev/null 2>&1; then
    echo "   systemctl: WORKING"
else
    echo "   systemctl: FAILED"
    echo "   Error: $(systemctl --version 2>&1 | head -1)"
fi
echo

echo "=== Diagnostic Complete ==="
```

### B. 一键修复脚本（谨慎使用）

```bash
#!/bin/bash
# WARNING: This script will reboot the system!
# Only use after proper diagnosis!

set -e

echo "=== Systemd/D-Bus Repair Script ==="
echo

# 1. Fix helper permissions
echo "1. Fixing dbus-daemon-launch-helper permissions..."
chmod 4755 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
chown root:root /usr/lib/dbus-1.0/dbus-daemon-launch-helper

# 2. Start D-Bus
echo "2. Starting D-Bus daemon..."
rm -f /var/run/dbus/pid
dbus-daemon --system --fork

# 3. Verify
echo "3. Verifying repairs..."
if [ "$(stat -c '%a' /usr/lib/dbus-1.0/dbus-daemon-launch-helper)" = "4755" ]; then
    echo "   ✓ Helper permissions fixed"
else
    echo "   ✗ Failed to fix permissions"
    exit 1
fi

if pgrep -f 'dbus-daemon.*--system' >/dev/null; then
    echo "   ✓ D-Bus daemon running"
else
    echo "   ✗ Failed to start D-Bus"
    exit 1
fi

# 4. Reboot
echo
echo "4. System reboot required to complete the repair."
echo "   Press Ctrl+C to cancel, or wait 5 seconds for automatic reboot..."
sleep 5

echo "Rebooting..."
echo 1 > /proc/sys/kernel/sysrq
echo b > /proc/sysrq-trigger
```

---

**文档版本**: 1.0
**最后更新**: 2026-01-12
**作者**: Claude Code
**许可**: MIT
