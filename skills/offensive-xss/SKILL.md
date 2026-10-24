---
name: offensive-xss
description: 在已授权测试中检测与验证跨站脚本（反射/存储/DOM/mXSS）时调用。覆盖上下文识别、过滤/编码绕过、CSP 评估与 PoC 构造，强调无害化验证。
license: MIT
---

# XSS — 授权测试方法论

> **授权前提**：PoC 以无害化为准（`alert(document.domain)` 或回调到自有标记端点），禁止窃取真实用户 Cookie/会话、投放蠕虫式存储载荷、或对真实用户发起点击劫持/钓鱼。存储型测试尽量使用测试账号与可清理的标记，测试后清除注入内容。

## 快速流程

1. 枚举所有反射/存储回显点与 DOM sink。
2. 判定**注入上下文**（HTML 文本 / 属性 / JS 字符串 / URL / CSS / 事件处理器）。
3. 按上下文构造最小闭合载荷，验证脚本执行。
4. 评估 CSP / 过滤器是否可绕过。
5. 记录可复现 PoC，给出输出编码/CSP 修复建议。

## 上下文与载荷

| 上下文 | 探针 | 典型载荷（无害） |
|---|---|---|
| HTML 文本 | `<x>` 是否未转义 | `<img src=x onerror=alert(document.domain)>` |
| 标签属性 | `"` `'` 能否闭合 | `" autofocus onfocus=alert(document.domain) x="` |
| JS 字符串 | `'`/`</script>` | `';alert(document.domain)//` |
| URL/href | `javascript:` 协议 | `javascript:alert(document.domain)` |
| DOM sink | `innerHTML`/`eval`/`document.write` | 跟踪 source→sink 数据流 |

## DOM XSS

定位 source（`location.hash`/`search`/`referrer`/`postMessage`）到 sink（`innerHTML`、`outerHTML`、`eval`、`setTimeout(str)`、`document.write`、jQuery `$()`）的链路。用浏览器 DevTools 断点或污点分析确认，不依赖服务端回显。

## 过滤/编码绕过（仅验证可绕过性）

- 大小写与标签变体：`<ScRiPt>`、`<svg/onload=>`、`<math>`。
- 事件处理器：`onpointerenter`、`onanimationstart` 等冷门事件。
- 编码：HTML 实体、`\u`/`\x`、URL 双重编码、`String.fromCharCode`。
- mXSS：利用 innerHTML 规范化差异（`<noscript>`、`<template>`、属性解析）。

## CSP 评估

检查 `Content-Security-Policy`：`unsafe-inline`、`unsafe-eval`、宽松 `script-src`（允许的 CDN 可被 JSONP 滥用）、缺失 `object-src`/`base-uri`。给出“CSP 可绕过/缺失”结论即可。

## 修复建议

上下文感知的输出编码；默认转义的模板引擎；`innerHTML` 改为 `textContent`；严格 CSP（nonce/hash + `strict-dynamic`）；`HttpOnly`/`SameSite` Cookie。

## 边界与禁止

| 禁止 | 替代 |
|---|---|
| 外带真实用户 Cookie/会话令牌 | `alert(document.domain)` 或自有端点回调标记 |
| 投放持久蠕虫载荷 | 可清理的测试标记，测试后删除 |
| 对真实用户诱导点击/钓鱼 | 仅在自有测试会话中验证执行 |
