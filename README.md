# OptikLink-M

> OptikLink 面板的自动签到 + 服务器保活，跑在 GitHub Actions 上，躺平不用管。

## 它做什么

每 3 天一次（`cron: '0 1 */3 * *'`，UTC 01:00），这个 workflow 自动：

1. **通过 Discord OAuth 登录 OptikLink 面板** — 用你的 Discord 令牌完成授权
2. **过 Cloudflare Turnstile 验证** — OptikLink 新增的 Quick Verification 步骤（随机数学题 + Turnstile 复选框），用真实有头浏览器过掉
3. **签到确认** — 检查 Dashboard 页面，提取登录状态、用户名、到期时间
4. **服务器保活** — 面板服务器（Pterodactyl）处于离线状态时自动发送启动指令
5. **推送报告** — 登录结果通过 Telegram Bot 推送给你
6. **自动更新 Client ID** — 如果 OptikLink 更换了 Discord OAuth 的 `client_id`，脚本会自动探测并更新 GitHub Secret（省得你手动改）

## 文件结构

| 文件 | 用途 |
|---|---|
| `optiklink_login.py` | 主脚本：登录、过 Turnstile、签到、服务器保活、TG 推送 |
| `generate_xray_config.py` | 从 VLESS 链接生成 Xray 客户端配置（用于代理出口） |
| `test_discord.py` | 调试工具：测试 Discord API 授权参数是否有效 |
| `time.txt` | 每次运行自动更新时间戳，保持仓库活跃 |
| `.github/workflows/optiklink.yml` | GitHub Actions 工作流定义 |

## 登录流程是怎么实现的

OptikLink 的 `/login?code=` 现在不再直接签发会话，而是先弹一个 **Quick Verification** 页面：一道随机数学题（`<strong>7 + 8</strong>`）、一个 Cloudflare Turnstile（managed 模式，`data-action="login"`）、一个 CONTINUE 按钮。两个都过才放行。

### 为什么必须开浏览器

Turnstile 的令牌由 `api.js` 生成后写进隐藏域 `input[name="cf-turnstile-response"]`，OptikLink 服务端会校验这个值。三条路实测：

| 方案 | 结果 |
|---|---|
| 纯 HTTP（`cloudscraper` / `requests`） | ❌ 服务端校验令牌，造不出来 |
| 无头 / CDP 自动化（puppeteer + `--disable-blink-features=AutomationControlled` + 伪装 `navigator.webdriver`，等 32 秒） | ❌ 令牌始终为空，与代理 IP 无关 —— 是自动化指纹被识别 |
| **SeleniumBase UC 模式（有头 Chrome + OS 级点击）** | ✅ 令牌正常生成，GitHub runner 上实测 9 秒出令牌 |

关键在点击方式：Turnstile 要的是操作系统级鼠标事件。`sb.uc_gui_click_captcha()` 走 pyautogui 发真实鼠标点击，不是在浏览器里派发合成事件，Cloudflare 分辨不出来。

### 四段式流程

1. **[A] 探测 OAuth 参数**（HTTP）— 抓 `/auth` 页面里的 Discord authorize 链接，解析出 `client_id` / `redirect_uri` / `scope`。探测不到就用硬编码后备值；探测到且和 Secret 不一致时写进 `$GITHUB_OUTPUT`，让后续步骤更新 Secret。
2. **[B] Discord 授权**（HTTP）— `POST https://discord.com/api/v10/oauth2/authorize`，`Authorization` 头带用户令牌，body 是 `{"authorize": true, "permissions": "0"}`，返回的 `location` 里带 `code=`。
3. **[C] 浏览器过验证**（SeleniumBase UC 模式）—
   - `uc_open_with_reconnect` 打开回调链接（页面被 Cloudflare 断开时自动重连）
   - 正则从页面解析数学题 → `solve_math` 算答案 → 填入 `input[name="math_answer"]`
   - `scrollIntoView` 把 Turnstile 滚到可视区 → `uc_gui_click_captcha()` 点击
   - 轮询 `cf-turnstile-response` 的值，长度 > 50 视为令牌就绪（实测 794 字符）
   - 点提交 → 落到真正的 Dashboard，检查最终 URL 里没有 `/error/`
4. **[D] Dashboard + 保活**（HTTP）— 用拿到的页面 HTML 正则提取用户名 / 服务器数 / 到期日期；再用 `PANEL_API_KEY` 走 Pterodactyl API 查服务器状态，offline 就发 `start` 信号并轮询到它起来。这一段和登录完全独立，登录挂了它也不会受影响。

最后 `build_report` 拼成 Markdown 推给 Telegram。

### CI 上怎么跑有头浏览器

GitHub 托管 runner 没有显示器，但有头 Chrome 必须要显示服务。workflow 里用 `xvfb-run -a --server-args="-screen 0 1920x1080x24"` 起一个虚拟 framebuffer，Chrome 挂上去就当自己有屏幕。

代价：WebGL 走 SwiftShader 软件渲染，这是个机器人信号。目前实测依然能过，但如果哪天开始大面积失败，这是第一个要怀疑的点。

## 部署

### 1. Fork 这个仓库

点右上角 Fork。

### 2. 配置 Secrets

去 Settings → Secrets and variables → Actions → New repository secret，添加以下内容：

| Secret | 说明 |
|---|---|
| `DISCORD_TOKEN` | 你的 Discord 用户令牌（`Authorization` header 值，`mfa.xxx` 格式） |
| `BOT_TOKEN` | Telegram Bot Token（从 [@BotFather](https://t.me/BotFather) 获取） |
| `CHAT_ID` | 接收推送的 Telegram 用户/群组 ID |
| `VLESS_NODE` | VLESS 链接，用于 Xray 代理出口（绕过 OptikLink 的 IP/地区限制） |
| `PANEL_API_KEY` | （可选）Pterodactyl 面板 API Key，用于服务器保活 |
| `PANEL_SERVER_ID` | （可选）指定服务器 ID，不填则自动取第一个 |
| `EXPIRE_DATE` | （可选）手动指定到期日期作为后备 |
| `DISCORD_CLIENT_ID` | （可选）Discord OAuth client_id，脚本会尝试自动探测 |
| `DISCORD_REDIRECT_URI` | （可选）Discord OAuth 回调地址 |

### 3. 启动 Workflow

- 默认每 3 天运行一次（`cron: '0 1 */3 * *'`）
- 也可以手动触发：Actions → OptikLink 每日自动登录+服务器保活 → Run workflow

## 依赖

全部由 workflow 自动安装，本地不用装：

- Python 3.12+
- `requests` / `cloudscraper` — 第一阶段的 HTTP 登录
- `seleniumbase` — 第二阶段浏览器过 Turnstile（UC 模式，内含 undetected-chromedriver）
- `xvfb` + Chrome 运行库（`libnss3` `libdrm2` `libxkbcommon0` `libgbm1` `libasound2t64` 等）— 给有头 Chrome 一个虚拟显示
- `xray` — VLESS 代理出口（workflow 里现下载二进制）

## 已知问题

- **Turnstile 令牌偶发拿不到就直接失败，没有重试。** Cloudflare 的风险评分每次不同，偶尔一次拿不到令牌就整次运行失败（`Turnstile 令牌未获取`），等 cron 下次补。
- **`提交 time.txt` 步骤写死了 `git push origin HEAD:main`。** 在非 main 分支上手动 dispatch 时，它会尝试把那个分支的代码推到 main。目前靠 fast-forward 拒绝挡住，不炸，但这是个雷。

## 免责声明

这个项目仅供学习和个人自动化使用。使用 Discord 用户令牌进行 OAuth 授权可能违反 Discord 服务条款，请自行评估风险。
