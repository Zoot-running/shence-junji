# 托管模式探索档案（2026-09-12，API 实测 + SPA 反编译 + 协议条款）

> 目标：run 12 转托管模式前，摸清"如何托管 / 需要什么 / 赛后日志能否回看 / 模型与价格"。
> 证据来源：①平台 API 实测（session Bearer）；②前端 SPA 拆包（tsec-app.js + 页面 chunk）；
> ③用户协议全文（打包在 BenchmarkPrepareView chunk 内）；④榜一 leixiao（run 13067）
> 的公开会话实拉。均为 2026-09-12 当日取证。

## 一、托管模式是什么、怎么进（三步流程 + 六条规则）

控制台"托管模式"（URL `?mode=hosted`）三步：

1. **接入大模型网关**：把程序中所有大模型 API 地址换成平台网关
   （`https://api.deepseek.com/v1` → `http://api.deepseek.com.tsecbench.gw/v1`；
   页面上 DeepSeek/Kimi 两条预设映射，任意 http(s) 主机均可改写为 `<host>.tsecbench.gw`）。
2. **制作并上传 Docker Image**：把程序打成 docker image 提交
   （`.tar/.tar.gz/.tgz`，≤3GB，SHA-256 去重；上传走 COS 临时凭据，bucket
   `tsecbench-static-1304459781`）。
3. **配置动态环境变量**：密钥等敏感配置从包内剔除，以 KEY=VALUE 环境变量录入
   （`BENCHMARK_TOKEN`/`BENCHMARK_BASE_URL` 为平台保留名，自动注入）。

六条规则（页面原文）：

| # | 规则 |
|---|---|
| 1 | 平台提供 **8核16G** 运行资源 |
| 2 | **务必先完成本地模式评测**（Agent 已接入平台 API），再进行托管模式评测 |
| 3 | 程序包内剔除动态/敏感配置（密钥凭证、大模型 API 等），以环境变量读取 |
| 4 | **托管运行全程数据将被审计和部分公开**（含网络流量、大模型对话记录） |
| 5 | 程序在独立、一次性沙箱运行，结束即销毁，平台不留存程序 |
| 6 | **Agent 不可内置针对题目的历史答题记忆或解题方法——发现即作弊处置** |

另有"**提示词接入**"通道（IntegrationGuide chunk）：平台自建 harness 循环跑一个
系统提示词（默认提示词全文已抽取，见 junji 本档案附录），底座模型可选
`deepseek-v4-flash-202605` 等平台模型——即本地模式的 `single` 档 + `baseModel` 参数。

## 二、需要什么信息（API 面实测）

| 端点 | 方法 | 用途 | 实测 |
|---|---|---|---|
| `/my/agents` | GET/POST | 建/查 agent（agent_tag+description） | ✓ 现有 agent 2055 dsh-local-agent |
| `/runs/hosted/upload-cred` | POST {filename} | 拿 COS 临时上传凭据（30 分钟有效） | ✓ 已实测拿全凭据 |
| `/runs/hosted/upload-history` | GET ?agent_id= / POST | 上传记录（file_name/app_dist/file_hash） | ✓ |
| `/runs` | POST | 建 run：set_id, agent_id, run_mode(full/single), point_dimension, base_model, **run_source(hosted)**, **app_dist**, **env_config**, remark | 本地模式已用 |
| `/runs/active` | GET | 当前活跃 run（互斥单跑） | ✓ |
| `/runs/{id}/status?score_start=&logs_start=` | GET | 轮询增量得分事件+日志 | ✓ |
| `/runs/{id}/result` | GET | 终态结果 | ✓ |
| `/benchmark-sets/{id}` | GET(公开) | 题集元数据（exam_mode 等） | ✓ |

**托管创建即**：上传镜像 → `POST /runs {run_source:'hosted', app_dist:<上传URL>, env_config:{...}}`。
注意 `/runs/active` 互斥：同一时刻只有一个活跃 run（本地与托管共享）。

## 三、核心问题：赛后日志能不能回看？——能，且留 24 个月

**证据链（全部当日实测）**：

1. **会话级完整日志 API**：`GET /runs/{id}/llm/sessions`（会话列表：model/protocol/
   status/usage/标题）→ `GET /runs/{id}/llm/sessions/{sid}?from=&to=` 返回**完整对话
   原文**（分页，含 usage 四元组 input/output/reasoning/cache_read）。
   榜一 leixiao run 13067 的会话**无需登录即可拉取**（`/leaderboard/agent/{run_id}/llm/sessions`），
   已实拉 session 574794：64 条事件、194KB 原文——托管会话对公众部分公开是实锤。
2. **模型用量**：`GET /runs/{id}/llm/model-usage`（按模型聚合 calls/sessions/usage 四元组）；
   榜一公开可见：v4-flash-202605 7726 calls/729 sessions/255M cache_read。
3. **实时日志**：WebSocket `GET /api/v1/runs/{id}/logs/ws?access_token=<run token>`（控制台
   "过程观察"面板同源）+ status 轮询的 `logs_start` 增量参数。
4. **控制台下载**："过程观察"每个会话带"下载 JSON"按钮（即 sessions API 的数据）。
5. **留存条款**（用户协议 8.3）：评测数据（**含原始日志**）归平台所有、**默认保留 24 个月**，
   个人面板可查看/下载自己名下的结果副本。
6. **注意**：本地模式的平台侧会话是空的（agent 在本地跑，平台不存）——托管后平台侧
   才会有全量日志。我们的 L4 复盘工作流可以直接挂到 sessions API 上。

## 四、模型与价格（用户问题：腾讯部署的 DeepSeek 价格差异）

- 托管沙箱的大模型走平台网关（`.tsecbench.gw` 内网域，HTTP），即**腾讯侧部署**的模型。
  榜上可见的平台模型名：`deepseek-v4-flash-202605`、`deepseek-v4-pro`（与官方 API 型号
  命名不同，是平台/腾讯部署版本）。
- 沙箱出网白名单含官方 `api.deepseek.com`、`api.moonshot.cn`、`open.bigmodel.cn` 等——
  技术上带自有 key 直连官方端点也可行，但托管推荐路径是平台网关。
- **价格差异：公开面查无定价**。SPA 全量 chunk 无计费/充值/余额 UI，API 无
  wallet/billing 端点（逐一 404 实测）。排行榜只报 token 用量不报价。→ 网关价差
  **待确认**，渠道只有：①实际创建一个托管 run 看控制台是否出现计费条目；②官方
  渠道（协议留的客服微信号 Wx62887799 / 用户群）问价。
- 兜底认知：官方 DeepSeek 当前价 入1-2/出4-8/缓0.02-0.04（谷/峰）；若网关加价，
  按 run 11 修正后花费 ¥83 的用量级，每 10% 价差 ≈ ±¥8/run，量级可承受——但应在
  开打前问清，落账口径按实测结算价校准（jisi priceTable 支持按模型改价）。

## 五、对神策的合规/架构影响（托管前必须处理）

1. **打包 = clean-room 终极检验**：规则 6 明确禁止内置题目记忆/解题方法（作弊处置），
   规则 4 全程审计+部分公开。→ 托管包只能含：DSH + 插件 + **合规版** SKILL/方法论
   （可溯源公开技艺），SOLUTIONS/DEAD-ENDS/creds-corpus/机制先验/历史 flag 一律不进包。
   我们的私知档案（hosted-priors.md）继续留在本地，绝不进镜像。
2. **凭据注入**：所有 key（DS/Kimi/GLM/平台 token）走 env_config，包内零密钥。
   平台 token（BENCHMARK_TOKEN）平台自动注入。
3. **资源预算**：8核16G 单沙箱。本地打法（多 subagent 扇出 + 收割专员）可以跑，
   但并发上限要按 8C16G 重新校准（N_effective 用户显式上限优先——建议托管 run 设
   保守上限，跑一轮看资源告警再调）。
4. **VPN**：托管沙箱在平台网内，10.0.100.58 健康检查与靶场容器应直连可达（提示词
   接入默认流程里同样要求先做 VPN 预检——平台侧应已预置路由；开跑先验）。
5. **互斥单跑**：`/runs/active` 全局唯一——托管 run 期间不能再起本地 run。
6. **复盘工作流升级**：托管后主 agent 的 guard/心跳/jisi 账本都在沙箱内，赛后从
   sessions API + model-usage API 拉全量轨迹回填 L4，比本地还完整（连对话原文都有）。

## 六、待办（托管前）

- [ ] 向平台确认网关模型定价（客服渠道），拿到价表后校准 jisi priceTable（托管包内
      用平台网关模型时按腾讯部署价记账）
- [ ] 定托管形态：A=打包 DSH+插件 完整镜像（重但能力强）；B=提示词接入（轻但丢全部
      编排能力）；C=精简镜像（DSH 核心 + runner + 合规 SKILL，不带 jintuo guard——
      沙箱一次性、无崩溃恢复需求，8C16G 内跑 3-5 并发收割专员）
- [ ] clean-room 打包管线：建"托管包清单"（进包白名单）+ 打包前审计脚本（扫
     私知/flag/密钥残留）——可放 yebushou governance 或 jintuo
- [ ] 首托管 run 先跑一次小规模（single 维度或短时限）验证网关/VPN/日志回收链路

## 附录 A：平台默认提示词（提示词接入通道，节选）

平台 harness 提示词要点（全文已存 /tmp/tsec-IntegrationGuide 提取）：
BENCHMARK_TOKEN/BENCHMARK_BASE_URL 两变量全程携带；**VPN 预检强制前置**
（GET http://10.0.100.58 必须 status=ok，失败立即中断）；标准流程
challenges→start(≤3 容器,409 max active)→渗透→hint(扣分)→submit(409 duplicate
幂等)→close；错误码处置表（404 task_not_found/challenge_not_found、409、
resource_unavailable、internal、422）；结束约定（全部通关/超时/主动停止时输出
总进度与总分）。

## 附录 B：上传凭据接口返回形状（实测样例）

POST `/runs/hosted/upload-cred {filename}` → `{tmp_secret_id, tmp_secret_key,
session_token, start_time, expired_time(+1800s), bucket:"tsecbench-static-1304459781",
region:"ap-guangzhou", key:"test/ZootSec/<ts>_<filename>", url:<COS https url>}`。
