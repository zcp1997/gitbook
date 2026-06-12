---
description: SSH、防火墙、fail2ban、用户权限、最小权限和审计日志等安全配置。
icon: shield-halved
---

# 安全与加固

用于记录 SSH、防火墙、fail2ban、用户权限、最小权限、审计日志等安全配置。这里的命令通常会改变服务器暴露面，执行前要保留当前 SSH 会话，避免把自己锁在机器外面。

## Fail2ban 与 CyberSentry

[CyberSentry](https://github.com/CurtisLu1/CyberSentry) 是一个面向服务器安全防护的一键部署项目，集成了 SSH 蜜罐、Fail2ban、防火墙配置、日志清理、系统监控和 SSH 安全配置等能力。

根据项目 README，它主要包含这些能力：

- 部署 Cowrie SSH 蜜罐，用于记录攻击者行为和攻击模式。
- 集成 Fail2ban，对异常登录尝试进行自动封禁。
- 支持日志轮转与清理，默认清理 30 天前日志。
- 可选配置 SSH 端口、认证方式和密钥。
- 可选配置 UFW 防火墙规则，自动处理 SSH 与蜜罐端口。
- 修改关键配置前会做配置备份，备份目录通常在 `/root/config_backups/`。

{% hint style="warning" %}
这类一键安全脚本会修改 SSH、防火墙、Fail2ban 和系统服务配置。远程服务器上执行前，务必保留一个已登录的 SSH 会话，并确认云厂商安全组、UFW 和 SSH 新端口都允许访问。
{% endhint %}

### 一键安装

```bash
bash <(curl -sL https://raw.githubusercontent.com/CurtisLu1/CyberSentry/main/install.sh)
```

### 适用场景

- 新服务器初始化后，希望快速补上 SSH 防护、Fail2ban 和基础安全配置。
- 暴露公网 SSH 的个人服务器，需要自动封禁暴力破解来源。
- 希望部署 SSH 蜜罐，观察扫描和攻击行为。
- 希望统一处理安全日志、日志轮转和基础监控。

### 常用检查命令

查看 SSH jail 的封禁状态：

```bash
fail2ban-client status sshd
```

常见服务状态检查：

```bash
systemctl status fail2ban
systemctl status cowrie
ufw status
```

常见日志查看：

```bash
tail -f /var/log/fail2ban.log
tail -f /opt/cowrie/var/log/cowrie/cowrie.log
journalctl -u cowrie -f
```

### 关键参数记录

项目 README 中给出的 Fail2ban 默认配置思路：

- 封禁时间：24 小时，即 `86400` 秒。
- 检测窗口：5 分钟，即 `300` 秒。
- 最大重试：3 次。
- 监控日志：系统认证日志和 Cowrie 日志。

### 执行后检查清单

- 确认当前 SSH 端口还能正常连接。
- 确认云厂商安全组已放行实际 SSH 端口。
- 确认 UFW 规则没有误拦截自己的管理入口。
- 确认 `fail2ban` 服务正常运行。
- 确认 `fail2ban-client status sshd` 能看到 sshd jail 状态。
- 如果启用了 Cowrie，确认蜜罐端口和真实 SSH 端口没有冲突。
