# ✨ PHP Code Audit Skill

本仓库是一套面向 **PHP Web** 的白盒代码安全审计 Skill 集合，覆盖「路由枚举 → 鉴权建模 → 数据流追踪 → 分类漏洞审计 → 证据一致性校验 → 报告汇总」全流程；在 Cursor 等环境中以 **Agent 按 SKILL 文档执行** 的方式落地（非随仓库附带的可执行脚本，详见 `php-audit-pipeline` 说明）。

**项目特点**
- **证据契约驱动**：以 `php-route-tracer` 输出的证据点（`EVID_*`）作为多数子审计 skill 的硬门槛。
- **全量路由与参数**：`php-route-mapper` 须输出完整路由与参数结构（禁止省略）。
- **可控性与分支证明**：trace 含参数可控性矩阵、分支执行证据与 sink 定位汇总。
- **结论强度约束**：高危结论依赖证据链闭合，避免仅凭关键字猜测即标「已确认」。

> **适用场景**：在合法授权前提下，用于 PHP 项目的白盒安全评估、上线前审计、修复验证与回归辅助。  
> [![PHP](https://img.shields.io/badge/PHP-Whitebox%20Audit-blue)](#)
> [![Evidence Contract](https://img.shields.io/badge/EVID_%20Contract-Enabled-success)](#)
> [![Pipeline](https://img.shields.io/badge/Workflow-Route%20%2B%20Trace%20%2B%20Audit-orange)](#)
> ![License](https://img.shields.io/badge/License-Add%20LICENSE-lightgrey)

## 📌 快速索引
- 🧩 能力总览
- 🔒 证据契约（含 `shared/IO_PATH_CONVENTION.md` 路径约定）
- 🚀 主流水线
- 🧭 如何使用
- 🚀 配置与使用示例
- 🧰 新增/扩展 skill 的规范
- 🗺️ 路线图
- 📁 目录结构

## 🛡️ 免责声明

本项目仅用于**合法合规的安全测试**与代码安全审计。请勿用于未授权目标。

---

## 🧩 能力总览

### 路由建模与追踪
- `php-route-mapper`：提取**所有**路由与完整请求参数结构，并生成可用于测试的请求模板。
- `php-route-tracer`：基于路由与参数输入，输出从 handler 到最终 sink 的 **数据流链、分支执行证据、可控性矩阵**（不做漏洞结论）。

### 漏洞审计
每类漏洞审计遵循证据契约：须逐项引用 `php-route-tracer` 的 `## 9) Sink Evidence Type Checklist` 中对应行的证据点 ID（无 trace 时的静态对齐方式见 `shared/EVIDENCE_POINT_IDS.md` 文首说明）。
- SQL 注入：`php-sql-audit`
- NoSQL 注入：`php-nosql-audit`
- 命令注入：`php-cmd-audit`
- SSRF：`php-ssrf-audit`
- XSS：`php-xss-audit`
- 任意文件读取/路径穿越（含 include/require 执行面边界）：`php-file-read-audit`
- 文件上传：`php-file-upload-audit`
- 任意文件写入（路径穿越到落点/写入链）：`php-file-write-audit`
- 归档解压路径穿越（Zip Slip）：`php-archive-extract-audit`
- XXE：`php-xxe-audit`
- 反序列化/对象注入：`php-deser-audit`
- 模板注入/SSTI：`php-tpl-audit`
- LDAP 注入：`php-ldap-audit`
- 表达式注入（非模板）：`php-expr-audit`
- 认证/鉴权绕过/越权/IDOR：`php-auth-audit`
- CSRF：`php-csrf-audit`
- 开放重定向：`php-open-redirect-audit`
- CRLF/响应分割：`php-crlf-audit`
- 会话与 Cookie 安全：`php-session-cookie-audit`
- 安全配置（危险开关/CORS/错误暴露/安全头）：`php-config-audit`
- 加密与密钥安全：`php-crypto-audit`
- 业务逻辑漏洞：`php-logic-audit`
- 安全日志与监控：`php-logging-audit`

### 非 trace-gate 的增强能力
- `php-filesystem-audit`：审计文件系统操作（权限/链接/删除/TOCTOU 等），用于提升链式利用可行性（不替代 FILE/UPLOAD/WRITE 等 sink 审计）。

### 链路聚合
- `php-exploit-chain-audit`：把已产出的漏洞报告按前置条件串成跨漏洞利用链叙事，并输出「未能串联原因清单」。

### 供应链与框架检测
- `php-vuln-scanner`：基于 `composer.json` / `composer.lock`（及可选规则集）做依赖版本与已知漏洞匹配；可与环境中的 `composer audit` 交叉比对（见该 SKILL 内说明）。
- `php-*-audit`（框架专项）：Laravel / Symfony / Yii / ThinkPHP / WordPress / CodeIgniter 等典型配置与用法风险映射。

---


## 🔒 证据契约（最重要）

本项目核心是「字段级同名对齐」：

1. `php-route-tracer` 必须输出：
   - `## 5) 参数可控性矩阵`
   - `## 6) Sink 定位`
   - `## 7) Sink Summary`
   - `## 8) Trace 完整性声明`
   - `## 9) Sink Evidence Type Checklist`
2. `shared/EVIDENCE_POINT_IDS.md` 定义证据点 ID 字典（`EVID_*`）；**尚未有 trace 时**如何仍以静态代码对齐 ID，见该文件文首「尚未执行 php-route-tracer」小节。
3. 各子审计 skill 在「证据引用」部分须逐项引用对应证据点 ID；缺失则只能标记为 `⚠️待验证`，不得直接声称 `✅已确认可利用`。
4. `shared/IO_PATH_CONVENTION.md` 统一 **路由/参数等输入路径**：`routes_*.md` 与 `route_mapping/routes_*.md`、流水线合并报告中的章节**逻辑等价**。

可理解为：**trace 输出是「合同」，子审计按 `EVID_*`「盖章对齐」**。



---

## 🚀 主流水线：`php-audit-pipeline`

`php-audit-pipeline` 为编排总入口，阶段顺序为：  
路由/参数 → 鉴权映射 → 组件漏洞扫描 →（可选框架专项）→ 分批追踪（`php-route-tracer`）→ 分类审计 → 汇总质量报告 → 利用链聚合。

**补充约定**（与仓库内 SKILL 一致）：
- **路径别名**：`shared/IO_PATH_CONVENTION.md`（合并流水线 vs 独立落盘 `route_mapping/` 等）。
- **Sink 细则**：`shared/PHP_SINK_REFERENCE.md`（pipeline 正文引用该文件，避免重复冗长）。
- **覆盖矩阵**：最终汇总须含子 skill / 审计面矩阵（每行：已执行 / 不适用 / 已延期 + 依据）；非 HTTP 入口使用合成 `route_id`，见 pipeline 阶段 3。
- **漏报抑制**：trace 为 `PARTIAL/UNRESOLVED` 或静态兜底未闭合时，须在质量报告中保留 **「Trace 未闭合 / 待补证风险池」**，不得静默删除（详见 `php-audit-pipeline`）。

### 输入参数
- `source_path`：PHP 项目源码根目录（目录或仓库根）。
- `output_path`（可选）：输出目录（默认 `{source_path}_audit`）。

### 输出目录
流水线在**合并模式**下通常只产生**一份**汇总 MD（不强制拆成多文件落盘）：
- `{output_path}/{目录名}_代码审计_{timestamp}.md`

其中：
- `{目录名}` 为 `source_path` 的目录名（basename）
- `timestamp` 建议格式为 `YYYY_MM_DD_HH_mm`（示例：`2026_03_20_18_00`）

---

## 🧭 如何使用

1. **在 Cursor 中选择 Agent Skill**  
   以 `php-audit-pipeline` 为入口，提供 `source_path` 与可选 `output_path`。

   如在Ai对话中直接使用  ``` 使用我的skill php-audit-pipeline 为入口 进行代码审计 报告输出在根目录 ```

2. **trace 未闭合时的预期表现**  
   若部分路由 trace 契约校验失败，会标记为 `PARTIAL/UNRESOLVED`：结论**不得静默消失**，须进入 `⚠️待验证` / `🔍环境依赖`，并汇总到质量报告 **「Trace 未闭合 / 待补证风险池」**（可与阶段 2.5 静态兜底交叉引用）。

3. **阅读顺序**  
   优先查看汇总文件中的 **「质量报告章节」**，再下钻 trace / 漏洞详情。

---

## 🚀 配置与使用示例

### 1) 需要你提供的配置

- `source_path`：要审计的 PHP 项目根目录（必填）
- `output_path`（可选）：输出目录（默认 `{source_path}_audit`）

### 2) 一键运行示例（推荐：`php-audit-pipeline`）

在 Cursor 里给 Agent 的示例提示词（路径请替换为实际值）：

```text
请使用 `php-audit-pipeline` 审计：
source_path = "C:/path/to/your-php-project"
output_path = "C:/path/to/your-php-project_audit"
```

建议重点查看：
- 汇总文件中的「质量报告章节」：风险统计、证据完整度、覆盖矩阵、待补证风险池
- 「漏洞详情段落」：编号、数据流、PoC、修复建议
- 「利用链聚合章节」：跨漏洞链路引用

### 3) 可选：手动分阶段运行

1. 路由枚举与参数建模：`php-route-mapper` → 写入汇总对应章节  
2. 鉴权映射：`php-auth-audit`（`STATIC_MAPPING`）  
3. 深度追踪：`php-route-tracer`  
4. 分类漏洞审计（trace-gate 等）：如 `php-sql-audit`、`php-cmd-audit`、`php-file-read-audit` 等  
5. 利用链聚合：`php-exploit-chain-audit`

### 4) 常见现象：为什么大量 `⚠️待验证`

多数漏洞类型为 trace-gate：当 trace 缺失关键证据点或 `trace_status=PARTIAL/UNRESOLVED` 时，pipeline **禁止**标 `✅已确认`，并须保留 `⚠️待验证`；同时在 **「Trace 未闭合 / 待补证风险池」** 列出缺失证据与补证建议（**不得**用「未生成漏洞条目」代替记录）。

排查建议：质量报告未完成项 → 对应 trace → 子 skill 的 `EVID_*` 是否齐全（见 `EVIDENCE_POINT_IDS.md`）。

---

## 🧰 新增/扩展 skill 的规范

每个 skill 的 `SKILL.md` 建议包含 YAML 头部：
- `name`
- `description`

并尽量遵循本仓库约束：
- **完整输出**：禁止省略占位符、禁止只给结论。
- **证据可追溯**：trace-gate 类别须引用 `shared/EVIDENCE_POINT_IDS.md` 中的证据点 ID。
- **字段名对齐**：与 `php-route-tracer` 表格列名一致，勿随意改名。

---

## 🗺️ 路线图

可按证据契约复杂度推进：
1. 新增 trace-gate 类型：同步扩展 `EVIDENCE_POINT_IDS.md`、`php-route-tracer` 的 `## 9)` 与 pipeline 接入说明。
2. 新增非 trace-gate 的聚合/质量类 skill：不改动契约前提下提升产出一致性。

---

## 📁 目录结构

根目录包含多个 `php-*/SKILL.md` 子 skill；基础设施与编排为：
- `php-audit-pipeline`：统一编排与汇总约定
- `php-route-mapper` / `php-route-tracer`：路由与追踪
- `shared/`：证据 ID、路径约定、严重度、Sink 参考等

---

## 🪪 许可（License）

仓库根目录若尚未包含 `LICENSE` 文件，请按需自行补充（如 MIT、Apache-2.0 等）并更新本节说明。当前 Badge 显示为 *Add LICENSE* 时，表示许可尚未最终选定。
