---
description: SSH、防火墙、fail2ban、用户权限、最小权限和审计日志等安全配置。
icon: shield-halved
---

# 安全与加固

用于记录 SSH、防火墙、fail2ban、用户权限、最小权限、审计日志等安全配置。这里的命令通常会改变服务器暴露面，执行前要保留当前 SSH 会话，避免把自己锁在机器外面。

## Fail2ban 与 CyberSentry

[CyberSentry：Fail2ban 与 Cowrie 3 安全部署](an-quan-yu-jia-gu/cybersentry-fail2ban-cowrie.md)记录了经过修复和真实 VPS 验证的安装器，包括完整源码入口、固定版本安装命令、安全边界、Fail2ban jail、Cowrie systemd unit、验证方式与失败恢复。

维护版本只部署 Fail2ban SSH 防护、Cowrie 蜜罐及其日志轮转，不会修改 SSH、UFW、APT 软件源或其他服务日志；它会安装所需 Python 软件包，但不会替换系统默认 `python3`、改写 alternatives 或添加跨发行版软件源。
