# Senflare Proxy Test

Senflare Proxy Test —— Cloudflare ProxyIP 聚合 / 测试脚本 —— 多源汇聚, GitHub Actions 每小时自动更新结果


🌐 **主站**：<https://proxy.seeck.cn/> ｜ **备用**：<https://proxy-vercel.seeck.cn/>


## ✨ 工作流程

```
拉取多源 → 组内去重
  ├─ 只拉取组 ─ 直接汇入（上游已验证）
  └─ 需测试组 → ① TCP 存活测试 → ② HTTP 验证 → 通过者写入
                    ↓
        ③ 地区补全（缓存优先，纯 IP 统一格式）→ 合并去重 → Senflare-Proxy.txt
```

| 层 | 说明 |
|---|---|
| ① TCP 存活 | socket 直连测延迟，成功率不达标直接淘汰（300 并发） |
| ② HTTP 验证 | `HEAD /cdn-cgi/trace` 必须返回 400 且 `server: cloudflare*`，过滤假节点，采样计算延迟/抖动（128 并发） |
| ③ 地区补全 | 无地区码的节点调 [ipinfo.io lite](https://ipinfo.io) 补齐；本地 LRU 缓存 1 万条，重复 IP 不重复查询 |

## 📡 数据来源

[Xiaobei09](https://github.com/Xiaobei09) · [Cmliu](https://github.com/cmliu) · [Wentao883](https://github.com/wentao883) · [ChatBotPlus](https://github.com/ChatBotPlus) · [Ymyuuu](https://github.com/ymyuuu) · [Mountain787](https://github.com/mountain787) · [Fangsia Karlina](https://github.com/papapapapdelesia) · [Xgonce](https://github.com/xgonce) · [Xinyitang3](https://github.com/xinyitang3)

## 📤 输出格式

`Senflare-Proxy.txt`，每行一个节点，统一 `IP:端口#地区码`：

```
132.226.157.11:443#US
193.108.112.65:443#AL
157.22.240.45:8443#AR
```

免测组在前，测试通过的按 HTTP 延迟升序在后，跨组按 `ip:port` 去重
兼容解析：干净标签、emoji 国旗富标签、纯 IP（默认 443 端口）

## 💻 本地运行

零第三方依赖，Python 3.8+ 直接跑：

```bash
python Start.py
```

常用参数在脚本头部常量区，自己看代码，不多做介绍。

## 🤖 GitHub Actions

仓库自带 [`.github/workflows/run.yml`](.github/workflows/run.yml)：

- ⏰ 每 3 小时自动运行一次（UTC 错峰），支持手动触发
- 💾 运行结束自动提交 `Senflare-Proxy.txt` 与地区缓存回仓库
- 🔁 带 concurrency 防重入，无变化跳过提交

## 🙏 致谢

数据均来自各位作者的持续验证与分享，感谢。





这份方案梳理了从 **Cloudflare 边缘拦截排查** 到 **Python 脚本鉴权同步** 的全套成功经验，方便你随时查阅或复用。

---

## ☁️ Cloudflare 侧配置要点

1. **WAF 自定义规则（放行特定路径）**
* 进入 Cloudflare 后台，选择你的域名，导航至 **Security** -> **Security rules** (或 WAF -> Custom rules)。
* 创建一条自定义规则（例如命名为 `Bypass-WeChat-Bot`）。
* **表达式设置**：点击 **Edit expression** 并直接写入以下代码：
```text
(http.request.uri.path == "/wxsend")

```


* **动作（Action）**：选择 **Skip**（跳过），并勾选 **Super Bot Fight Mode Rules** 相关组件以允许自动化请求通过。


2. **DNS 解析管理**
* 优选 IP 的解析记录（如 `us.proxyip.p30.kdns.fr`、`all.proxyip.p30.kdns.fr` 等）保持 **`DNS only`（仅 DNS，灰色云朵）** 或根据需要配置代理。
* 微信推送接口所在的 Worker 路由确保已通过 WAF 的放行规则。



---

## 🐍 Python 脚本端核心代码实现

在 `Start.py` 的微信推送函数中，必须**将 Token 作为 URL 参数拼接**，并补全标准的浏览器 `User-Agent` 与请求头，以绕过数据中心 IP 的默认阻断：

```python
def send_wechat_notification(msg):
    if not WECHAT_API_URL:
        return
    try:
        # 1. 将 Token 拼接到 URL 参数中（确保与 curl 请求形式完全一致）
        api_url = WECHAT_API_URL
        if WECHAT_AUTH_TOKEN:
            delimiter = '&' if '?' in api_url else '?'
            api_url = f"{api_url}{delimiter}token={WECHAT_AUTH_TOKEN}"

        # 2. 组装标准的 JSON 请求体
        body_dict = {
            "title": "Proxyip 优选",
            "content": msg
        }
        data_bytes = json.dumps(body_dict, ensure_ascii=False).encode('utf-8')
        
        # 3. 模拟完整的标准请求头，防止被识别为简单脚本拦截
        headers = {
            'Content-Type': 'application/json',
            'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/125.0.0.0 Safari/537.36',
            'Accept': '*/*',
            'Accept-Encoding': 'gzip, deflate'
        }
        
        req = urllib.request.Request(api_url, data=data_bytes, headers=headers, method='POST')
        with urllib.request.urlopen(req, timeout=10):
            print("📱 微信通知发送成功")
    except Exception as e:
        print(f"⚠️ 微信通知发送失败: {e}")

```

---

## 🧪 本地排查与验证方法

如果在 GitHub Actions 中遇到 `403 Forbidden`，可以先在本地或终端通过 `curl` 进行连通性排查：

```bash
curl -X POST "https://wx.djcf.pp.ua/wxsend?token=你的Token" \
     -H "Content-Type: application/json" \
     -d '{"title":"测试通知","content":"这是一条来自本地的测试消息"}'

```

* **返回 `ok` / 成功提示**：说明接口和鉴权无误，只需对齐 Python 脚本中的 Headers 与 URL 传参方式即可。
* **返回 `403**`：说明 Cloudflare 规则未生效或 Worker 内部代码存在更严格的白名单/IP 限制。

---

# Senflare Proxy Test —— 功能说明与 GitHub Actions 变量配置指南

本项目是一个基于 Python 标准库实现的 Cloudflare ProxyIP 聚合、存活测试、地区分类、DNS 自动同步以及多渠道消息通知的自动化工具。以下是该脚本的核心功能特性说明以及如何在 GitHub Actions 中配置相关变量的完整指南。

---

## 一、 核心功能特性

* **多源汇聚与去重**：支持拉取多个主流优选 IP 数据源（包含免测直入的聚合源及需要经过本地测试的源），并自动完成组内及最终结果的去重。


* **双重节点验证**：
1. **TCP 存活测试**：对候选节点进行多线程高并发连通性及延迟测试，过滤失效节点。


2. **HTTP 真 CF 验证**：通过请求 `/cdn-cgi/trace` 并校验响应头与状态码，确保节点确实具备 Cloudflare 代理加速能力。




* **智能地区补全**：无地区标识的节点可通过本地缓存或高效率 API 自动补全国家/地区码（如 `us`、`jp`、`sg` 等）。


* **Cloudflare DNS 自动化同步**：将优选出的优质 IP 按照指定地区（如 `jp`、`kr`、`sg`、`us`、`ca`）以及一个总聚合 `all` 域名，自动同步更新到 Cloudflare DNS 解析记录中，并可限制每个子域名的写入数量（如各地区 5 个、all 组 20 个），保持后台整洁。


* **双通道消息推送**：支持将每次优选与 DNS 同步的统计报告通过**微信**和 **Telegram** 实时推送通知。



---

## 二、 GitHub Actions 环境变量配置指南

为了保证安全性并实现灵活配置，脚本中的各项鉴权参数和目标 ID 均已适配 GitHub 仓库的 **Secrets** 环境变量。

### 1. 配置步骤

1. 进入你的 GitHub 仓库，点击 **Settings**（设置） -> **Secrets and variables** -> **Actions**。
2. 点击 **New repository secret**，依次添加下方列出的各项变量。

### 2. 必需与可选的环境变量说明

| 环境变量名 | 说明 | 示例值 |
| --- | --- | --- |
| `CF_ZONE_ID` | 你的 Cloudflare 域名 Zone ID | `14f4e6bb43c8...`<br> |
| `CF_API_TOKEN` | 具有 `Zone.DNS (Edit)` 权限的 Cloudflare API 令牌 | `your_cf_api_token_here` |
| `WECHAT_API_URL` | 微信推送接口地址（支持自建 Worker 或 Server酱） | `[https://wx.djcf.pp.ua/wxsend](https://wx.djcf.pp.ua/wxsend)` |
| `WECHAT_AUTH_TOKEN` | 微信接口鉴权密码/Token | `sb123`<br> |
| `TG_BOT_TOKEN` | Telegram Bot 机器人 Token | `123456789:ABCdef...` |
| `TG_CHAT_ID` | Telegram 接收通知的聊天/频道 ID | `-100123456789` |

---

## 三、 GitHub Actions 工作流（Workflow）配置示例

在你的仓库 `.github/workflows/` 目录下创建一个 YAML 文件（例如 `proxy-test.yml`），并在其中通过 `env` 将上述 Secrets 注入到脚本运行环境中：

```yaml
name: Senflare Proxy Test

on:
  schedule:
    - cron: '0 */6 * * *' # 每 6 小时自动运行一次
  workflow_dispatch:      # 支持手动触发

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.10'

      - name: Run Proxy Test Script
        env:
          CF_ZONE_ID: ${{ secrets.CF_ZONE_ID }}
          CF_API_TOKEN: ${{ secrets.CF_API_TOKEN }}
          WECHAT_API_URL: ${{ secrets.WECHAT_API_URL }}
          WECHAT_AUTH_TOKEN: ${{ secrets.WECHAT_AUTH_TOKEN }}
          TG_BOT_TOKEN: ${{ secrets.TG_BOT_TOKEN }}
          TG_CHAT_ID: ${{ secrets.TG_CHAT_ID }}
        run: |
          python Start.py

      - name: Commit and push updated proxy list
        uses: stefanzweifel/git-auto-commit-action@v5
        with:
          commit_message: "Auto: Update Senflare-Proxy.txt & DNS records"
          file_pattern: "Senflare-Proxy.txt Senflare-Country.json"

```
