---
name: offensive-sqli
description: 在已授权的 Web/API 渗透测试中进行 SQL 注入检测、确认与取证时调用。覆盖报错型、UNION、布尔/时间盲注、二阶注入、NoSQL 与 ORM 场景，以及 WAF 绕过思路与 sqlmap 用法。
license: MIT
---

# SQL 注入 — 授权测试方法论

> **授权前提**：仅对书面授权范围内的目标测试。注入确认以“证明存在”为止，禁止批量拖库、删改数据、堆叠查询执行写操作或 DROP/TRUNCATE 等破坏性语句。取证只取最小必要样本（如版本号、当前库名、单行脱敏数据）。超范围或破坏性动作须走人在环路审批。

## 快速流程

1. 枚举所有可达数据库的输入点：URL 参数、POST body、Cookie、HTTP 头、JSON 字段、GraphQL 变量、WebSocket 消息。
2. 逐点插入探针，优先观察报错 → UNION，无回显则退到布尔/时间盲注。
3. 识别数据库类型与版本（指纹）。
4. **仅做最小确认**：取 `version()`/`current_user`/`database()` 等证据，不做全量导出。
5. 记录可复现的 PoC 请求与证据，给出修复建议。

## 检测探针

```
'            "            `            )            ')
1'||'1       1 AND 1=1      1 AND 1=2
' OR '1'='1' --            " OR "1"="1" --
```

- **报错型**：构造语法错误观察是否回显数据库错误（`SQL syntax`、`ORA-`、`PG::`、`SQLite error`）。
- **布尔盲注**：对比 `AND 1=1` 与 `AND 1=2` 的响应差异（长度、状态码、关键字）。
- **时间盲注**：`AND SLEEP(5)`（MySQL）、`pg_sleep(5)`（PostgreSQL）、`WAITFOR DELAY '0:0:5'`（MSSQL）、`dbms_pipe.receive_message(('a'),5)`（Oracle）。以稳定的时延差作为判据，注意网络抖动误报。

## 指纹与枚举（最小化）

| DB | 版本 | 当前库/用户 |
|---|---|---|
| MySQL | `@@version` | `database()` / `current_user()` |
| PostgreSQL | `version()` | `current_database()` / `current_user` |
| MSSQL | `@@version` | `DB_NAME()` / `SYSTEM_USER` |
| Oracle | `banner FROM v$version` | `SYS.DATABASE_NAME` / `USER` |

UNION 注入：先用 `ORDER BY N` 或 `UNION SELECT NULL,...` 定列数，再定可回显列类型。枚举 schema 时查 `information_schema`（MySQL/PG/MSSQL）或 `all_tables`（Oracle），**只取结构，不整表导出**。

## 盲注取证

无回显时用二分法逐字符推断，但仅针对单个证据字段（如一个用户名或 flag），禁止脚本化全表外带。可借助 sqlmap 的受控参数：

```
sqlmap -r req.txt --batch --banner --current-db --current-user
# 禁止在授权范围外使用 --dump-all / --os-shell / --sql-shell
```

## 特殊场景

- **二阶注入**：输入在写入时无害，在后续读取/拼接处触发——跟踪数据流而非单请求。
- **NoSQL（MongoDB）**：`{"$ne": null}`、`{"$gt": ""}`、`[$regex]` 参数污染；JSON body 中注入操作符。
- **ORM/CVE**：关注已知 ORM 原始查询拼接点与历史 CVE（如不当的 `raw()`/`extra()` 用法）。

## WAF 绕过（仅用于确认可绕过性）

大小写混淆、内联注释 `/*!50000...*/`、等价函数替换、分块编码、参数污染。记录“WAF 可被绕过”这一事实即可，不做进一步利用。

## 修复建议

参数化查询/预编译语句；ORM 安全 API；最小权限数据库账号；输入白名单与输出编码；统一错误处理避免泄露。

## 边界与禁止

| 禁止 | 替代 |
|---|---|
| 全表 `--dump`/`--dump-all` | 取单行或计数证明存在即可 |
| 堆叠查询写/删数据、`--os-shell` | 只读确认，写操作走审批 |
| 对未授权子域/旁站注入 | 严格限定授权清单 |
| 以时延盲注暴力外带整库 | 仅推断单个证据字段 |
