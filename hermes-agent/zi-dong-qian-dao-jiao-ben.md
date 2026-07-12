---
description: V2EX 与一点万象自动签到脚本的实现思路
icon: robot
---

# 自动签到脚本

这页记录两个在 Hermes Agent 中运行过的自动签到脚本：V2EX 使用浏览器辅助获取登录态，再通过 HTTP 会话领取每日奖励；一点万象按移动端 H5 接口规则生成签名并提交签到请求。代码示例保留核心流程，实际部署时把本机相关配置替换为自己的值。

## V2EX：浏览器辅助登录 + HTTP 签到

### 思路

V2EX 的每日奖励页会在已登录状态下返回一个带 `once` 参数的动态领取链接。最初的方案是复用本机 Chrome Profile 中已有的登录态；实际运行后发现，在 macOS 上这条路容易被两个问题卡住：Chrome 正在运行时 Profile 文件会被锁，Chrome 退出后自动化进程又可能因为系统隐私权限读不到真实用户数据目录。

于是登录态改成由 `browser-use` 的独立自动化 Profile 维护：在可见窗口里手动登录一次，导出 cookie，定时脚本只读取该文件。这样不需要触碰真实 Chrome Profile，也不会受 Chrome 是否正在运行影响。

但“能打开每日任务页”不等于“能完成领取”。后续实际运行中，browser-use 可以正常读取已登录页面、解析出新的 `once`，访问领取链接后却仍然没有领取成功。领取响应里真正的错误是：

> 你的浏览器有一些奇奇怪怪的设置，请用一个干净安装的浏览器重试一下吧

直接导航领取链接、点击页面中的真实按钮、改用 `--headed` 可视模式都得到同样结果。这说明问题不在登录态、按钮点击方式或旧 token，而在 V2EX 对自动 Chromium 请求环境的判断。最终方案保留 browser-use 负责人工登录和 cookie 导出，定时签到则改用 `requests.Session`：加载同一份 cookie，以普通浏览器导航请求的形式获取动态链接、领取并复查。

{% hint style="info" %}
这里不是简单地在原 browser-use 脚本上补一个请求头。实际改动是把定时签到的执行引擎从自动 Chromium 换成 HTTP 会话；浏览器只保留在人工刷新登录态的环节。
{% endhint %}

核心流程：

1. 从本地 cookie JSON 创建 `requests.Session`。
2. 请求 `https://www.v2ex.com/mission/daily`，检查登录态和今日领取状态。
3. 从 HTML 提取 `/mission/daily/redeem?once=...`。
4. 如果已领取，直接返回成功；如果未登录，提示人工刷新 cookie。
5. 携带普通浏览器导航请求头，并将每日任务页设为 `Referer`，请求领取链接。
6. 再次请求每日任务页，以“页面明确显示已领取”为唯一成功条件。
7. 如果领取响应出现浏览器设置警告，单独报告为请求环境被拒绝，不要误判成 cookie 过期。
8. 日志只保留状态摘要，不输出 cookie、完整 HTML 或动态领取链接。

### 首次登录与刷新 cookie

第一次部署，或者 V2EX 登录态失效时，打开一个可见的自动化窗口手动登录：

```bash
browser-use --session v2ex-auth close
browser-use --session v2ex-auth --headed open 'https://www.v2ex.com/signin?next=%2Fmission%2Fdaily'
```

登录完成后，先确认自动化窗口已经看到登录态：

```bash
browser-use --session v2ex-auth eval "(() => document.body.innerText.includes('登出'))()"
```

再导出 cookie 到脚本使用的位置：

```bash
mkdir -p ~/.local/share/v2ex-signin
browser-use --session v2ex-auth cookies export --url 'https://www.v2ex.com' ~/.local/share/v2ex-signin/cookies.json
chmod 600 ~/.local/share/v2ex-signin/cookies.json
```

{% hint style="warning" %}
cookie 文件等同于登录凭据，应只允许当前用户读取。定时脚本不需要保存账号密码，也不应把 cookie、`once` 链接或完整响应体写入通知和普通日志。
{% endhint %}

如果仍想复用真实 Chrome Profile，需要先确保 Chrome 完全退出，并且运行自动化的终端或服务进程有权限访问 Chrome 用户数据目录；否则会遇到 Profile lock 或 `Operation not permitted`。

### 这次故障如何定位

失败表面上只是“访问领取链接后复查仍未显示已领取”。真正有区分度的证据来自领取后的页面正文：

1. 每日页显示“登出”入口，证明 cookie 仍有效。
2. 页面能生成 `redeem?once=...`，证明当前登录态能进入任务流程。
3. 领取前后 `once` 数字发生变化，只说明每日页重新生成了动态链接，不能证明上一次领取成功。
4. 领取后的页面包含“浏览器有一些奇奇怪怪的设置”，说明 V2EX 主动拒绝了请求环境。
5. browser-use 的直接导航、真实按钮点击和 headed 模式均失败，而同一份 cookie 在普通 HTTP 会话中领取并复查成功，最终把故障范围收敛到自动 Chromium 请求环境。

{% hint style="warning" %}
不要把“已访问 redeem URL”“HTTP 200”或“once 已变化”当成签到成功。成功必须由随后重新获取的每日页明确显示“每日登录奖励已领取”来确认。
{% endhint %}

{% code title="v2ex-signin-core.py" %}
```python
#!/usr/bin/env python3
from __future__ import annotations

import html
import json
import os
import re
import warnings
from datetime import datetime

warnings.filterwarnings("ignore", message="urllib3 v2 only supports OpenSSL.*")

import requests

BASE_URL = "https://www.v2ex.com"
DAILY_URL = f"{BASE_URL}/mission/daily"
COOKIE_FILE = "<PATH_TO_V2EX_COOKIE_JSON>"
TIMEOUT = 30


def build_session() -> requests.Session:
    with open(COOKIE_FILE, encoding="utf-8") as handle:
        raw = json.load(handle)
    cookies = raw.get("cookies", raw) if isinstance(raw, dict) else raw
    if not isinstance(cookies, list):
        raise RuntimeError("Cookie 文件格式无法识别")

    session = requests.Session()
    for cookie in cookies:
        if not isinstance(cookie, dict) or "name" not in cookie or "value" not in cookie:
            continue
        options = {}
        if cookie.get("domain"):
            options["domain"] = cookie["domain"].lstrip(".")
        if cookie.get("path"):
            options["path"] = cookie["path"]
        session.cookies.set(cookie["name"], cookie["value"], **options)

    session.headers.update(
        {
            "User-Agent": os.environ.get(
                "V2EX_USER_AGENT",
                "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) "
                "AppleWebKit/537.36 (KHTML, like Gecko) "
                "Chrome/150.0.0.0 Safari/537.36",
            ),
            "Accept": (
                "text/html,application/xhtml+xml,application/xml;q=0.9,"
                "image/avif,image/webp,image/apng,*/*;q=0.8"
            ),
            "Accept-Language": "zh-CN,zh;q=0.9,en;q=0.8",
            "Upgrade-Insecure-Requests": "1",
            "Sec-Fetch-Dest": "document",
            "Sec-Fetch-Mode": "navigate",
            "Sec-Fetch-Site": "same-origin",
            "Sec-Fetch-User": "?1",
        }
    )
    return session


def visible_text(document: str) -> str:
    text = re.sub(r"(?is)<(?:script|style)\b.*?</(?:script|style)>", " ", document)
    text = re.sub(r"(?s)<[^>]+>", " ", text)
    return re.sub(r"\s+", " ", html.unescape(text)).strip()


def page_info(document: str) -> dict:
    text = visible_text(document)
    redeem_match = re.search(
        r"location\.href\s*=\s*['\"]([^'\"]*/mission/daily/redeem\?once=[^'\"]+)['\"]",
        document,
    )
    streak_match = re.search(r"已连续登录\s*(\d+)\s*天", text)
    return {
        "loggedIn": "登出" in text,
        "signed": "每日登录奖励已领取" in text,
        "redeemPath": redeem_match.group(1) if redeem_match else None,
        "streakDays": int(streak_match.group(1)) if streak_match else None,
        "browserWarning": "奇奇怪怪的设置" in text,
    }


def main() -> int:
    now = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    session = build_session()
    response = session.get(DAILY_URL, timeout=TIMEOUT)
    response.raise_for_status()
    info = page_info(response.text)

    if not info["loggedIn"]:
        print("V2EX 签到失败：Cookie 已失效，需要重新导出登录态。")
        return 2
    if info["signed"]:
        print(f"V2EX 今日已签到，连续天数：{info['streakDays'] or '未知'}。{now}")
        return 0
    if not info["redeemPath"]:
        print("V2EX 签到失败：未找到领取链接。")
        return 3

    redeem_url = (
        info["redeemPath"]
        if info["redeemPath"].startswith("http")
        else f"{BASE_URL}{info['redeemPath']}"
    )
    redeem_response = session.get(
        redeem_url,
        headers={"Referer": DAILY_URL},
        timeout=TIMEOUT,
        allow_redirects=True,
    )
    redeem_response.raise_for_status()

    verify_response = session.get(
        DAILY_URL,
        headers={"Referer": redeem_response.url},
        timeout=TIMEOUT,
    )
    verify_response.raise_for_status()
    after = page_info(verify_response.text)

    if after["signed"]:
        print(f"V2EX 签到成功，连续天数：{after['streakDays'] or '未知'}。{now}")
        return 0
    if page_info(redeem_response.text)["browserWarning"] or after["browserWarning"]:
        print("V2EX 签到失败：V2EX 拒绝了当前请求环境。")
    else:
        print("V2EX 签到失败：领取请求完成，但复查仍未显示已领取。")
    return 4


if __name__ == "__main__":
    try:
        raise SystemExit(main())
    except Exception as exc:
        print(f"V2EX 签到脚本异常：{exc}")
        raise SystemExit(1)
```
{% endcode %}

## 一点万象：抓包参数签名签到

### 思路

一点万象签到接口来自移动端 H5 请求。脚本复现请求中的字段组织方式：每次运行生成时间字段，按接口规则计算 `sign`，再用 `application/x-www-form-urlencoded` 提交签到。

核心流程：

1. 从本机配置读取 `mallNo`、`imei`、`deviceParams`、`params`、`token`、`cookie` 和签名密钥。
2. 生成当前毫秒级 `timestamp`、偏移后的 `t`、当前日期字符串。
3. 按接口规则构造待签名字段：排除 `sign`，按 key 排序，部分字段需要 URL decode 后参与签名。
4. 拼接 `key=value` 串并追加签名密钥，计算 MD5 得到 `sign`。
5. 组装 `application/x-www-form-urlencoded` 请求体。注意已经 percent-encoded 的字段不要再次 urlencode。
6. 用接近 H5 的请求头发送 `curl` 请求。
7. 解析 JSON 响应，输出成功、已签到、请求频繁或失败摘要。

{% code title="mixc-signin-core.sh" %}
```bash
#!/usr/bin/env bash
set -euo pipefail

# 按自己的环境替换这些配置。
MALL_NO="<MALL_NO>"
IMEI="<DEVICE_ID>"
DEVICE_PARAMS="<URL_ENCODED_DEVICE_PARAMS>"
TOKEN="<TOKEN>"
PARAMS="<URL_ENCODED_PARAMS>"
COOKIE="<COOKIE>"
SIGN_SECRET="<SIGN_SECRET>"

URL="https://app.mixcapp.com/mixc/gateway"
APP_ID="<APP_ID>"
PLATFORM="h5"
APP_VERSION="<APP_VERSION>"
OS_VERSION="<OS_VERSION>"
ACTION="mixc.app.memberSign.sign"
API_VERSION="1.0"
SWIMLANE="<SWIMLANE>"
USER_AGENT="<MOBILE_H5_USER_AGENT>"

body_file="$(mktemp)"
trap 'rm -f "$body_file"' EXIT

read -r post_body referer < <(python3 - <<'PY' "$ACTION" "$API_VERSION" "$APP_ID" "$APP_VERSION" "$DEVICE_PARAMS" "$IMEI" "$MALL_NO" "$OS_VERSION" "$PARAMS" "$PLATFORM" "$TOKEN" "$SIGN_SECRET" "$SWIMLANE"
import hashlib
import sys
import time
import urllib.parse
from datetime import datetime

action, api_version, app_id, app_version, device_params, imei, mall_no, os_version, params, platform, token, secret, swimlane = sys.argv[1:]
now_ms = int(time.time() * 1000)
ts = str(now_ms)
t = str(now_ms + 25)
date = datetime.now().strftime("%Y-%m-%d %H:%M:%S")

fields_for_sign = {
    "X-Mixc-Swimlane": swimlane,
    "action": action,
    "apiVersion": api_version,
    "appId": app_id,
    "appVersion": app_version,
    "date": date,
    "deviceParams": urllib.parse.unquote(device_params),
    "imei": imei,
    "mallNo": mall_no,
    "osVersion": os_version,
    "params": urllib.parse.unquote(params),
    "platform": platform,
    "t": t,
    "timestamp": ts,
    "token": token,
}

sign_src = "&".join(f"{key}={fields_for_sign[key]}" for key in sorted(fields_for_sign)) + f"&{secret}"
sign = hashlib.md5(sign_src.encode("utf-8")).hexdigest()

body_parts = [
    ("mallNo", mall_no),
    ("appId", app_id),
    ("platform", platform),
    ("imei", imei),
    ("appVersion", app_version),
    ("osVersion", os_version),
    ("action", action),
    ("apiVersion", api_version),
    ("timestamp", ts),
    ("deviceParams", device_params),
    ("X-Mixc-Swimlane", swimlane),
    ("t", t),
    ("date", urllib.parse.quote(date)),
    ("token", token),
    ("params", params),
    ("sign", sign),
]

post_body = "&".join(f"{key}={value}" for key, value in body_parts)
referer = f"https://app.mixcapp.com/m/m-{mall_no}/signIn?appVersion={app_version}&mallNo={mall_no}&timestamp={ts}&showWebNavigation=true&hideNativeNavigation=true"
print(post_body, referer)
PY
)

http_code="$({
  curl --location "$URL" \
    --header 'Accept: application/json, text/plain, */*' \
    --header 'Origin: https://app.mixcapp.com' \
    --header "User-Agent: ${USER_AGENT}" \
    --header 'Content-Type: application/x-www-form-urlencoded' \
    --header "Referer: ${referer}" \
    --header "Cookie: ${COOKIE}" \
    --data-raw "${post_body}" \
    --compressed \
    --silent --show-error \
    --max-time 60 \
    --output "$body_file" \
    --write-out '%{http_code}'
} 2>&1)" || {
  echo "一点万象自动签到请求失败：${http_code}"
  exit 1
}

python3 - <<'PY' "$http_code" "$body_file"
import json
import pathlib
import sys

http_code, body_path = sys.argv[1:]
body = pathlib.Path(body_path).read_text(errors="replace")

try:
    data = json.loads(body)
except Exception:
    print(f"一点万象自动签到失败：响应不是 JSON，HTTP {http_code}")
    sys.exit(1)

message = str(data.get("message") or data.get("msg") or "")

if not str(http_code).startswith("2"):
    print(f"一点万象自动签到失败：HTTP {http_code}，{message or '无错误信息'}")
    sys.exit(1)

if data.get("code") == 0 or data.get("success") is True:
    print("一点万象自动签到成功")
    sys.exit(0)

if "已签到" in message or "不可重复签到" in message:
    print("一点万象今日已签到")
    sys.exit(0)

if "请求频繁" in message:
    print("一点万象请求频繁：接口签名已通过，但需要稍后重试")
    sys.exit(0)

print(f"一点万象自动签到失败：{message or '接口未返回明确原因'}")
sys.exit(1)
PY
```
{% endcode %}

## 运行建议

* V2EX 推荐用 browser-use 独立自动化 Profile 完成人工登录和 cookie 导出，定时领取使用普通 HTTP 会话，避免依赖真实 Chrome Profile，也避开自动 Chromium 的请求环境限制。
* 如果脚本报告 V2EX 未登录，优先重新打开 `v2ex-auth` 手动登录并导出 cookie；如果登录有效但领取失败，应检查领取响应里的浏览器设置警告，而不是只看 HTTP 状态码或 `once` 是否变化。
* V2EX 签到只有在复查页面明确显示已领取时才算成功；访问过领取链接不等于成功。
* 一点万象方案依赖接口字段和签名规则，字段变化后需要重新核对请求结构。
* 定时任务日志保留结果摘要即可，方便推送通知，也避免把整段响应体刷进通知里。
