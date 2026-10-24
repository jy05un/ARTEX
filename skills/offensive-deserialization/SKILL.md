---
name: offensive-deserialization
description: 在已授权测试中检测不安全反序列化（Java/.NET/PHP/Python/Node）时调用。覆盖序列化数据识别、魔术方法/gadget 概念、无害化确认，强调不投放破坏性 gadget 链。
license: MIT
---

# 不安全反序列化 — 授权测试方法论

> **授权前提**：确认漏洞以“证明受控数据被反序列化并可影响执行流”为止。禁止直接投放 RCE gadget 链执行系统命令、反弹 shell 或读写文件；若需证明可达 RCE，用无害带外探针（如触发一次到自有端点的 DNS/HTTP 回连）或只读命令截断输出，破坏性利用走审批。

## 快速流程

1. 识别序列化数据的位置与格式。
2. 判定可控性与反序列化框架。
3. 用无害探针确认数据被反序列化、并评估 gadget 可达性。
4. 记录 PoC 与依赖证据，给出安全反序列化的修复建议。

## 识别序列化格式

| 语言/框架 | 特征 |
|---|---|
| Java | `rO0` (base64 的 `0xACED0005`)、`application/x-java-serialized-object` |
| PHP | `O:8:"..."`、`a:2:{...}` |
| .NET | `AAEAAAD/////`、ViewState、`BinaryFormatter` |
| Python | pickle 字节、`__reduce__` |
| Node | `_$$ND_FUNC$$_`、`node-serialize` |
| 通用 | Cookie/隐藏字段/缓存里的 base64 结构化数据 |

## 确认思路（概念层）

- 触发点通常是框架在读取受控数据时的魔术方法（Java `readObject`、PHP `__wakeup`/`__destruct`、Python `__reduce__`）。
- gadget 链依赖目标 classpath/依赖；先用工具（如 ysoserial 的**带外/探测型**载荷）确认“是否被反序列化”，而非直接上命令执行链。
- 无回显时以 DNS/HTTP 带外回连自有端点作为存在性判据。

## 最小化影响证明

确认后，分级证明：可控属性注入 → 触发无害副作用（带外回连）→ 如需证明 RCE，仅执行只读命令并截断输出。全程不破坏数据、不持久化。

## 修复建议

避免反序列化不可信数据；改用 JSON 等数据格式 + 严格 schema；若必须，使用带类型白名单的安全反序列化、签名/HMAC 校验完整性、隔离运行；及时升级存在反序列化 CVE 的依赖。

## 边界与禁止

| 禁止 | 替代 |
|---|---|
| 投放破坏性 RCE gadget、反弹 shell | 带外探针确认，必要时只读命令截断输出 |
| 篡改/删除反序列化涉及的业务数据 | 只读与无害副作用 |
| 对无关主机回连 | 仅自有带外端点 |
