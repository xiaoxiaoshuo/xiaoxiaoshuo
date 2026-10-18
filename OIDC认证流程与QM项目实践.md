# OIDC 认证流程与 QM 项目实践

> 目标：这份笔记既能帮你理解 OIDC 的真实工作流程，也能用于面试复述。重点记住：**OIDC 解决“你是谁”，OAuth2 解决“你能访问什么”；现代登录核心流程是：前端拿 code，后端 code 换 token，应用验证 id_token 后发自己的 session cookie。**

---

## 1. OIDC 是什么？

OIDC 全称：

```text
OpenID Connect
```

它是建立在 OAuth 2.0 之上的身份认证协议。

一句话：

> **OIDC 让应用可以通过可信身份提供方确认“当前用户是谁”。**

例如：

```text
Sign in with Google
Sign in with Microsoft
Sign in with Slack
企业 SSO 登录
```

这些背后经常就是 OIDC。

---

## 2. 谁制定的？

OIDC 不是某个公司单独发明的，而是由：

```text
OpenID Foundation
```

制定和维护的开放标准。

主要规范大约在 2014 年成熟。它继承了早期 OpenID 的思想，但使用 OAuth 2.0 作为底座，更适合现代 Web、移动端、API、企业 SSO。

---

## 3. 一句话总结 OIDC 解决了什么问题

最短版：

> **OIDC 标准化地回答：“这个用户是谁？”**

稍完整版：

> **应用不想自己保存密码，也不想自己验证身份，于是把登录交给可信身份提供商；身份提供商验证用户后，给应用一个标准化的用户身份证明。**

---

## 4. OAuth2 和 OIDC 的区别

很多人混淆 OAuth2 和 OIDC。

记忆法：

```text
OAuth2 = 授权 Authorization = 能不能访问资源
OIDC   = 认证 Authentication = 用户是谁
```

### OAuth2 例子

```text
某 App 请求访问你的 Google Drive 文件
```

重点是：

```text
这个 App 能访问哪些资源？
```

### OIDC 例子

```text
用 Google 账号登录某网站
```

重点是：

```text
当前用户是谁？邮箱是什么？Google 是否验证过？
```

OIDC 在 OAuth2 基础上增加了关键产物：

```text
id_token
```

`id_token` 通常是 JWT，里面包含用户身份声明。

例子：

```json
{
  "iss": "https://accounts.google.com",
  "sub": "1234567890",
  "email": "alice@gmail.com",
  "email_verified": true,
  "aud": "your-client-id",
  "exp": 1780000000
}
```

应用验证这个 `id_token` 后，就知道用户身份。

---

## 5. OIDC 的三个核心角色

| 角色 | 英文 | 作用 | Google 登录例子 | QM 项目例子 |
|---|---|---|---|---|
| 用户 | User / End User | 要登录的人 | 你 | 171232349@qq.com |
| 应用 | Client / Relying Party | 需要登录能力的网站 | example.com | QM Portal |
| 身份提供方 | Identity Provider / OpenID Provider | 负责验证用户并签发身份令牌 | Google | QM auth broker |

---

## 6. OIDC 最常见流程：Authorization Code Flow + PKCE

现代 Web 登录最常见流程是：

```text
Authorization Code Flow + PKCE
```

面试重点记住一句：

> **浏览器只拿 authorization code，真正的 token 由应用后端拿 code 去 token endpoint 换。**

---

## 7. 标准 OIDC 登录流程详解

假设网站是：

```text
https://example.com
```

身份提供方是 Google。

### 第 1 步：用户点击登录

用户点击：

```text
Sign in with Google
```

网站发现用户没有 session，于是发起 OIDC 登录。

---

### 第 2 步：应用生成登录参数

应用生成：

```text
client_id
redirect_uri
scope
state
nonce
code_challenge
code_challenge_method
```

常见 scope：

```text
openid email profile
```

其中：

| 参数 | 作用 |
|---|---|
| `client_id` | 应用在身份提供方注册得到的 ID |
| `redirect_uri` | 登录成功后跳回应用的地址 |
| `scope=openid email profile` | 请求 OIDC 登录、邮箱、用户资料 |
| `state` | 防 CSRF，回调时必须一致 |
| `nonce` | 防 id_token 重放，token 里要带回来 |
| `code_challenge` | PKCE 参数，防 code 被截获 |
| `code_challenge_method=S256` | 使用 SHA256 生成 challenge |

---

### 第 3 步：浏览器跳转到身份提供方

浏览器被重定向到：

```text
https://accounts.google.com/o/oauth2/v2/auth
  ?response_type=code
  &client_id=...
  &redirect_uri=https%3A%2F%2Fexample.com%2Fauth%2Fcallback
  &scope=openid%20email%20profile
  &state=...
  &nonce=...
  &code_challenge=...
  &code_challenge_method=S256
```

注意：

```text
用户密码只输入给 Google，不输入给 example.com。
```

---

### 第 4 步：身份提供方验证用户

Google 会验证用户：

```text
账号密码
二次验证
设备风险
账号状态
用户同意授权
```

如果用户已经登录 Google，可能只需要选账号。

---

### 第 5 步：身份提供方带 code 跳回应用

验证成功后，Google 重定向回：

```text
https://example.com/auth/callback?code=abc123&state=xyz
```

这里返回的是：

```text
authorization code
```

不是直接返回用户信息，也不是直接返回长期 token。

---

### 第 6 步：应用校验 state

应用检查：

```text
回来的 state 是否等于发起登录时保存的 state
```

如果不一致，拒绝。

目的：

```text
防 CSRF / 防登录请求被替换
```

---

### 第 7 步：后端用 code 换 token

应用后端向身份提供方发请求：

```text
POST https://oauth2.googleapis.com/token
```

带上：

```text
grant_type=authorization_code
code=abc123
client_id=...
client_secret=...
redirect_uri=...
code_verifier=...
```

这一步是面试高频点：

> **code 换 token。**

也可以记成：

```text
前端拿 code，后端换 token。
```

---

### 第 8 步：身份提供方返回 token

返回通常包括：

```json
{
  "access_token": "...",
  "expires_in": 3599,
  "scope": "openid email profile",
  "token_type": "Bearer",
  "id_token": "..."
}
```

其中 OIDC 最关键的是：

```text
id_token
```

`id_token` 证明用户是谁。

`access_token` 更偏 OAuth2，用于访问资源 API，例如 userinfo。

---

### 第 9 步：应用验证 id_token

应用必须验证 `id_token`：

```text
签名是否正确
issuer 是否正确
 audience 是否是自己的 client_id
是否过期
nonce 是否匹配
email 是否 verified
```

常见字段：

| 字段 | 含义 |
|---|---|
| `iss` | issuer，谁签发的 token |
| `sub` | subject，用户在该 IdP 下的唯一 ID |
| `aud` | audience，这个 token 发给哪个 client |
| `exp` | 过期时间 |
| `iat` | 签发时间 |
| `nonce` | 防重放随机值 |
| `email` | 用户邮箱 |
| `email_verified` | 邮箱是否已验证 |

Google 的公钥通过 JWKS 获取，例如：

```text
https://www.googleapis.com/oauth2/v3/certs
```

应用用 Google 公钥验证 `id_token` 确实是 Google 签发的。

---

### 第 10 步：应用创建自己的 session

Google / IdP 只是证明用户是谁。

应用最终通常会创建自己的 session：

```http
Set-Cookie: session=...
```

之后用户访问应用，靠应用自己的 cookie 保持登录状态。

重点：

```text
OIDC 登录成功后，不是每个请求都去问 Google。
应用会设置自己的 session cookie。
```

---

## 8. 面试速记版流程

### 8.1 最短流程

```text
未登录 → 跳 IdP → 用户登录 → 回调带 code → 后端 code 换 token → 验 id_token → 发自己的 cookie
```

### 8.2 八字口诀

```text
跳转授权，回调拿码；后端换令，验后发票。
```

解释：

```text
跳转授权 = 跳到 Google/IdP
回调拿码 = callback 带 authorization code
后端换令 = 后端用 code 换 token
验后发票 = 验 id_token 后发自己的 session cookie
```

### 8.3 重点句

> **OIDC 登录不是应用拿用户密码，而是应用信任身份提供方签发的 id_token。**

> **Authorization Code Flow 的关键是：浏览器拿 code，后端用 code 换 token。**

> **真正让用户在网站保持登录的，是网站自己的 session cookie，不是 Google 密码，也不一定是 Google token。**

---

## 9. state、nonce、PKCE 分别干什么？

| 名称 | 主要目的 | 防什么 |
|---|---|---|
| `state` | 绑定登录请求和回调 | CSRF、登录流程被替换 |
| `nonce` | 绑定 id_token 和本次认证 | id_token 重放 |
| `PKCE` | 绑定 code 和发起方 | authorization code 被截获 |

### state

应用发起登录时生成：

```text
state=random123
```

回调必须带回来：

```text
/auth/callback?code=...&state=random123
```

如果不一致，拒绝。

---

### nonce

应用发起登录时生成 nonce，身份提供方把 nonce 放进 `id_token`。

应用验证 `id_token` 时检查 nonce 是否一致。

目的：防别人拿旧的 `id_token` 重放。

---

### PKCE

PKCE 包含两个值：

```text
code_verifier
code_challenge
```

流程：

```text
应用生成 code_verifier
用 SHA256 得到 code_challenge
授权请求只发送 code_challenge
后端换 token 时提交 code_verifier
IdP 校验 code_verifier 是否能推导出 code_challenge
```

这样即使 authorization code 被截获，攻击者没有 `code_verifier`，也换不到 token。

记忆：

```text
PKCE = code 防盗锁
```

---

## 10. id_token、access_token、refresh_token 的区别

| Token | 用途 | 给谁用 |
|---|---|---|
| `id_token` | 证明用户是谁 | 应用验证登录身份 |
| `access_token` | 访问资源 API | 调 userinfo/API |
| `refresh_token` | 刷新 access token | 长期授权场景使用 |

面试常见回答：

> **OIDC 的核心是 id_token；OAuth2 的核心是 access_token。**

---

## 11. Google 登录原理

Google 登录其实就是 OIDC 的典型实现。

角色：

| OIDC 角色 | Google 登录中是谁 |
|---|---|
| User | 用户本人 |
| Client | 接入 Google 登录的网站 |
| Identity Provider | Google |

流程：

```text
用户点击 Sign in with Google
→ 网站跳转到 Google authorize endpoint
→ Google 验证账号密码/2FA
→ Google 回调网站 callback，带 authorization code
→ 网站后端用 code 调 Google token endpoint
→ Google 返回 id_token/access_token
→ 网站验证 id_token 签名和字段
→ 网站创建自己的 session cookie
→ 用户登录成功
```

Google 常见端点：

```text
Authorization endpoint:
https://accounts.google.com/o/oauth2/v2/auth

Token endpoint:
https://oauth2.googleapis.com/token

Userinfo endpoint:
https://openidconnect.googleapis.com/v1/userinfo

JWKS:
https://www.googleapis.com/oauth2/v3/certs

Issuer:
https://accounts.google.com
```

---

## 12. Google 登录为什么安全？

核心点：

1. 网站拿不到用户 Google 密码；
2. Google 只回调提前登记的 `redirect_uri`；
3. `state` 防 CSRF；
4. `nonce` 防 id_token 重放；
5. PKCE 防 code 被截获；
6. id_token 有签名，网站用 Google 公钥验证；
7. 网站自己建立 session，不长期依赖浏览器里的第三方 token。

---

## 13. QM 项目中的 OIDC 实现

当前 QM 项目不是接 Google，而是自己内置了一个 OIDC 身份提供方。

当前链路：

```text
Browser
  ↓
Portal :8097
  ↓
auth broker :8099
  ↓
Resend
  ↓
QQ 邮箱
```

---

## 14. QM 中的角色对应

| OIDC 角色 | QM 项目中是谁 |
|---|---|
| User | 171232349@qq.com |
| Client / Relying Party | Portal |
| Identity Provider | plugins/auth，也就是 auth broker |
| Mail service | Resend |
| Backend/Core | QM Core |

---

## 15. QM 登录流程详解

### 第 1 步：用户访问 Portal

用户打开：

```text
http://127.0.0.1:8097/
```

如果没有 session，Portal 要求登录。

登录入口：

```text
http://127.0.0.1:8097/auth/login?returnTo=/
```

---

### 第 2 步：Portal 发起 OIDC 请求

Portal 生成：

```text
state
nonce
code_verifier
code_challenge
```

然后把浏览器重定向到：

```text
http://127.0.0.1:8097/idp/authorize
```

注意这个地址是 Portal 的 8097，但 `/idp/*` 会被 Portal 代理到 auth broker：

```text
http://127.0.0.1:8099/authorize
```

---

### 第 3 步：auth broker 显示邮箱输入页

用户输入：

```text
171232349@qq.com
```

auth broker 检查白名单：

```env
AUTH_ALLOWED_EMAIL_DOMAIN=qq.com
```

因为邮箱属于 `qq.com`，所以允许发送登录链接。

---

### 第 4 步：auth broker 调 Resend 发邮件

配置：

```env
AUTH_EMAIL_TRANSPORT=resend
AUTH_EMAIL_FROM="QM <login@mail.lijingang.ccwu.cc>"
RESEND_API_KEY=...
```

邮件里包含一次性登录链接：

```text
http://127.0.0.1:8097/idp/verify#token=...
```

注意：token 在 `#` 后面。

---

###
`#token=...` 是 URL fragment，浏览器不会把它作为 HTTP 请求路径发给服务器，能减少 token 出现在服务端访问日志、代理日志、Referer 里的概率。

---

### 第 5 步：用户打开邮件链接并确认

用户点击邮件链接：

```text
http://127.0.0.1:8097/idp/verify#token=...
```

页面会显示：

```text
Finish signing in
Sign in
```

用户点 `Sign in` 后，页面把 fragment 里的 token 提交给 auth broker。

auth broker 验证：

```text
签名是否正确
是否过期
是否已经用过
邮箱是否仍然允许登录
client_id / redirect_uri 是否匹配
```

验证通过后，auth broker 生成 authorization code。

---

### 第 6 步：auth broker 回调 Portal

浏览器被重定向到：

```text
http://127.0.0.1:8097/auth/callback?code=...&state=...
```

Portal 检查：

```text
state 是否匹配
```

如果不匹配，说明这不是同一个浏览器发起的登录，或者可能存在攻击，登录失败。

这就是为什么邮件里提示：

```text
Open it in the browser you started from.
```

---

### 第 7 步：Portal 用 code 换 token

Portal 后端请求 auth broker：

```text
POST http://127.0.0.1:8099/token
```

提交：

```text
code
client_id
client_secret
redirect_uri
code_verifier
```

这里就是经典的：

> **code 换 token。**

---

### 第 8 步：auth broker 返回 id_token

返回内容包含：

```text
id_token
access_token
```

`id_token` 里包含用户身份：

```json
{
  "iss": "http://127.0.0.1:8097/idp",
  "sub": "...",
  "email": "171232349@qq.com",
  "email_verified": true,
  "aud": "qm-local"
}
```

---

### 第 9 步：Portal 验证 id_token

Portal 验证：

```text
issuer 是否等于 http://127.0.0.1:8097/idp
audience 是否等于 qm-local
签名是否正确
nonce 是否匹配
是否过期
邮箱是否属于 qq.com
```

Portal 通过 JWKS 获取 auth broker 的公钥：

```text
http://127.0.0.1:8099/.well-known/jwks.json
```

---

### 第 10 步：Portal 建立自己的 session

验证通过后，Portal 设置自己的 session cookie。

之后用户访问：

```text
http://127.0.0.1:8097/
```

Portal 就知道当前用户是：

```text
171232349@qq.com
```

然后 Portal 把请求代理到：

```text
Web UI
Admin
Core
```

---

## 16. QM 项目中 OIDC 相关组件

### 16.1 Portal

路径：

```text
plugins/portal
```

角色：

```text
OIDC Client / Relying Party
```

职责：

```text
/auth/login
/auth/callback
验证 id_token
管理 Portal session cookie
代理 /idp/* 到 auth broker
保护 Web UI / Admin / Core
```

关键文件：

```text
plugins/portal/src/index.ts
plugins/portal/src/oidc.ts
plugins/portal/src/session.ts
plugins/portal/src/proxy.ts
```

当前端口：

```text
http://127.0.0.1:8097
```

---

### 16.2 auth broker

路径：

```text
plugins/auth
```

角色：

```text
OIDC Identity Provider / OpenID Provider
```

职责：

```text
/authorize        显示邮箱登录页
/verify           验证邮件链接
/token            code 换 token
/userinfo         返回用户信息
/.well-known/jwks.json  提供公钥
发送一次性登录邮件
签发 id_token
```

关键文件：

```text
plugins/auth/src/index.ts
plugins/auth/src/server.ts
plugins/auth/src/config.ts
plugins/auth/src/tokens.ts
plugins/auth/src/email.ts
plugins/auth/src/keys.ts
```

当前端口：

```text
http://127.0.0.1:8099
```

---

### 16.3 Resend

Resend 不是 OIDC 组件，它只是邮件发送服务。

作用：

```text
把一次性登录链接发送到用户邮箱
```

当前配置：

```env
AUTH_EMAIL_TRANSPORT=resend
AUTH_EMAIL_FROM="QM <login@mail.lijingang.ccwu.cc>"
```

---

### 16.4 Core / PostgreSQL

Core 负责持久化一些安全状态，例如：

```text
登录链接是否已经使用
authorization code 是否已经使用
发邮件频率限制
admin grants
用户会话数据
runtime config
custom provider config
```

PostgreSQL 负责持久化。

---

## 17. QM 当前关键配置

Portal OIDC 配置：

```env
AUTH_BROKER_UPSTREAM=http://127.0.0.1:8099
AUTH_BROKER_PREFIX=/idp

OIDC_ISSUER=http://127.0.0.1:8097/idp
OIDC_AUTH_ENDPOINT=http://127.0.0.1:8097/idp/authorize
OIDC_TOKEN_ENDPOINT=http://127.0.0.1:8099/token
OIDC_USERINFO_ENDPOINT=http://127.0.0.1:8099/userinfo
OIDC_JWKS_URI=http://127.0.0.1:8099/.well-known/jwks.json
OIDC_CLIENT_ID=qm-local
OIDC_PRINCIPAL_CLAIM=email
OIDC_ALLOWED_EMAIL_DOMAIN=qq.com
```

auth broker 配置：

```env
AUTH_ISSUER=http://127.0.0.1:8097/idp
AUTH_CLIENT_ID=qm-local
AUTH_REDIRECT_URI=http://127.0.0.1:8097/auth/callback
AUTH_ALLOWED_EMAIL_DOMAIN=qq.com
AUTH_EMAIL_TRANSPORT=resend
AUTH_EMAIL_FROM="QM <login@mail.lijingang.ccwu.cc>"
```

注意：

```text
AUTH_ISSUER 是 http://127.0.0.1:8097/idp
```

虽然 auth broker 实际跑在 8099，但浏览器看到的 issuer 是 Portal 暴露出来的 `/idp` 地址。

---

## 18. Cookie 清了以后怎么办？

Portal 登录成功后会设置自己的 session cookie。

如果 cookie 被清除：

```text
Portal 不再知道用户是谁
需要重新登录
```

当前 QM 使用邮箱一次性登录，所以需要：

```text
访问 /auth/login
输入邮箱
重新收邮件
点击最新的一次性链接
完成登录
```

旧邮件链接通常不能复用，因为它：

```text
一次性
有过期时间，默认约 15 分钟
```

---

## 19. 项目依赖哪些 npm 包？

除了 Resend/SMTP 外部服务，OIDC 登录主要依赖：

| 包 | 用途 |
|---|---|
| `jose` | JWT / JWK / JWKS / id_token 签发和验证 |
| `lru-cache` | Portal 端短期 state 防重放缓存 |
| `pg` | Core 用 PostgreSQL 持久化登录链接/code/限流等状态 |

### 19.1 `jose`

最核心。

用于：

```text
签发 JWT
验证 JWT
导入 JWK
验证 JWKS
验证 id_token
计算 JWK thumbprint
```

相关文件：

```text
plugins/auth/src/tokens.ts
plugins/auth/src/keys.ts
plugins/portal/src/oidc.ts
```

---

### 19.2 `lru-cache`

用于 Portal 端缓存短期状态，例如已消耗的 state，辅助防重放。

相关文件：

```text
plugins/portal/src/index.ts
```

---

### 19.3 `pg`

Core 使用 PostgreSQL 保存耐久状态。

作用包括：

```text
一次性登录链接已使用记录
authorization code 已使用记录
限流记录
admin_grants
runtime config
```

---

### 19.4 没有使用 Resend npm SDK

项目没有依赖 Resend 官方 npm SDK，而是直接用 Node 原生 `fetch` 调用：

```text
https://api.resend.com/emails
```

相关文件：

```text
plugins/auth/src/email.ts
```

---

### 19.5 没有使用 nodemailer

SMTP 模式下也没有使用 `nodemailer`，而是用 Node 内置：

```text
node:net
node:tls
```

自己实现 SMTP 发送。

相关文件：

```text
plugins/auth/src/smtp.ts
```

---

## 20. OIDC 面试精华总结

### 20.1 一句话定义

> **OIDC 是基于 OAuth2 的身份认证协议，用来让应用通过可信身份提供方确认用户是谁。**

---

### 20.2 OAuth2 vs OIDC

```text
OAuth2：授权，关心能访问什么资源。
OIDC：认证，关心用户是谁。
```

---

### 20.3 最重要流程

```text
用户点登录
→ 应用跳转到 IdP authorize endpoint
→ 用户在 IdP 登录
→ IdP 回调应用 callback，带 code
→ 应用后端用 code 换 token
→ 应用验证 id_token
→ 应用创建自己的 session cookie
```

---

### 20.4 必背关键词

```text
Authorization Code Flow
PKCE
state
nonce
id_token
access_token
JWKS
issuer
audience
redirect_uri
session cookie
```

---

### 20.5 必背口诀

```text
跳 IdP，拿 code；
后端换 token；
验证 id_token；
自己发 cookie。
```

或者：

```text
跳转授权，回调拿码；后端换令，验后发票。
```

其中：

```text
码 = authorization code
令 = token
票 = 应用自己的 session cookie
```

---

### 20.6 面试回答模板

如果面试官问：“说说 Google 登录原理？”

可以这样答：

> Google 登录通常是 OIDC 的 Authorization Code Flow。用户点击登录后，网站生成 state、nonce、PKCE challenge，并把浏览器重定向到 Google 的 authorize endpoint。用户在 Google 完成登录后，Google 回调网站的 redirect_uri，并带上 authorization code。网站后端校验 state 后，用 code、client_secret 和 code_verifier 去 Google token endpoint 换 id_token 和 access_token。网站验证 id_token 的签名、issuer、audience、nonce、过期时间和 email_verified，确认用户身份后，创建自己的网站 session cookie。后续用户访问网站靠网站自己的 cookie，而不是每次都找 Google。

---

### 20.7 面试回答模板：OIDC 和 OAuth2 区别

> OAuth2 主要解决授权问题，即一个应用能否代表用户访问某些资源；OIDC 在 OAuth2 之上增加了身份认证能力，核心是 id_token，用来证明当前用户是谁。所以 access_token 更偏资源访问，id_token 更偏用户身份认证。

---

### 20.8 面试回答模板：state、nonce、PKCE

> state 用来防 CSRF，把登录请求和回调绑定起来；nonce 用来防 id_token 重放，要求 id_token 里带回同一个 nonce；PKCE 用 code_verifier 和 code_challenge 绑定授权码和发起方，即使 authorization code 被截获，没有 code_verifier 也换不到 token。

---

### 20.9 面试回答模板：本项目如何实现

> 本项目里 Portal 是 OIDC Client，plugins/auth 是内置 OIDC Provider，也叫 auth broker。用户访问 Portal 后，Portal 发起 OIDC 登录并代理 `/idp/*` 到 auth broker。auth broker 让用户输入邮箱，检查 `qq.com` 白名单后通过 Resend 发一次性登录链接。用户打开 `/idp/verify#token=...` 并确认后，auth broker 生成 authorization code，Portal 在 `/auth/callback` 收到 code，再向 auth broker 的 `/token` 用 code 换 id_token。Portal 验证 id_token 后创建自己的 session cookie，并把邮箱作为 QM principal，例如 `171232349@qq.com`。

---

## 21. 最终总图

```text
用户浏览器
  |
  | 访问 /auth/login
  v
Portal，OIDC Client
  |
  | 302 跳转 /idp/authorize
  v
auth broker，OIDC Provider
  |
  | 输入邮箱，发邮件
  v
Resend → 用户邮箱
  |
  | 用户点击 /idp/verify#token=...
  v
auth broker 验证邮件 token
  |
  | 302 回调 /auth/callback?code=...
  v
Portal 后端 code 换 token
  |
  | 验证 id_token
  v
Portal 设置 session cookie
  |
  v
Web UI / Admin / Core
```

---

## 22. 最后一行记忆

> **OIDC 登录的本质：应用不碰用户密码，只信任身份提供方签发的 id_token；前端拿 code，后端换 token，验完身份后应用发自己的 cookie。**
