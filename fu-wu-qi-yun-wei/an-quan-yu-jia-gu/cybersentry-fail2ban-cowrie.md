---
description: 在 Debian/Ubuntu VPS 上部署 Fail2ban SSH 防护与 Cowrie 3 蜜罐，并保留 SSH、UFW 和系统 Python 的现有配置。
icon: shield
---

# CyberSentry：Fail2ban 与 Cowrie 3 安全部署

CyberSentry 是一个面向 Debian/Ubuntu VPS 的 Bash 安装器，用于部署：

- Fail2ban `sshd` jail；
- Cowrie 3 SSH 蜜罐；
- Cowrie systemd 服务；
- 仅覆盖 Cowrie 日志的 logrotate 规则。

这个维护版本修复了原项目与 Cowrie 3.x 的兼容问题，并收窄了安装器对整台服务器的修改范围。

<a href="https://github.com/zcp1997/CyberSentry/blob/55c0084edb4be3959ad64689d17699dbbc156490/install.sh" class="button primary" data-icon="github">查看完整 install.sh</a>

[下载固定版本 `install.sh`](https://raw.githubusercontent.com/zcp1997/CyberSentry/55c0084edb4be3959ad64689d17699dbbc156490/install.sh)，固定到提交 `55c0084`。

## 安全边界

安装器会安装依赖，并写入 Cowrie、Fail2ban、systemd 与 logrotate 的受管配置，但不会：

- 修改 SSH 端口、认证方式或密钥；
- 启用或修改 UFW；
- 改写 APT 软件源或执行发行版升级；
- 替换系统默认 `python3`；
- 修改系统时区；
- 清理其他服务的 `/var/log` 日志；
- 直接删除无法识别的 `/opt/cowrie`。

{% hint style="warning" %}
这是 root 级服务器安装器，会安装软件包并修改 `/etc/fail2ban`、`/etc/systemd/system`、`/etc/logrotate.d` 和 `/opt/cowrie`。远程执行时应保留一个已登录的 SSH 会话，并先确认云安全组和当前 SSH 管理入口可用。
{% endhint %}

## 环境要求

- Debian 或 Ubuntu；
- systemd；
- root 权限；
- Python 3.10 或更高版本。

通常建议使用 Debian 12+ 或 Ubuntu 22.04+。安装器在 Python 版本不满足要求时会停止，不会通过 PPA、跨发行版软件源或 `update-alternatives` 强行替换系统 Python。

## 安装

建议固定到经过验证的提交，先下载、检查，再执行：

```bash
curl -fsSLo install-cybersentry.sh \
  https://raw.githubusercontent.com/zcp1997/CyberSentry/55c0084edb4be3959ad64689d17699dbbc156490/install.sh
printf '%s  %s\n' \
  '886019e7c2fc6c41cd3ad62d3670d427440b137a2e06c1fc28c125fe1ded59bd' \
  'install-cybersentry.sh' | sha256sum -c -
less install-cybersentry.sh
sudo bash install-cybersentry.sh
```

默认值：

- Cowrie 版本：`3.0.0`；
- 安装目录：`/opt/cowrie`；
- 蜜罐监听端口：`2222/tcp`；
- 蜜罐显示主机名：脚本内置固定默认值，部署前建议通过 `COWRIE_HOSTNAME` 覆盖为不对应真实资产的名称；
- 单个下载大小限制：1 MiB；
- Cowrie 日志轮转：每日轮转，保留 30 份；
- Fail2ban：24 小时封禁、30 分钟检测窗口、3 次失败。

## 自定义参数

安装器通过环境变量接收自定义值：

```bash
sudo \
  COWRIE_VERSION=3.0.0 \
  COWRIE_HOSTNAME=my-honeypot \
  COWRIE_SSH_PORT=2222 \
  COWRIE_DOWNLOAD_LIMIT=1048576 \
  LOG_RETENTION_DAYS=30 \
  bash install-cybersentry.sh
```

参数约束：

- `COWRIE_SSH_PORT`：`1024-65535`；
- `COWRIE_HOSTNAME`：最多 64 个字符，只允许字母、数字、点、下划线和连字符；
- `COWRIE_VERSION`：应固定为已经验证的 Cowrie 发行版本；
- `COWRIE_SETTLE_SECONDS`：Cowrie 启动稳定性观察时间，允许 `3-60` 秒；
- `FAIL2BAN_READY_TIMEOUT`：Fail2ban 就绪等待时间，允许 `5-120` 秒。

## 部署逻辑

安装器按以下顺序工作：

1. 检查 root、Debian/Ubuntu、systemd 和参数格式。
2. 记录 Fail2ban、Cowrie 的启用及运行状态，并备份将要修改的受管文件。
3. 安装 Cowrie 编译依赖、Fail2ban、logrotate 和 Python venv 支持，不执行系统升级。
4. 创建专用的 `cowrie` 系统用户和 `/opt/cowrie/cowrie-env` 虚拟环境。
5. 从 PyPI 安装固定版本的 Cowrie，并运行 `cowrie init` 初始化状态目录。
6. 协调 Cowrie 的 `hostname`、`download_limit_size` 和 `listen_endpoints` 三个受管键。
7. 从 `sshd -T` 读取真实生效的一个或多个 SSH 端口，再生成 Fail2ban jail。
8. 写入 Cowrie systemd unit 和 Cowrie 专用 logrotate 规则。
9. 启动服务，等待 Fail2ban socket 与 `sshd` jail 就绪。
10. 连续观察 Cowrie 服务状态，并确认指定 TCP 端口存在监听 socket。

## Fail2ban 配置

安装器生成 `/etc/fail2ban/jail.d/cybersentry.local`。其中 `port` 不是固定写死为 22，而是来自 `sshd -T` 返回的有效端口：

```ini
[sshd]
enabled = true
backend = systemd
port = <sshd 实际生效端口，多个端口用逗号分隔>
filter = sshd
bantime = 86400
findtime = 1800
maxretry = 3
```

写入后会依次执行：

```bash
fail2ban-client -t
systemctl enable fail2ban
systemctl restart fail2ban
```

随后轮询检查：

```bash
fail2ban-client ping
fail2ban-client status sshd
```

只有 Fail2ban socket 和 `sshd` jail 都就绪，安装器才会继续。

## Cowrie systemd 配置

Cowrie 以无登录权限的专用用户运行。关键 unit 配置如下：

```ini
[Unit]
Description=Cowrie SSH/Telnet Honeypot
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
User=cowrie
Group=cowrie
WorkingDirectory=/opt/cowrie
Environment=HOME=/opt/cowrie/var/lib/cowrie
Environment=PATH=/opt/cowrie/cowrie-env/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
ExecStart=/opt/cowrie/cowrie-env/bin/twistd --umask=0022 --nodaemon --pidfile= -l - cowrie
Restart=on-failure
RestartSec=10
UMask=0077
NoNewPrivileges=true
PrivateTmp=true
ProtectHome=true
ProtectSystem=strict
ReadWritePaths=/opt/cowrie/var
ProtectKernelTunables=true
ProtectKernelModules=true
ProtectControlGroups=true
RestrictSUIDSGID=true

[Install]
WantedBy=multi-user.target
```

unit 中的 `Description` 沿用 Cowrie 的通用 SSH/Telnet 名称；这个安装器只配置和验证 SSH endpoint，没有启用 Telnet listener。

`ProtectSystem=strict` 将安装树设为只读，仅允许 Cowrie 写入 `/opt/cowrie/var`。`--pidfile=` 明确关闭 Twisted PID 文件，因为 systemd 已负责跟踪主进程；否则 Twisted 会尝试在只读的 `/opt/cowrie/twistd.pid` 写文件并退出。

Cowrie 静态安装树、配置和 venv 由 root 管理，运行时状态目录 `/opt/cowrie/var` 归 `cowrie` 用户所有。

## Cowrie 日志轮转

安装器只处理 Cowrie 自己的日志，不会扫描或删除其他服务的日志：

```text
/opt/cowrie/var/log/cowrie/*.log
/opt/cowrie/var/log/cowrie/*.json
```

规则为每日轮转、压缩、延迟压缩和 `copytruncate`，保留份数由 `LOG_RETENTION_DAYS` 控制。

## 安装后验证

查看服务状态：

```bash
systemctl status cowrie fail2ban --no-pager
systemctl is-active cowrie fail2ban
```

确认 Cowrie 端口监听：

```bash
ss -ltnp 'sport = :2222'
systemctl show cowrie -p MainPID -p SubState
```

使用自定义 `COWRIE_SSH_PORT` 时，将 `2222` 替换为实际端口。安装器内置检查会同时要求 Cowrie 服务保持 `active` 且目标端口存在 TCP listener，但不会进一步核对该 socket 的 PID 归属；安装后可通过上面的 `ss -ltnp` 与 systemd 主进程信息交叉确认。

查看 Fail2ban jail：

```bash
fail2ban-client status sshd
```

查看日志：

```bash
journalctl -u cowrie -f
tail -f /opt/cowrie/var/log/cowrie/cowrie.log
tail -f /var/log/fail2ban.log
```

安装器的服务状态与端口检查均通过时，会明确输出：

```text
Cowrie 已稳定运行 8 秒并监听 2222/tcp
Fail2ban：active
Cowrie：active
```

## 防火墙与公网 22 端口

安装器不会自动启用或修改 UFW。已经使用 UFW 时，手动放行蜜罐端口：

```bash
ufw allow 2222/tcp comment 'Cowrie Honeypot'
ufw status
```

{% hint style="danger" %}
Cowrie 默认监听高位端口 `2222`，不会自动接管公网 `22`。如需捕获发往 22 端口的流量，应先把真实 SSH 安全迁移到其他端口，并通过另一个会话验证登录成功，再单独配置 nftables/iptables 的 `22 -> 2222` 转发。
{% endhint %}

## 重复执行、备份与失败恢复

安装器会识别由自身管理且版本一致的 Cowrie 安装，跳过重复安装 venv，但仍协调三个受管配置键并重新验证服务。

无法识别、旧版或不完整且非空的 `/opt/cowrie` 不会被删除，而会完整移动到：

```text
/opt/cowrie.legacy-YYYYMMDD_HHMMSS-PID
```

安装前已经存在的普通受管配置文件会备份到：

```text
/root/config_backups/
```

如果安装中途失败，EXIT trap 会尝试：

- 恢复 Fail2ban、systemd、logrotate 和 Cowrie 配置；
- 恢复服务原先的启用和运行状态；
- 将本次失败的新安装保留为 `/opt/cowrie.failed-*`；
- 将安装前迁移的旧 Cowrie 目录放回原位置。

回滚范围不包括 apt 已安装或升级的软件包、现有 `cowrie` 账户的 home/shell/group 调整，以及所有已经发生的递归权限变更。它主要覆盖受管文件、服务状态与 Cowrie 目录迁移，不等于完整的系统快照恢复。

成功部署后如果需要回退，应先停止 Cowrie，核对 `/root/config_backups/` 与 `.legacy-*` 中的文件，再按实际旧版本恢复；不要直接删除当前目录或猜测旧版 unit 的启动方式。

## Cowrie 3 兼容问题记录

这个版本处理了几类常见故障：

### `cowrie.cfg.dist` 不存在

Cowrie 3 的 PyPI 安装布局不再适合依赖源码仓库里的固定模板路径。安装器改为安装固定 PyPI 版本，然后执行：

```bash
/opt/cowrie/cowrie-env/bin/cowrie init
```

由当前版本生成配置和状态目录。

### Fail2ban 重启后立即查询失败

`systemctl restart fail2ban` 返回时，Fail2ban socket 和 jail 不一定已经可查询。安装器会等待 `fail2ban-client ping` 与 `status sshd` 同时成功，而不是立即判定失败。

### Twisted 尝试写入只读 PID 文件

在 `ProtectSystem=strict` 下，Twisted 默认写入 `/opt/cowrie/twistd.pid` 会触发：

```text
OSError: [Errno 30] Read-only file system: '/opt/cowrie/twistd.pid'
```

systemd unit 直接运行 `twistd` 并传入空 PID 文件参数：

```bash
--pidfile=
```

这样保留只读加固，不需要扩大 Cowrie 对安装目录的写权限。

### Twisted 插件缓存只读警告

只读 venv 下可能出现无法更新 `dropin.cache` 的警告。只要 Cowrie 插件已加载、服务保持 `active` 且端口实际监听，这条警告本身不是启动失败原因。

## 版本与审查状态

本文固定的安装器版本：

```text
55c0084edb4be3959ad64689d17699dbbc156490
```

该版本已经完成：

- `bash -n install.sh` 语法检查；
- ShellCheck 静态检查；
- 25 项安装器契约测试；
- GitHub Actions 检查；
- 一次真实 VPS 安装输出记录：Fail2ban 与 Cowrie 均为 `active`，目标端口 `2222/tcp` 存在监听 socket。

项目使用 MIT License。后续版本应以仓库的提交历史、测试状态和源码审查结果为准。
