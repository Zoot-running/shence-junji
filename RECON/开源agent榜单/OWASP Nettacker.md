# OWASP Nettacker 黑盒 SRC 分析报告

> 评估场景：只有网站、API、域名、IP、测试账号和授权范围，没有目标源码。
>
> **综合评分：40/100。定位：稳定的确定性侦察和已知漏洞扫描器，不是自主猎洞 Agent。**

## 1. 结论

OWASP Nettacker 适合 SRC 的前置层：子域、端口、服务、目录、弱口令、配置和已知 CVE。它没有 LLM 幻觉，事件会落库，报告格式和多目标并发成熟。

但它不能操作浏览器、保持认证态、理解业务逻辑或自主产生未知漏洞假设。YAML 模块命中是网络响应证据，不一定等于可提交的影响证明。

## 2. 主要架构

- Python CLI + Flask API/Web UI。
- `nettacker.py` → `nettacker/main.py` → `core/app.py::Nettacker`。
- `core/module.py` 加载约 135 个 YAML 模块。
- `core/lib/http.py::response_conditions_matched` 按响应条件和正则判断。
- 多进程按目标、多线程按模块并发。
- SQLite/MySQL/PostgreSQL 保存扫描和事件。
- `core/graph.py` 生成多种报告和历史对比。

## 3. 黑盒能力

### 3.1 侦察和已知漏洞

支持：

- 子域枚举；
- 端口和服务发现；
- 目录和文件发现；
- HTTP、FTP、SSH、SMB、SMTP、Telnet 等协议；
- 默认凭据和爆破；
- 已知 CVE、配置和安全 Header 模块；
- socks 代理、随机 User-Agent、延时和重试。

服务发现结果可驱动后续协议模块，适合批量资产基线。

### 3.2 不具备的能力

- 无 JS 浏览器；
- 无 Cookie/多角色认证态；
- 无 MITM 拦截与请求重放；
- 无业务逻辑、IDOR、竞态和流程测试；
- 无自主漏洞假设；
- 无独立 PoC 生成和复测 Agent。

## 4. 证据与报告

Nettacker 的工程优势在数据留存：

- 完整扫描事件和请求/响应数据写数据库；
- HTML、JSON、CSV、文本、SARIF 和 DefectDojo 输出；
- 两次扫描可以生成 compare report；
- API 有 access key、上传限制和报告路径清理。

限制：SARIF 的 URI 是网络目标而不是源码位置；历史对比主要比较 target/module/port，不能表示复杂漏洞状态。

## 5. Scope、限速和安全性

- 支持目标列表、IP/CIDR 和多目标扫描；
- 有 `time_sleep`、retry、proxy 和 `thread_per_host`；
- API 支持 key 和 IP 白名单。

但没有按 SRC 规则定义的 in-scope/out-of-scope 域名继承、重定向策略和危险动作审批。爆破、DoS 类模块仍需人工禁用。

## 6. README 与实现差异

README 对模块化、并发、报告和 drift detection 的描述基本属实。需要降级理解的部分：

- “漏洞扫描”主要是已知模板和响应签名；
- drift detection 是扫描事件差异，不是智能漏洞生命周期；
- 部分协议支持是通过通用 HTTP 或模块模拟，不代表完整协议引擎。

## 7. 评分

| 维度 | 分数 |
|---|---:|
| 资产/子域/API 发现 | 45 |
| 浏览器/代理/认证态 | 5 |
| 业务逻辑 | 5 |
| 自动利用与 PoC | 30 |
| 误报控制 | 45 |
| 证据与报告 | 75 |
| Scope/限速/审批 | 40 |
| 多目标与扩展 | 70 |
| 部署成本 | 60 |

**最终判断：40/100。** 适合批量资产和已知漏洞前置扫描。需要交给人工或 Agent 复测后，才能形成可提交 SRC finding。

## 8. 关键源码依据

- `nettacker.py`
- `nettacker/main.py`
- `nettacker/core/app.py`
- `nettacker/core/module.py`
- `nettacker/core/lib/http.py`
- `nettacker/core/graph.py`
- `nettacker/config.py`
- `nettacker/modules/vuln/*.yaml`
- `nettacker/api/engine.py`
- `nettacker/database/`
