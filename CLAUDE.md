# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is a system repair documentation repository (`system_xiufu_20260112`) containing diagnostic reports and troubleshooting guides for Linux system administration issues. The repository focuses on documenting resolved system problems with detailed analysis, root cause identification, and solution strategies.

## Repository Structure

This repository contains documentation files (Markdown format) that document:
- System-level troubleshooting (systemd, D-Bus service issues)
- Application diagnostics (Java/Kafka dependency issues)
- Product architecture analysis
- Step-by-step repair procedures

All documentation is written in Chinese and follows a structured format including problem description, root cause analysis, diagnosis steps, and solutions.

## Common Operations

### Git Workflow

This repository uses a standard Git workflow with GitHub as the remote (`git@github.com:hujinnvshi/system_xiufu_20260112.git`):

```bash
# Check status
git status

# Add and commit changes
git add <filename>
git commit -m "description"

# Push to remote
git push origin main
```

### Creating Documentation

When creating new diagnostic documents:

1. **Use descriptive filenames** with system identifier or problem type
   - Example: `systemd_dbus_troubleshooting_guide.md`
   - Example: `product_diagnosis_172.16.196.203.md`

2. **Follow the established template**:
   - Problem date and system information
   - Problem symptoms (with command examples)
   - Root cause analysis
   - Step-by-step diagnosis process
   - Solutions (multiple approaches when applicable)
   - Prevention measures
   - Technical appendix

3. **Include code blocks** with proper syntax highlighting for:
   - Bash commands
   - Configuration files
   - Log excerpts
   - Error messages

4. **Document the complete diagnostic flow**, including:
   - Commands used for investigation
   - Expected vs actual output
   - Reasoning process
   - Verification steps

### Documentation Best Practices

- **Be specific**: Include exact error messages, file paths, and command outputs
- **Provide context**: Explain why a particular command or check is performed
- **Include verification**: Always document how to verify that a fix worked
- **Add prevention**: Include monitoring suggestions and preventive measures
- **Use diagrams**: Include architecture diagrams when explaining complex systems

## Key Information for Common Issues

### Systemd Service Failures

Common diagnostic pattern for systemctl issues:
1. Check if PID 1 is systemd (`ps -p 1 -f`, `readlink -f /sbin/init`)
2. Verify D-Bus daemon is running (`ps aux | grep dbus-daemon`)
3. Check dbus-daemon-launch-helper permissions (should be 4755)
4. Examine systemd file descriptors (`ls -la /proc/1/fd`)
5. Review system logs (`journalctl -p err -b`, `dmesg`)

**Key file**: `/usr/lib/dbus-1.0/dbus-daemon-launch-helper`
- Correct permissions: `4755` (setuid root)
- If permissions are wrong (e.g., `0777`), systemctl will fail with "The permission of the setuid helper is not correct"

### Java/Kafka Dependency Issues

Common diagnostic pattern for application startup failures:
1. Check service logs (`journalctl -u <service> -n 50`)
2. Look for `ClassNotFoundException` or `NoClassDefFoundError`
3. Examine classpath in process command (`ps aux | grep java`)
4. Check Kafka version compatibility (API changed between 0.8.x and 0.9+)
5. Verify jar files in `/usr/share/server/lib/`

**Critical issue**: Kafka 0.8.x API (`kafka.serializer.Encoder`) was removed in later versions and replaced with `org.apache.kafka.common.serialization.Serializer`

### Remote System Diagnosis

When diagnosing remote systems (e.g., `root@172.16.196.203`):

1. **Check SSH access**: Ensure SSH keys are configured for passwordless access
2. **Verify system state**: `uptime`, `systemctl is-system-running`
3. **List services**: `systemctl list-units --type=service --state=running`
4. **Check for failed units**: `systemctl --failed`
5. **Examine logs**: Tail relevant log files and use `journalctl`

### System Restart Considerations

When systemd PID 1 has corrupted state (e.g., only 3 file descriptors - stdin/stdout/stderr with no sockets):
- Normal service restart won't work
- Must use SysRq for forced reboot: `echo b > /proc/sysrq-trigger`
- This bypasses systemd's normal shutdown mechanism

## Writing Style

- **Language**: Chinese (as this is a Chinese system administration context)
- **Tone**: Technical and objective
- **Format**: Structured with clear headings and subheadings
- **Audience**: System administrators and DevOps engineers
- **Verbosity**: Detailed enough to follow the reasoning process

## Document Metadata

Each diagnostic document should include:
- Date of diagnosis
- System information (OS, hostname, IP address)
- Problem statement
- Root cause
- Solutions attempted
- Final resolution
- Verification steps

## Commit Message Format

Use conventional commit format in Chinese:

```
docs: 添加 <主题> <文档类型>

- <要点1>
- <要点2>
- <要点3>

<详细说明>

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
```

Example:
```
docs: 添加 SystemD 和 D-Bus 服务故障排查指南

- 记录了 systemctl 完全失效的诊断流程
- 包含 dbus-daemon-launch-helper 权限错误的修复方案
- 提供了完整的诊断脚本和命令速查表

问题主机：172.16.196.203
修复时间：2026-01-12

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
```

## Safety Considerations

- **Always create backups** before suggesting system modifications
- **Warn users** about destructive operations (reboots, service restarts)
- **Provide rollback options** when possible
- **Never assume** SSH passwordless access is configured; verify first
- **Check system state** before and after operations
- **Document risks** clearly when suggesting fixes that require system restarts
