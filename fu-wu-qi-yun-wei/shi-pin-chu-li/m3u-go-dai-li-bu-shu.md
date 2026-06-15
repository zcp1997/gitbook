---
description: 部署一个 Go 写的 M3U/HLS 代理，用于改写播放列表并让播放器通过代理拉取 m3u8、ts、key 等资源。
icon: tower-broadcast
---

# M3U Go 代理部署

## 场景

当 IPTV 或 HLS 播放列表里的源地址在当前网络环境下访问不稳定，或需要统一从一台 VPS 中转时，可以部署一个轻量 Go 代理。

代理做三件事：

* 拉取原始 M3U/M3U8 播放列表。
* 把播放列表里的媒体地址、子播放列表、`EXT-X-KEY`、`EXT-X-MAP` 等 URI 改写为代理地址。
* 播放器后续请求 `/proxy?u=BASE64URL` 时，由 VPS 去请求真实上游并转发。

{% hint style="warning" %}
这是一个中转代理。公网部署时必须设置 `-token`，并优先放在 HTTPS/Nginx 后面。不要裸开无鉴权服务，否则会变成开放代理，也可能带来 SSRF 风险。
{% endhint %}

## 关键鉴权要求

`main.go` 必须使用运行时参数 `-token`，不要依赖硬编码 token。

`proxyConfig` 需要包含：

```go
type proxyConfig struct {
    PublicBase string
    UserAgent  string
    AuthToken  string
    Verbose    bool
}
```

创建服务时传入 token：

```go
svc := newProxyService(proxyConfig{
    PublicBase: *publicBase,
    UserAgent:  *userAgent,
    AuthToken:  *authToken,
    Verbose:    *verbose,
})
```

生成代理 URL 时必须把 token 一起带给二跳请求：

```go
func (s *proxyService) makeProxyURL(realURL string) string {
    proxyURL := fmt.Sprintf("%s/proxy?u=%s", s.cfg.PublicBase, encodeURL(realURL))
    if s.cfg.AuthToken != "" {
        proxyURL += "&token=" + url.QueryEscape(s.cfg.AuthToken)
    }
    return proxyURL
}
```

否则初始 M3U 虽然能返回，但播放器后续拉 `/proxy?u=...` 时会因为缺少 token 被 401 拒绝。

## 部署目录

```bash
mkdir -p /opt/m3u-proxy
cp m3u-proxy-linux-amd64 /opt/m3u-proxy/m3u-proxy
chmod +x /opt/m3u-proxy/m3u-proxy
```

如果是在服务器本机编译：

```bash
cd /opt/m3u-proxy/src
go test ./...
go vet ./...
go build -o /opt/m3u-proxy/m3u-proxy .
chmod +x /opt/m3u-proxy/m3u-proxy
```

## 前台测试

先用前台方式启动，确认参数和日志正常：

```bash
/opt/m3u-proxy/m3u-proxy \
  -port 8099 \
  -base http://SERVER_HOST:8099 \
  -v \
  -token 'TOKEN_PLACEHOLDER'
```

测试健康检查：

```bash
curl -fsS 'http://127.0.0.1:8099/health'
```

测试动态改写：

```bash
curl -fsS 'http://127.0.0.1:8099/?url=https://example.com/list.m3u&token=TOKEN_PLACEHOLDER' | head
```

返回内容里应能看到被改写后的代理地址，形如：

```text
http://SERVER_HOST:8099/proxy?u=BASE64URL&token=TOKEN_PLACEHOLDER
```

## nohup 后台运行

确认前台测试正常后，再用 `nohup` 后台运行：

```bash
nohup /opt/m3u-proxy/m3u-proxy \
  -port 8099 \
  -base http://SERVER_HOST:8099 \
  -v \
  -token 'TOKEN_PLACEHOLDER' \
  > /opt/m3u-proxy/m3u-proxy.log 2>&1 &
```

查看进程：

```bash
pgrep -af m3u-proxy
```

查看日志：

```bash
tail -f /opt/m3u-proxy/m3u-proxy.log
```

停止进程：

```bash
pkill -f '/opt/m3u-proxy/m3u-proxy'
```

{% hint style="warning" %}
`pkill -f` 会按命令行匹配进程。执行前先用 `pgrep -af m3u-proxy` 确认只会命中目标服务。
{% endhint %}

## 固定上游模式

如果希望服务定时缓存一个固定 M3U，可以加 `-upstream`：

```bash
nohup /opt/m3u-proxy/m3u-proxy \
  -port 8099 \
  -base http://SERVER_HOST:8099 \
  -upstream 'https://example.com/list.m3u' \
  -cron 30 \
  -v \
  -token 'TOKEN_PLACEHOLDER' \
  > /opt/m3u-proxy/m3u-proxy.log 2>&1 &
```

播放器订阅地址：

```text
http://SERVER_HOST:8099/list.m3u?token=TOKEN_PLACEHOLDER
```

## Nginx 反代建议

生产环境建议使用 HTTPS 域名反代到本地端口：

```nginx
server {
    listen 443 ssl http2;
    server_name tv.example.com;

    location / {
        proxy_pass http://127.0.0.1:8099;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_buffering off;
        proxy_request_buffering off;
        proxy_read_timeout 5m;
        proxy_send_timeout 5m;
    }
}
```

反代后启动参数里的 `-base` 应改成 HTTPS 外部地址：

```bash
-base https://tv.example.com
```

## 验证清单

部署前检查：

```bash
go test ./...
go vet ./...
```

部署后检查：

```bash
curl -fsS 'http://127.0.0.1:8099/health'
curl -I 'http://127.0.0.1:8099/list.m3u?token=TOKEN_PLACEHOLDER'
tail -n 80 /opt/m3u-proxy/m3u-proxy.log
```

播放器无法播放时，优先检查：

* `-base` 是否是播放器可访问的公网地址。
* 初始 M3U 里的 `/proxy?u=...` 是否自动带上 `&token=TOKEN_PLACEHOLDER`。
* Nginx 或防火墙是否允许长连接和大流量转发。
* 上游是否要求特定 `User-Agent`，必要时用 `-ua` 覆盖。

## 安全注意

* `-token` 必须设置为长随机值，不要使用示例值。
* 不要把真实订阅 URL、token、内网 IP 写进公开文档或仓库。
* 服务如果暴露公网，建议只允许自己的播放器或反代入口访问。
* `/?url=` 和 `/proxy?url=` 能请求外部 URL，不能当作完全隔离的安全边界。
