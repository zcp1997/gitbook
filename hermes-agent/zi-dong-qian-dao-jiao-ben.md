---
description: V2EX 与一点万象自动签到脚本的实现思路
icon: robot
---

# 自动签到脚本

这页记录两个在 Hermes Agent 中运行过的自动签到脚本：V2EX 使用浏览器会话领取每日奖励；一点万象按移动端 H5 接口规则生成签名并提交签到请求。代码示例保留核心流程，实际部署时把本机相关配置替换为自己的值。

## V2EX：浏览器会话签到

### 思路

V2EX 的每日奖励页会在已登录状态下返回一个带 `once` 参数的动态领取链接。最初的方案是复用本机 Chrome Profile 中已有的登录态；实际运行后发现，在 macOS 上这条路容易被两个问题卡住：Chrome 正在运行时 Profile 文件会被锁，Chrome 退出后自动化进程又可能因为系统隐私权限读不到真实用户数据目录。

更稳的做法是：先用 `browser-use` 的独立自动化 Profile 手动登录一次 V2EX，导出该自动化 Profile 的 cookie；定时脚本每次启动自己的浏览器 session，导入这份 cookie，再打开每日任务页领取奖励。这样不需要触碰真实 Chrome Profile，也不会受 Chrome 是否正在运行影响。

核心流程：

1. 关闭同名浏览器自动化 session，避免上次 cron 残留状态影响本次运行。
2. 打开 V2EX 首页，导入之前保存的 V2EX cookie。
3. 打开 `https://www.v2ex.com/mission/daily`。
4. 在页面内执行 JS，读取正文状态并从 HTML 提取 `/mission/daily/redeem?once=...`。
5. 如果页面未登录，提示重新在自动化窗口登录并刷新 cookie。
6. 如果页面显示已经领取，直接返回成功。
7. 如果未领取且找到 redeem 链接，用浏览器导航访问领取链接。
8. 重新打开每日页复查，确认页面显示已领取。
9. 失败时只输出状态摘要，避免把完整页面内容刷进日志。

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
不要在用户刚刚登录的同一个 session 上直接运行带 `close_session()` 的定时脚本：脚本开头通常会关闭同名 session 来清理状态，容易把刚登录好的窗口关掉。手动登录建议使用 `v2ex-auth` 这样的临时 session，定时任务使用 `v2ex-daily-signin` 这样的固定 session。
{% endhint %}

如果仍想复用真实 Chrome Profile，需要先确保 Chrome 完全退出，并且运行自动化的终端或服务进程有权限访问 Chrome 用户数据目录；否则会遇到 Profile lock 或 `Operation not permitted`。

{% code title="v2ex-signin-core.py" %}
```python
#!/usr/bin/env python3
from __future__ import annotations

import ast
import json
import subprocess
import time
from datetime import datetime

SESSION = "v2ex-daily-signin"
BASE_URL = "https://www.v2ex.com"
DAILY_URL = f"{BASE_URL}/mission/daily"
COOKIE_FILE = "<PATH_TO_V2EX_COOKIE_JSON>"


def bu_args(*args: str) -> list[str]:
    return ["browser-use", "--session", SESSION, *args]


def run(args: list[str], timeout: int = 120, check: bool = True) -> str:
    proc = subprocess.run(
        args,
        text=True,
        stdout=subprocess.PIPE,
        stderr=subprocess.STDOUT,
        timeout=timeout,
    )
    if check and proc.returncode != 0:
        raise RuntimeError(f"command failed ({proc.returncode}): {' '.join(args)}\n{proc.stdout.strip()}")
    return proc.stdout.strip()


def run_bu(args: list[str], timeout: int = 120, retries: int = 1) -> str:
    last_error: Exception | None = None
    for attempt in range(retries + 1):
        try:
            return run(args, timeout=timeout)
        except Exception as exc:
            last_error = exc
            if attempt >= retries:
                break
            run(bu_args("close"), timeout=60, check=False)
            time.sleep(2)
    raise last_error or RuntimeError("browser-use command failed")


def parse_browser_use_result(output: str):
    text = output.strip()
    if text.startswith("result:"):
        text = text.split("result:", 1)[1].strip()
    try:
        return json.loads(text)
    except json.JSONDecodeError:
        return ast.literal_eval(text)


def page_info() -> dict:
    js = r"""
(() => {
  const html = document.documentElement.innerHTML;
  const text = document.body ? document.body.innerText : '';
  const redeemMatch = html.match(/location\.href\s*=\s*['"]([^'"]*\/mission\/daily\/redeem\?once=[^'"]+)['"]/);
  const streakMatch = text.match(/已连续登录\s*(\d+)\s*天/);
  const claimedMatch = text.match(/已成功领取每日登录奖励\s*([^\n]+)/);
  return JSON.stringify({
    url: location.href,
    title: document.title,
    loggedIn: text.includes('登出'),
    signed: text.includes('每日登录奖励已领取'),
    redeemPath: redeemMatch ? redeemMatch[1] : null,
    streakDays: streakMatch ? Number(streakMatch[1]) : null,
    claimed: claimedMatch ? claimedMatch[1].trim() : null,
  });
})()
""".strip()
    return parse_browser_use_result(run_bu(bu_args("eval", js), timeout=60))


def main() -> int:
    now = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    run(bu_args("close"), timeout=60, check=False)
    try:
        run_bu(bu_args("open", BASE_URL), timeout=180)
        run_bu(bu_args("cookies", "import", COOKIE_FILE), timeout=60)
        run_bu(bu_args("open", DAILY_URL), timeout=180)
        info = page_info()

        if not info.get("loggedIn"):
            print("V2EX 签到失败：自动化 Profile 未登录或 cookie 已失效。")
            return 2

        if info.get("signed"):
            print(f"V2EX 今日已签到，连续天数：{info.get('streakDays', '未知')}。{now}")
            return 0

        redeem_path = info.get("redeemPath")
        if not redeem_path:
            print("V2EX 签到失败：未找到领取链接。")
            return 3

        redeem_url = redeem_path if redeem_path.startswith("http") else f"{BASE_URL}{redeem_path}"
        run_bu(bu_args("open", redeem_url), timeout=180)
        run_bu(bu_args("open", DAILY_URL), timeout=180)
        after = page_info()

        if after.get("signed"):
            print(f"V2EX 签到成功，连续天数：{after.get('streakDays', '未知')}。{now}")
            return 0

        print("V2EX 签到失败：访问领取链接后复查仍未显示已领取。")
        return 4
    finally:
        run(bu_args("close"), timeout=60, check=False)


if __name__ == "__main__":
    raise SystemExit(main())
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

* V2EX 推荐使用 browser-use 独立自动化 Profile + cookie 导入，避免依赖真实 Chrome Profile。
* 如果脚本报告 V2EX 未登录，优先重新打开 `v2ex-auth` 手动登录并导出 cookie，不要先怀疑 redeem 逻辑。
* 一点万象方案依赖接口字段和签名规则，字段变化后需要重新核对请求结构。
* 定时任务日志保留结果摘要即可，方便推送通知，也避免把整段响应体刷进通知里。
