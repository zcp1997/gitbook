---
description: 服务器日常运维记录、排障手册与可复用脚本索引。
icon: server
---

# 服务器运维

这里用来沉淀服务器日常维护中可复用的命令、脚本和排障经验。内容尽量写成“问题场景 → 适用条件 → 操作步骤 → 验证方式”的结构，避免只保存一条孤立命令，之后回看时更容易判断能不能直接使用。

{% hint style="warning" %}
涉及系统配置的命令，尤其是 `/etc` 下的文件修改，执行前建议先备份原文件，并确认当前机器的发行版、网络环境和当前 SSH 会话可恢复。
{% endhint %}

## 常见工具脚本

用于记录系统初始化、网络配置、包管理、日志清理、证书、备份等高频命令。原则是：能复用的命令保留下来，同时写清楚适用场景、风险和验证方式。

### 设置 IPv4 优先

部分服务器同时启用 IPv6 和 IPv4 时，系统默认地址选择可能优先走 IPv6。如果当前网络的 IPv6 质量不稳定，可能出现 `apt`、`curl`、`wget`、Docker 拉取镜像等操作连接慢、超时或偶发失败。

可以通过调整 `/etc/gai.conf` 中的地址选择优先级，让系统在解析到 IPv4-mapped IPv6 地址时优先使用 IPv4。

{% hint style="info" %}
这个配置不会关闭 IPv6，只是调整地址选择优先级。适合“IPv6 可用但质量不稳定，想优先走 IPv4”的场景。
{% endhint %}

#### 操作命令

```bash
sudo cp /etc/gai.conf /etc/gai.conf.bak.$(date +%Y%m%d_%H%M%S)

sudo sed -i 's/#precedence ::ffff:0:0\/96  100/precedence ::ffff:0:0\/96  100/' /etc/gai.conf
```

#### 验证方式

查看配置是否已经生效写入：

```bash
grep '^precedence ::ffff:0:0/96  100' /etc/gai.conf
```

再用实际访问测试确认效果，例如：

```bash
curl -4 -I https://github.com
curl -I https://github.com
```

如果普通 `curl` 的表现恢复正常，说明地址选择优先级调整对当前场景有效。

#### 回滚方式

如果修改后出现异常，恢复备份文件：

```bash
sudo cp /etc/gai.conf.bak.YYYYMMDD_HHMMSS /etc/gai.conf
```

把命令里的备份文件名替换成实际生成的文件名。

## 性能测试脚本

用于记录服务器性能基线和压测相关命令，后续可以按场景拆分为 CPU、内存、磁盘、网络和业务压测。

建议每条性能测试记录都补齐这些信息：

- 测试目标：例如磁盘吞吐、网络延迟、并发请求能力。
- 测试工具：例如 `fio`、`iperf3`、`wrk`、`sysbench`。
- 测试命令：保留可直接复制执行的版本。
- 环境信息：机器规格、系统版本、地域、磁盘类型、网络线路。
- 结果记录：保留关键指标，方便以后横向对比。

## 排障手册

用于记录线上异常和系统问题的定位流程。优先写成检查清单，减少临场排障时的遗漏。

建议按问题类型沉淀：

- 端口占用：确认监听进程、启动来源和是否允许停止。
- DNS 异常：检查解析结果、上游 DNS、IPv4/IPv6 选择和缓存。
- 磁盘写满：定位大文件、日志增长、Docker 占用和 inode 状态。
- 负载过高：区分 CPU、IO、内存、网络和异常进程。
- 服务不可用：检查 systemd 状态、日志、端口、反向代理和依赖服务。

## 安全与加固

用于记录 SSH、防火墙、fail2ban、用户权限、最小权限、审计日志等安全配置。这里的命令通常会改变服务器暴露面，执行前要保留当前 SSH 会话，避免把自己锁在机器外面。

### Fail2ban 与 CyberSentry

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

#### 一键安装

```bash
bash <(curl -sL https://raw.githubusercontent.com/CurtisLu1/CyberSentry/main/install.sh)
```

#### 适用场景

- 新服务器初始化后，希望快速补上 SSH 防护、Fail2ban 和基础安全配置。
- 暴露公网 SSH 的个人服务器，需要自动封禁暴力破解来源。
- 希望部署 SSH 蜜罐，观察扫描和攻击行为。
- 希望统一处理安全日志、日志轮转和基础监控。

#### 常用检查命令

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

#### 关键参数记录

项目 README 中给出的 Fail2ban 默认配置思路：

- 封禁时间：24 小时，即 `86400` 秒。
- 检测窗口：5 分钟，即 `300` 秒。
- 最大重试：3 次。
- 监控日志：系统认证日志和 Cowrie 日志。

#### 执行后检查清单

- 确认当前 SSH 端口还能正常连接。
- 确认云厂商安全组已放行实际 SSH 端口。
- 确认 UFW 规则没有误拦截自己的管理入口。
- 确认 `fail2ban` 服务正常运行。
- 确认 `fail2ban-client status sshd` 能看到 sshd jail 状态。
- 如果启用了 Cowrie，确认蜜罐端口和真实 SSH 端口没有冲突。

## 服务部署片段

用于记录 systemd、Docker、反向代理、定时任务和健康检查模板。后续可以把常用服务拆成固定片段，例如：

- systemd service 模板。
- Docker Compose 模板。
- Nginx / Caddy 反向代理配置。
- cron 定时任务模板。
- 健康检查和自动重启策略。
