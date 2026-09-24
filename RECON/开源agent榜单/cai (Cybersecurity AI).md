# cai (Cybersecurity AI) 分析报告

- **能力范围**：全流程
- **SRC 适用度**：中——面向 bug bounty 设计：侦察、发现↔复测↔报告链路成熟且有多例真实 CVE 战果；但利用/提权/横移工具目录多为空、深度利用依赖通用 shell 与强模型，且已归档停维。

## 核心思路
以 ReACT 式 LLM 智能体+按杀伤链组织的安全工具与多智能体 handoff/模式，让 AI 半自主地执行侦察→利用→提权→横移→验证→报告的攻防闭环。

## 架构
基于 openai-agents-python，Chat Completions 无状态调用。八支柱：Agent/Tools/Handoffs/Patterns/Turns/Tracing/Guardrails/HITL；工具按杀伤链六类分组；编排 agent 广度优先委派、双路竞赛、并行专员子进程(带轮次预算)；Ctrl+C 人工接管。

## 技术面
黑盒 ReACT 智能体；多智能体模式(swarm/层次/并行/双路竞赛)与 agent-as-tool；杀伤链工具链(nmap/shodan/c99/子域枚举+通用 shell/exec_code/sshpass)；OSINT 网页情报带 SSRF 防护与提示注入四层护栏；MCP 与 Computer-Use 扩展。

## 短板
利用/提权/横移/渗出工具目录基本为空；无状态调用缺持久记忆；提示注入面大；Tracing 未完成；偏半自主；已归档无维护。

## 值得神策借鉴
按杀伤链组织工具与专员智能体(红队/漏洞猎人/复测/报告/DFIR/APT)+handoff+模式编排；发现↔复测 swarm 去误报循环+无工具 Reporter 出报告；并行专员+双路竞赛编排(带轮次预算)与护栏/人工接管层。
