---
name: "whoop-oauth"
description: "Connect an AI agent to Whoop via OAuth2: dashboard app setup, the exact working parameters, the offline-scope fix for the 1-hour token death, and troubleshooting. Agent-agnostic (Muse / GPTs / Claude); Muse-connector specifics are marked [Muse]. Use when setting up Whoop access from scratch."
---

# whoop-oauth：AI Agent 接入 Whoop 的完整方法论

让 AI Agent 获得 Whoop 健康数据（恢复 / 睡眠 / 训练 / 消耗）**长期**访问权限的完整流程。
沉淀自 2026-10-09 / 10-10 / 10-12 三次实战接入，包含两个官方文档没写清楚的坑。

## 适用谁

- Meta Muse、OpenAI GPTs、Claude、任何支持 OAuth2 authorization-code 流程的 AI Agent
- 文中标 **[Muse]** 的是 Muse 专属（connector 机制），其他 Agent 用各自的 OAuth 接入方式替换
- **Whoop 侧的所有参数完全通用**，换 Agent 不用改

## 前置条件

1. Whoop 账号（手环会员）
2. 能打开 Whoop Developer Dashboard（https://developer.whoop.com）建应用
3. 一个公网可访问的隐私政策页面（Whoop 应用要求；个人自用写个简单的中英双语页就行）

## Step 1：后台建应用（手动，5 分钟）

1. 登录 Dashboard → 创建应用
2. Redirect URI：填你所用 Agent 的回调地址
   - [Muse] `https://agent.meta.ai/api/hatch/oauth/callback`
3. 记下 Client ID 和 Client Secret
4. Scope 勾选：6 个 read 权限全勾上
   `read:recovery` `read:cycles` `read:sleep` `read:workout` `read:profile` `read:body_measurement`
5. ⚠️ 后台**没有 `offline` 选项**——这是正常的，不是你的账号问题，见 Step 3

## Step 2：OAuth 参数（实测走通，不要改）

| 参数 | 值 |
|---|---|
| authorization endpoint | `https://api.prod.whoop.com/oauth/oauth2/auth` |
| token endpoint | `https://api.prod.whoop.com/oauth/oauth2/token` |
| token_endpoint_auth_method | `client_secret_post` ← **关键！** 默认的 `client_secret_basic` 会 401 |
| issuer domain | `whoop.com` |
| API host | `api.prod.whoop.com` |
| scopes | `offline` + Step 1 的 6 个 read |

[Muse] 用 `credentials.request_api_access` 建 custom connector（`provider=whoop`），按上表填。

## Step 3：`offline` scope —— "1 小时必死"的解药 ⭐

这是整个接入唯一真正的坑：

- Whoop access token 寿命只有 **1 小时**（官方设计，不是 bug）
- 授权时**不带** `offline` → 拿不到 refresh token → 1 小时后必 401，只能手动重连
- 授权时**带** `offline` → 拿到 refresh token → 自动续期，一劳永逸
- Dashboard 没有 `offline` 勾选项？**直接在授权 URL 的 scope 参数里加上 `offline`，Whoop 照样接受**
  （2026-10-12 实测：授权页正常加载、无报错、完整流程走通）

## Step 4：授权流程（用户手动，2 分钟）

1. 打开授权链接 → 填 Client ID / Secret
   - 注意：Whoop 表单**不校验** Secret，输错不会当时报错，之后直接 401——排错时先查这个
2. Whoop 账号登录 → 确认授权
3. 回到 Agent，调 `/user/profile/basic`（或等价 whoami）验证连通

## Step 5：验证自动续期（决定性测试，别跳过）

- 授权完成后立刻能通，只证明 access token 有效，**证明不了续期**
- 真正的验证：**等 70+ 分钟**（超过 1 小时寿命）再调一次
  - 通 → refresh 在工作，永久解决 ✅
  - 401 → `offline` 没生效，回 Step 3 排查
- [Muse] token 续期归 hatch OAuth 通道管，CLI/skill 里永远不要自己写 refresh 逻辑

## 故障排查速查

| 现象 | 真因 | 解法 |
|---|---|---|
| 授权页 "Missing required parameters" | 99% 是授权链接 session 过期，**不是** scope 非法 | 关掉重开一条新链接再走一遍 |
| token 接口 401 | auth method 用错（basic）或 Secret 输错 | 换 `client_secret_post`；重对 Secret |
| 1 小时后 401 | 当初授权没带 `offline` | 带 `offline` 重走授权 |
| collection 接口 404 | 该时间段无数据，**不是报错** | 按空结果处理 |
| recovery / sleep / workout 全空 | 手环晚上没戴 | 以用户体感为准，别硬等数据 |
| webhook 相关报错 | 个人自用不需要 webhook | 直接忽略 |

## 连通后的 API 要点

- base `https://api.prod.whoop.com/developer/v1`
- `/user/profile/basic`、`/user/body_measurement`、`/cycle`、`/recovery`、`/sleep`、`/workout`
- **cycle 是生理日**（睡到睡），会跨自然日；"今天"的 cycle 可能还没生成，取不到别慌
- collection 分页 limit 最大 25，用 `nextToken` 翻页

## 实测状态（更新至 2026-10-12）

- 10-09：`client_secret_post` + 6 scope（无 offline）走通；确认 1 小时过期机制
- 10-12：带 `offline` 的授权走通，授权页接受 `offline`；**自动续期已验证通过**（当日 15:10，token 发放 80+ 分钟后 `whoami` 依然成功，refresh token 在工作）✅
