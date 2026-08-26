---
description: 通过 Tailscale Exit Node 将 Windows 与 iPhone 流量送回家中 Mac mini，再由 Surge Enhanced Mode 统一完成 DNS、分流和代理出口。
icon: route
---

# Surge Mac + Tailscale Exit Node：让 Windows / iPhone 全部流量从家里 Mac 出口

最近折腾了一套比较舒服的远程代理方案：

- 家里的 Mac mini 长期开着 Surge Mac
- Surge 开启“系统代理”和“增强模式”
- Windows / iPhone 只安装 Tailscale
- 远程设备把 Mac mini 设为 Exit Node
- 所有外网流量先回家，再交给 Surge 分流和选择节点
- Windows / iPhone 不需要再单独运行 Clash、sing-box 等代理软件

最终结构：

```text
Windows / iPhone
       │
       │ Tailscale Direct P2P
       ▼
    Mac mini
       │
       ▼
     Surge
  Enhanced Mode
       │
       ├── DIRECT
       ├── Google
       ├── AI
       ├── Telegram
       └── 其他 Proxy Group
```

## 1. Mac mini 配置 Tailscale Exit Node

Mac 安装 Tailscale。对于 App Store 或 Standalone GUI 版本，可以从菜单中选择：

```text
Exit Node → Run Exit Node
```

如果已经为 Tailscale 启用 CLI integration，也可以执行：

```bash
sudo tailscale set --advertise-exit-node
```

然后到 Tailscale Admin Console：

```text
Machines
→ Mac mini
→ Edit route settings
→ Allow as exit node
```

远程 Windows / iPhone 就可以选择：

```text
Exit Node → Mac mini
```

Windows 如果还需要访问当前所在网络的本地资源，记得开启：

```text
Allow local network access
```

这时远程设备已经可以：

```text
Windows / iPhone
→ Tailscale
→ Mac mini
→ Internet
```

---

## 2. Surge Mac 保持增强模式

Mac 上 Surge 正常开启：

```text
系统代理      ON
增强模式      ON
```

远程设备不需要设置 HTTP/SOCKS 代理。

Windows 的系统代理反而应该关闭：

```text
使用代理服务器 → OFF
```

因为现在接管流量的是 Tailscale Exit Node，而不是 Windows 系统代理。

---

## 3. 最大的坑：Surge Fake-IP DNS

一开始会遇到一个很奇怪的问题：

Windows 可以 ping、Telegram 也能用，但 Chrome 打不开网页。

Windows 查询：

```powershell
Resolve-DnsName google.com
```

得到：

```text
google.com
198.19.6.148
```

`198.18.0.0/15` 是 Surge Enhanced Mode 使用的 Fake-IP 地址段。

问题在于：

```text
Windows
→ DNS 请求到 Mac Surge
→ Surge 返回 198.19.x.x
→ Windows 连接 Fake-IP
→ Fake-IP 没有路由回 Surge
→ 网页打不开
```

如果简单关闭 Tailscale DNS：

```powershell
tailscale set --accept-dns=false
```

网页确实能恢复。

但这样又产生另一个问题：

> Surge 收到远程设备流量时，经常只能看到真实 IP，无法完整使用 DOMAIN / DOMAIN-SET 规则。

比如 Speedtest：

```text
<Speedtest 节点 IP>:8080
→ FINAL
```

而不是：

```text
hk-hkg12.speed.misaka.one
→ DOMAIN-SET speedtest.conf
```

因此更好的办法不是禁用 Fake-IP，而是把 Fake-IP 路由打通。

---

## 4. 给 Tailscale 发布 Surge Fake-IP 网段

在 Mac mini 执行：

```bash
sudo tailscale set --advertise-routes=198.18.0.0/15
```

然后去 Tailscale Admin Console：

```text
Machines
→ Mac mini
→ Edit route settings
```

批准：

```text
198.18.0.0/15
```

路由批准和访问权限是两层独立配置。如果 tailnet 使用了自定义 ACL 或 grants，还需要允许目标客户端访问 `198.18.0.0/15`。

Windows、iOS 和 macOS 默认会自动接受新的 subnet routes。如果 Windows 之前关闭过接受路由，可以显式重新启用：

```powershell
tailscale set --accept-routes=true
```

DNS 也重新打开：

```powershell
tailscale set --accept-dns=true
ipconfig /flushdns
```

检查：

```powershell
Resolve-DnsName google.com
```

仍然应该得到类似：

```text
198.19.6.148
```

但这一次：

```powershell
Find-NetRoute -RemoteIPAddress 198.19.6.148
```

应该看到：

```text
DestinationPrefix : 198.18.0.0/15
InterfaceAlias    : Tailscale
```

再测试：

```powershell
Test-NetConnection 198.19.6.148 -Port 443
```

如果：

```text
TcpTestSucceeded : True
```

Fake-IP 链路就打通了。

完整路径变成：

```text
Windows / iPhone
      │
      │ DNS
      ▼
 Mac Surge DNS
      │
      ▼
 198.18/15 Fake-IP
      │
      ▼
 Tailscale Subnet Route
      │
      ▼
   Mac Surge VIF
      │
      ▼
恢复真实域名
      │
      ▼
DOMAIN / RULE-SET 分流
```

这一步是整套方案最关键的地方。

---

## 5. Surge 排除 Tailscale 自身流量

为了避免 Surge 把 Tailscale 自己的底层连接再套进代理，可以在 Surge 中加入：

```ini
[General]
tun-excluded-routes = 100.64.0.0/10
```

`100.64.0.0/10` 是 Tailscale 使用的地址范围。

如果遇到 Tailscale 明明显示 Direct，但延迟异常高，还可以临时把远程客户端的公网 IP 排除：

```ini
tun-excluded-routes = 100.64.0.0/10, <远程客户端公网 IP>/32
```

不过公网 IP 可能变化，所以这一条更适合作为排障手段，不建议无脑写死。

---

## 6. 怎么确认不是 DERP 中继

Windows：

```powershell
tailscale status
```

如果看到：

```text
active; exit node; direct <对端公网 IP>:21000
```

说明是 Direct P2P。

如果看到：

```text
relay "sfo"
```

才是 DERP。

也可以：

```powershell
tailscale ping <Mac mini 的 Tailscale IP>
```

直连类似：

```text
pong ... via <对端公网 IP>:21000 in 4ms
```

中继则会显示：

```text
via DERP(...)
```

我这边 Windows 到 Mac 最终稳定在：

```text
2ms ~ 9ms
```

iPhone 甚至可以直接通过 IPv6 建立：

```text
direct [<对端公网 IPv6>]:41641
```

家宽公网 IP 重新拨号也不用手工修改，Tailscale 会自动重新发现 Endpoint 和重新打洞。

---

## 7. 最终效果

Windows：

```text
只运行 Tailscale
Exit Node → Mac mini
```

iPhone：

```text
只运行 Tailscale
Exit Node → Mac mini
```

Mac：

```text
Tailscale Exit Node
+
Surge Enhanced Mode
+
Surge DNS / Fake-IP
+
Surge Rule / Proxy Group
```

最终：

```text
Windows / iPhone
        │
        ▼
Tailscale Direct P2P
        │
        ▼
     Mac mini
        │
        ▼
       Surge
        │
   DOMAIN 分流
        │
        ▼
   对应代理节点
```

实际测试 Speedtest 时，原本 Surge 只能看到：

```text
<Speedtest 节点 IP>:8080
→ FINAL
```

打通 `198.18.0.0/15` 后，已经可以正确恢复成：

```text
hk-hkg12.speed.misaka.one:8080
→ DOMAIN-SET speedtest.conf
→ 指定 HK 节点
```

包括 TCP、UDP 都可以正确命中域名规则。

## 总结

这套方案最大的好处是：

```text
Windows / iPhone 不再维护代理规则
                ↓
       所有策略集中到 Mac Surge
```

远程设备只负责运行 Tailscale，Surge 继续作为唯一的 DNS、分流和代理出口。

核心配置其实只有两个：

```bash
sudo tailscale set --advertise-exit-node
sudo tailscale set --advertise-routes=198.18.0.0/15
```

然后在 Tailscale Admin Console 批准 Exit Node 和 `198.18.0.0/15`。

如果只配置 Exit Node 而没有发布 `198.18.0.0/15`，很容易踩到 Surge Fake-IP 无法回流的问题。

**Exit Node + Fake-IP Subnet Route 打通以后，才算是真正把 Surge Mac 变成了一台远程 Surge Gateway。**

## 参考资料

- [Tailscale Exit nodes](https://tailscale.com/docs/features/exit-nodes)
- [Tailscale Subnet routers](https://tailscale.com/docs/features/subnet-routers)
- [Surge General Section Options](https://manual.nssurge.com/profile/general.html)
