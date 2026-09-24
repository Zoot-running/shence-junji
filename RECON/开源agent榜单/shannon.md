# shannon 分析报告

- **能力范围**：全流程
- **SRC 适用度**：中——必须同时提供源码仓库与可运行目标，属白盒授权渗透/CI-CD 场景；真实 SRC 多为黑盒无源码，且仅覆盖 OWASP 五类、单次 1-1.5h 成本较高。

## 核心思路
白盒自主渗透：先对源码做多阶段 agentic 静态分析定位攻击路径，再驱动浏览器/CLI 对运行中应用实弹利用，只报'有可复现 PoC 证明'的漏洞(No exploit, no report)。

## 架构
Temporal 确定性工作流编排，副作用隔离在 activity；14 个命名 agent(pre-recon→recon→5 漏洞类 vuln→exploit→report)按类并发(限流5)失败隔离；git checkpoint 断点续扫；SAST 走 Capella 十阶段子工作流；产出 PDF/MD/JSON/SARIF 接 CI 门禁。

## 技术面
白盒 agentic SAST(架构→威胁建模→研究→去重→评审→批判→确认→校准→SARIF 十阶段)；黑盒浏览器实弹利用(Playwright+认证流程/TOTP)；proof-by-exploitation(无 PoC 即丢弃)；双流发现对账(侦察候选+SAST 候选合并去重)；受限源码沙箱+多 provider BYOK。

## 短板
强依赖源码，纯黑盒众测不适用；仅覆盖注入/XSS/SSRF/认证/授权五类，缺逻辑/业务漏洞与供应链分析；LLM 结论需人工复核；自建模型效果弱；耗时与 token 成本偏高。

## 值得神策借鉴
Temporal 确定性编排+git checkpoint+每类失败隔离；proof-by-exploitation 与双流候选对账去重(显著降噪)；受限源码沙箱+分阶段 SAST 提升静态分析深度。
