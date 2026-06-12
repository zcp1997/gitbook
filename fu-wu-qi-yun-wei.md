---
description: 服务器日常运维记录、排障手册与可复用脚本索引。
icon: server
---

# 服务器运维

这里用来沉淀服务器日常维护中可复用的命令、脚本和排障经验。内容尽量写成“问题场景 → 适用条件 → 操作步骤 → 验证方式”的结构，避免只保存一条孤立命令，之后回看时更容易判断能不能直接使用。

## 目录规划

- **常见工具脚本**：系统初始化、网络配置、包管理、日志清理、证书、备份等高频命令。
- **性能测试脚本**：CPU、内存、磁盘、网络吞吐、延迟和压测相关工具。
- **排障手册**：线上异常、端口占用、DNS、IPv6/IPv4、磁盘写满、负载过高等问题的处理步骤。
- **安全与加固**：SSH、用户权限、防火墙、fail2ban、最小权限、审计日志等配置记录。
- **服务部署片段**：systemd、Docker、反向代理、定时任务和健康检查模板。

{% hint style="warning" %}
涉及系统配置的命令，尤其是 `/etc` 下的文件修改，执行前建议先备份原文件，并确认当前机器的发行版和网络环境。
{% endhint %}

## 常见工具脚本

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
