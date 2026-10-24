---
name: offensive-oauth
description: 在已授权测试中评估 OAuth2/OIDC 实现缺陷（redirect_uri 校验、state/CSRF、PKCE 缺失、隐式流、令牌泄露、scope 提升）时调用。强调用测试账号与自有回调域验证。
license: MIT
---

# OAuth2 / OIDC — 授权测试方法论

> **授权前提**：用测试账号与**自有**回调域验证流程缺陷。禁止利用缺陷窃取真实用户的授权码/令牌或接管真实账号；账号接管类验证仅在两个测试账号间进行。

## 快速流程

1. 梳理所用流程（authorization code / PKCE / implicit / client credentials）与各端点。
2. 审查 `redirect_uri` 校验、`state`、PKCE、令牌传递方式。
3. 逐项验证缺陷（见下），用测试账号 + 自有回调确认。
4. 记录 PoC 与修复建议。

## 常见缺陷

| 缺陷 | 测试 |
|---|---|
| `redirect_uri` 宽松 | 子串/前缀匹配、`//evil`、`@`、路径追加、开放重定向跳转（结合 open-redirect） |
| 缺 `state` / 不校验 | CSRF 绑定攻击（登录 CSRF），用测试账号验证 |
| 缺 PKCE（公共客户端） | 授权码可被截获重放 |
| 隐式流泄露 | 令牌出现在 URL fragment/Referer/日志 |
| `scope` 提升 | 请求/篡改为更高 scope 看是否签发 |
| 授权码复用 | 同一 code 二次兑换 |
| `id_token` 校验弱 | 结合 `offensive-jwt`（签名/aud/iss/nonce） |

## 验证要点

- 所有“跳到攻击者可控地址”的验证，回调域使用**自有**域名，证明 `redirect_uri` 校验不严即可，不针对真实用户钓鱼。
- 账号接管链路（如绑定劫持）仅在测试账号 A/B 间演示。

## 修复建议

`redirect_uri` 精确匹配白名单；强制 `state` 与 PKCE；避免隐式流、令牌不走 URL；授权码一次性短时效；服务端校验 `id_token` 的签名/aud/iss/nonce；scope 最小化与服务端强制。

## 边界与禁止

| 禁止 | 替代 |
|---|---|
| 窃取真实用户授权码/令牌 | 自有回调域证明校验缺陷 |
| 接管真实账号 | 两个测试账号间演示 |
| 对真实用户发起登录 CSRF | 在测试会话中验证机制缺失 |
