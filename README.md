# WeirdHost 自动续期（代理版）

自动续期 [hub.weirdhost.xyz](https://hub.weirdhost.xyz) 免费服务器/VPS，每天定时运行一次，结果推送到 Telegram。
基于 SeleniumBase 浏览器自动化 + sing-box 代理出口，支持多账号、多代理节点轮换重试、Cloudflare 验证自动处理、Cookie 自愈回写。

> ⚠️ 免责声明：本项目用于自动化保活，可能违反 WeirdHost 服务条款，有封号风险，请自行评估。

---

## 快速部署（5 步，约 10 分钟）

### 1. Fork 本仓库

点右上角 **Fork**。Fork 后进入你仓库的 **Actions** 标签页，如果提示工作流被禁用，点 **Enable** 启用。

### 2. 配置 Secrets（全部必配项都在这里）

进入你仓库的 **Settings → Secrets and variables → Actions → New repository secret**，逐个添加：

| Secret 名称 | 值 | 必需 | 说明 |
|---|---|---|---|
| `WEIRDHOST_COOKIE_1` | 登录 Cookie（获取方法见下） | ✅ | 最多支持 5 个账号：`WEIRDHOST_COOKIE_1` ~ `_5` |
| `PROXY_URL` | 代理节点链接 | 强烈建议 | 不配则直连（GitHub 机房 IP 大概率被 Cloudflare 拦截死循环） |
| `TG_BOT_TOKEN` | Telegram Bot Token | 可选 | 找 [@BotFather](https://t.me/BotFather) 创建 bot 获取 |
| `TG_CHAT_ID` | Telegram 数字 ID | 可选 | 找 [@userinfobot](https://t.me/userinfobot) 发消息获取 |
| `REPO_TOKEN` | GitHub Token | 建议 | 配好后 Cookie 被站点轮换时**自动写回 Secrets**，永久免维护 |

**获取 WeirdHost Cookie：**

1. 浏览器登录 https://hub.weirdhost.xyz
2. 按 `F12` → **Application（应用）** → 左侧 **Cookie**（点前面的小三角展开）→ 点 `https://hub.weirdhost.xyz`
3. 找到 **名称以 `remember_web_` 开头**的那一行（HttpOnly 为 ✓）
4. 复制**完整名称和值**，拼成一行填入 Secret，格式：
   ```
   备注-----remember_web_xxxxxxxx=eyJpdiI6....
   ```
   （备注可省略，直接 `remember_web_xxx=yyy` 也行）

> ⚠️ **重要**：复制完 Cookie 后**不要再登录** hub.weirdhost.xyz（任何设备/浏览器都算）——每登录一次，旧 Cookie 立即作废。日常使用不受影响，脚本自己用 Cookie 登录不会触发作废。

**获取 REPO_TOKEN：**

用 GitHub 个人访问令牌（PAT），需要 **repo 权限**（经典 PAT 勾选 `repo`；或 Fine-grained PAT 只授权本仓库 + Secrets 写权限）。作用：脚本登录后发现站点轮换了 Cookie，会用它把新 Cookie 自动写回上面的 Secret，实现自愈。

### 3. 代理节点格式（PROXY_URL）

支持：`vmess://` `vless://` `trojan://` `hy2://` `socks5://` `http(s)://` `tuic://` `anytls://`

从你的代理软件（Clash / v2rayN 等）右键节点 → 分享/导出链接，直接粘贴即可。**多个节点用换行分隔**，运行失败会自动换下一个节点重试。

节点选择建议（Cloudflare 对 IP 也评分）：
- 🏠 住宅/家庭宽带节点效果最好
- 🖥️ 自己的 VPS 次之
- ✈️ 机场节点优先选冷门的；临时隧道类（如 trycloudflare）随时会失效，失效后换一个更新 Secret 即可

### 4. 运行

- **自动**：每天 **UTC 4:20（北京时间 12:20）** 自动运行（GitHub 定时可能有几分钟延迟）
- **手动**：Actions → Weirdhost 多账号自动续期 → Run workflow

### 5. 看结果

- 成功/冷却/失败都会推送到 Telegram（配了 TG 的话）
- 详细日志在 Actions 运行记录里；每次运行上传调试截图（Artifact），代理模式下还包含 `singbox.log` 和 `config.json`

**日志关键行对照：**

```
✅ 代理可用                          ← 代理连通性测试通过
[INFO] 🔗 代理模式: http://127.0.0.1:8080
[INFO] ✅ CF 验证通过                 ← Cloudflare 验证
[INFO]   登录成功，已获取会话 Cookie    ← Cookie 有效
[INFO] 找到 N 个服务器
[INFO] 续期按钮已禁用（可能在冷却期）   ← 还没到续期时间，正常
  🟢 xxx | 1 个服务器 | success       ← 续期成功
  🟡 xxx | cooldown                  ← 冷却期内，等下次
  🔒 xxx | cookie_invalid            ← Cookie 失效，需重新抓
```

---

## 常见问题

| 现象 | 原因与解决 |
|---|---|
| 日志反复出现 "Cloudflare 验证页未完成" | GitHub 机房 IP 被 CF 风控死循环，**必须配代理**；已配代理则说明节点 IP 也被标记，换节点 |
| `cookie_invalid` | Cookie 失效：重新抓一次（注意抓完别再登录）；配了 `REPO_TOKEN` 则之后自动续 |
| 代理健康检查失败 `❌ 代理连接失败` | 节点挂了/参数不支持，先在本地代理软件确认节点可用，再更新 `PROXY_URL` |
| 续期按钮禁用 | 正常，免费服务器一般要临近到期（最后几天）才能续，脚本每天跑不会错过窗口 |
| 收不到 Telegram 通知 | 检查 `TG_BOT_TOKEN`/`TG_CHAT_ID`；先给 bot 发一条消息再触发 |

## 文件结构

```
├── .github/workflows/Weirdhost_renew.yml   # 定时任务编排（含 sing-box 启动与健康检查）
├── scripts/weirdhost_renew.py              # 续期主逻辑（登录/验证码/续期/通知/回写）
├── proxy_handler.py                        # 解析 PROXY_URL 生成 sing-box 配置（支持多节点、ws 早期数据）
└── README.md
```
