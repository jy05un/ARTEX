---
name: offensive-api-security
description: 在已授权测试中系统评估 REST/API 安全（对照 OWASP API Top 10：对象/属性/功能级越权、无限制资源消耗、SSRF、鉴权缺陷等）时调用。与 api-recon 衔接，强调最小化、测试账号。
license: MIT
---

# API 安全 — 授权测试方法论（OWASP API Top 10 导向）

> **授权前提**：仅测试授权范围内 API。越权用测试账号，不批量导出真实数据；速率/资源类缺陷以“证明缺少限制”为止，不做压测式冲击。与 `api-recon`（接口面侦察）衔接：recon 负责枚举接口，本 skill 负责按风险项验证。

## 快速流程

1. 从 `api-recon` 或文档/流量获取接口清单（path、method、参数、鉴权）。
2. 建立授权矩阵（角色 × 端点），用测试账号逐项对照验证。
3. 按 OWASP API Top 10 分类检查。
4. 最小化取证，记录可复现请求与修复建议。

## 检查清单（OWASP API Top 10 对应）

| 项 | 检查 | 关联 skill |
|---|---|---|
| API1 对象级越权 (BOLA) | 换账号改对象 id | `offensive-access-control` |
| API2 鉴权缺陷 | 弱令牌、无过期、可爆破、JWT 缺陷 | `offensive-jwt` / `offensive-oauth` |
| API3 属性级越权 (BOPLA) | mass assignment（传 `role`/`is_admin`）、过度返回敏感字段 | — |
| API4 资源消耗 | 缺分页/限流/大小限制（少量验证） | — |
| API5 功能级越权 | 低权账号调管理端点 | `offensive-access-control` |
| API6 敏感业务流滥用 | 自动化滥用关键流程 | `offensive-business-logic` |
| API7 SSRF | URL 参数被服务端取用 | `offensive-ssrf` |
| API8 安全配置错误 | CORS 宽松、调试开启、错误泄露、缺安全头 | — |
| API9 资产管理 | 影子/旧版本端点 `/v1` `/beta` `/internal` | `api-recon` |
| API10 不安全消费第三方 | 回调/webhook/依赖 API 信任链 | — |

## 重点技巧

- **Mass assignment**：在创建/更新请求体里追加未文档化字段（`role`、`verified`、`balance`），看是否被接受。
- **过度数据返回**：对象详情是否返回了前端不展示的敏感字段。
- **CORS**：`Origin: https://evil.example` 看是否回显 `Access-Control-Allow-Origin` + `Allow-Credentials: true`。
- **版本/影子端点**：对照 recon 结果探测旧版本与内部端点的鉴权。

## 修复建议

每端点服务端对象级/功能级授权；输入字段白名单（防 mass assignment）；响应字段白名单；统一限流与分页上限；严格 CORS；下线影子端点；错误信息脱敏。

## 边界与禁止

| 禁止 | 替代 |
|---|---|
| 批量遍历导出真实数据 | 测试账号证明越权 |
| 压测式请求验证限流 | 少量请求证明缺少限制 |
| 对第三方被消费 API 发起实际攻击 | 仅分析信任链风险 |
