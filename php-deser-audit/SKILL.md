---
name: php-deser-audit
description: PHP Web 源码反序列化/对象注入审计工具。识别 unserialize 注入点与可控数据来源，追踪魔术方法链，输出可利用性分级、PoC 与修复建议（禁止省略）。
---

# PHP 反序列化审计（php-deser-audit）

分析 PHP 项目源码，识别用户可控数据进入 `unserialize()` 或等价反序列化点，结合类的魔术方法（`__wakeup/__destruct/__call/__toString/{value}`）判断是否存在对象注入可利用链。

## 分级与编号
- 详见：`shared/SEVERITY_RATING.md`
- 漏洞编号：`{C/H/M/L}-DESER-{序号}`

## 危险 Sink（必做）
- `unserialize($data)`
- `unserialize(base64_decode({value}))`
- 任何将用户输入转换后再反序列化的路径

## 可控性（必做）
必须追踪并输出：
- 反序列化入参来自哪里：GET/POST/Cookie/Session/Header/数据库字段等
- 是否存在 base64/加密/解码/拼接步骤（仍可能可控）
- 是否有校验/签名（如 `hash_hmac`）以及签名是否可靠（需要证据）

## gadget 链识别（必做）
必须输出：
- 反序列化后的对象类型（或可能类型集合）
- 该类型类中魔术方法列表与触发条件
- “从对象创建 -> 魔术方法触发 -> 最终敏感操作（如文件/命令/SSRF/数据库）”的数据流链

## tracer 触发条件（必做）
- 反序列化入参经过多层 decode/封装
- 魔术方法链跨多个类/文件

## 证据引用（强制：来自 php-route-tracer）
每条反序列化/对象注入疑似漏洞必须逐项引用 trace 输出中 `## 9) Sink Evidence Type Checklist` 的 **DESER 行**对应证据要点（允许状态为待验证，但证据引用必须存在）：
1. `EVID_DESER_CALLSITE`：反序列化调用点位置证据（unserialize 等等价点）
2. `EVID_DESER_INPUT_SOURCE`：入参用户可控来源证据
3. `EVID_DESER_OBJECT_TYPE_MAGIC_TRIGGER_CHAIN`：反序列化后对象类型/魔术方法触发条件证据（以及最终敏感操作点链路证据，若能定位）

## tracer 证据缺失处理（强制）
- 若无法定位上述 1~3 任一关键证据要点：该漏洞状态只能标记为 `⚠️待验证`，不得直接给出 `✅已确认可利用`。

## 报告输出
输出到：
```
{output_path}/vuln_audit/deser_{timestamp}.md
```

## PoC（强制，但标注为“概念 PoC/验证 PoC”）
必须给出：
- 真实路由（或 cookie/session 字段使用方式）
- payload 生成思路（若无法生成真实 gadget payload，必须给出“需人工补齐的部分”并说明原因，同时仍保留可执行框架与可替换字段）

