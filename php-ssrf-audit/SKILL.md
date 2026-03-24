---
name: php-ssrf-audit
description: PHP Web 源码 SSRF 审计工具。识别用户可控 URL/地址进入网络请求 Sink，追踪内网/协议/端口限制与回显，输出可利用性分级、PoC 与修复建议（禁止省略）。
---

# PHP SSRF 审计（php-ssrf-audit）

分析 PHP 项目源码，识别 SSRF 风险：用户可控 URL/主机/路径进入网络请求或读取类函数，且缺少协议 allowlist、DNS/IP 内网拒绝、端口限制等控制。

## 分级与编号
- 详见：`shared/SEVERITY_RATING.md`
- 漏洞编号：`{C/H/M/L}-SSRF-{序号}`

## SSRF Sink（必做）
识别至少以下网络/地址访问点：
- `curl_setopt(CURLOPT_URL, $url)` / `curl_exec`
- `file_get_contents($url)` / `readfile($url)` / `fopen($url, {value})`
- `stream_get_contents($stream)` / `fsockopen` / `pfsockopen`
- `get_headers($url)` / `dns_get_record`（配合地址解析时）
危险模式：
- URL/Host/Port/Path 来自用户输入且可控
- 协议未限制（允许 `file://`、`gopher://`、`php://` 等）

## 必做的安全检查点
- 协议白名单：仅允许 `http/https`（如有必要）
- DNS/IP 解析后内网网段拦截：拒绝 `127.0.0.1/::1/10.0.0.0/8/172.16.0.0/12/192.168.0.0/16` 等
- 端口限制：拒绝内网高危端口（由项目策略决定）
- 重定向处理：跟随跳转的限制（如 `CURLOPT_FOLLOWLOCATION`）
- 重定向与最终地址校验：即使初始 URL 通过 allowlist，仍必须对“重定向后的最终 URL”执行同等拦截与协议校验
- DNS rebinding 防护：允许域名访问时，必须在实际发起请求前进行最终解析，或使用安全解析策略避免“先外网后内网”
- URL 归一化：对编码后的主机/路径（双重编码、大小写、用户信息段 `user:pass@host`）先归一化再做 allowlist 判断

## 触发 tracer 的条件
- URL 由多字段拼接（schema + host + path + query）
- 存在多层函数调用或编码/解码/替换
- 安全判断依赖字符串规范化（必须追踪真实最终 URL）

## 报告输出
```
{output_path}/vuln_audit/ssrf_{timestamp}.md
```

## 漏洞条目模板（强制）
与 `php-sql-audit` 同结构，但在数据流链里必须覆盖：
- 最终用于发起请求的 URL/Host/Port 变量来源与拼接过程
- 协议选择与内网拦截判断的分支证据

## 证据引用（强制：来自 php-route-tracer）
每条 SSRF 疑似漏洞必须逐项引用 trace 输出中 `## 9) Sink Evidence Type Checklist` 的 **SSRF 行**对应证据要点（禁止只写“可能”；允许状态为待验证，但证据引用必须存在）：
1. `EVID_SSRF_URL_NORMALIZATION`：URL 归一化步骤（对应 SSRF 行中的证据点）
2. `EVID_SSRF_FINAL_URL_HOST_PORT`：发起请求前的最终 URL/Host/Port（对应 SSRF 行中的证据点）
3. `EVID_SSRF_FINAL_REDIRECT_URL`：若存在重定向/跟随跳转：最终重定向后地址证据（对应 SSRF 行中的证据点）
4. `EVID_SSRF_DNSIP_AND_INNER_BLOCK`：DNS/IP 解析与内网拦截判定证据（对应 SSRF 行中的证据点）

## tracer 证据缺失处理（强制）
- 若在 trace 中找不到上述 1~4 任一关键证据点：该条漏洞状态只能标记为 `⚠️待验证`，不得给出 `✅已确认可利用`。

#### PoC（强制）
- 必须给出可执行请求（HTTP 代码块）
- PoC URL 必须使用真实路由并替换为真实可控字段名

