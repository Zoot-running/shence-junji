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

## 四、模型与价格（用户问题：腾讯部署的 DeepSeek 价格差异）——已解答

- 托管沙箱的大模型走平台网关（`.tsecbench.gw` 内网域，HTTP），网关代理 **19 项白名单域名**
  （tokenhub.tencentmaas.com、api.deepseek.com、open.bigmodel.cn、api.moonshot.cn、
  ark/dashscope/qianfan/siliconflow 等全在列）。**平台不转售 token、无代付、无计费口径
  （公开面查无定价）——BYO key，模型费直接付给上游厂商，平台侧没有账单**。
- **价格差异实锤（两家官方公告交叉验证）**：腾讯云 TokenHub「原厂直供」DeepSeek：
  - Flash（`deepseek-v4-flash-202605` 即此通道 model id）：空闲 **0.05/1.5/4.5**、
    高峰 0.10/3.0/9.0 —— **比 DeepSeek 官方（0.02/1/4）贵 50%**；
  - Pro：0.15/4.5/13.5 —— 与官方**完全一致**。
  - 榜一 base_model 正是 `deepseek-v4-flash-202605`（腾讯云通道）；但其 token 成本
    交叉验算 ≈ 官方价（64~128 元）——疑走官方通道或腾讯云已调价，待实测定论。
- **我方结论**：托管时仍可直连官方 `api.deepseek.com`（网关代理，自带官方 key）——
  Flash 便宜 50%，与本地模式同价，无需买腾讯云 key；若腾讯云通道有稳定性/额度红利
  再议。jisi 价表届时按实测结算价校准。

## 四补、排行榜规则与公开机制（子代理实测补证 2026-09-12）

- **官方认证榜 = 纯托管**：「以下成绩均使用托管运行模式完成跑分，杜绝本地刷分」；
  实测 set 4 前 6 名 run 全部 `run_source:"hosted"`。本地测试榜仅支持 set_id=3、
  成绩打码仅供参考、不提供跑分详情。
- **排名按综合得分降序，用时是次要可排序列**（前端默认 sort=score desc）。当前榜一
  leixiao 100 分@15170s 是唯一满分——我们托管满分后与其同分，用时优势
  （3098s vs 15170s）在同分排序下有望居首，具体同分规则待实测。
- **上榜 = 主动公开**：`PUT /my/leaderboard-publications/{run_id} {board:"official"}`。
  公开后**全量 LLM 会话（含 thinking/工具调用）与逐题事件在免登录 API 上可被任何人
  拉取**（先例：云鼎实验室曾抓取前六名 6379 条会话）。不 publish 则只有自己登录态可见。
  → 策略：先托管打榜、复盘后再决定是否 publish；publish 前做"公开面泄露审计"。
- **反作弊公示在运行**：`GET /leaderboard/cheat-disclosures` 90 天窗口 9 条，reason 含
  「存在直接内置赛题信息和解法的违规行为」——规则 6 真会处置，clean-room 打包是硬底线。
- 沙箱级日志（`sandbox_logs` 字段）：抽样 7 个 hosted run 全为空数组——平台侧进程/网络
  日志不通过公开 API 暴露，赛后复盘主要靠 llm/sessions + run_events。

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

- [x] 网关模型定价已查明：平台不代付、BYO key；腾讯云 Flash 通道 +50%、Pro 同价；
  我方走官方 api.deepseek.com 通道（同价于本地），jisi 价表无需改（待实测结算复验）
- [ ] 定托管形态：A=打包 DSH+插件 完整镜像（重但能力强）；B=提示词接入（轻但丢全部
      编排能力）；C=精简镜像（DSH 核心 + runner + 合规 SKILL，不带 jintuo guard——
      沙箱一次性、无崩溃恢复需求，8C16G 内跑 3-5 并发收割专员）
- [ ] clean-room 打包管线：建"托管包清单"（进包白名单）+ 打包前审计脚本（扫
     私知/flag/密钥残留）——可放 yebushou governance 或 jintuo
- [ ] 首托管 run 先跑一次小规模（single 维度或短时限）验证网关/VPN/日志回收链路
- [ ] 榜前决策：托管满分后是否 publish（公开=全量会话免登录可拉；先复盘后决定，
      publish 前做公开面泄露审计）

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

## 七、诊断工作流迁移：本地文件 → 托管可见性（用户 2026-09-12 提问）

问题：值守/复盘找问题时看的本地文件，托管模式下还能看到吗？
答案：**对话级信息都能看（平台留 24 个月）；文件级信息看不到（沙箱销毁）**。
→ 托管包必须把关键产物"上移对话层"（以工具结果/收尾消息形式进 LLM 会话）。

| 本地文件（现在看的） | 用途 | 托管下等价物 | 可见性 |
|---|---|---|---|
| `/tmp/guard-runner-cmd.log`（主 agent 全文转写） | 复盘主因（本次 F28 取证、工具调用计数、调度形态分析全靠它） | `GET /runs/{id}/llm/sessions[/{sid}?from&to]`——**含 thinking/工具调用/tool 结果原文，比本地日志更全**（per-call usage 都带） | ✅ 24 个月 |
| `~/.dsh-dev/storages/llm-usage.jsonl`（花费账） | 计费对账/价表校准 | `GET /runs/{id}/llm/model-usage`（平台网关观测的权威四元组，按模型聚合）+ 每会话 usage | ✅ 更权威（平台侧口径，天然免掉本次"账本 2.4x 高估"那类问题） |
| `xiaochang-run-audit.jsonl`（心跳/终态/子代理回报） | 审计轨迹 | run_events（instance_launch/close/answer_correct/wrong）+ 会话内 tool 调用可重建 | ✅/⚠️ 重建而非直读 |
| `hufu-campaigns/*.json`（快照/DAG） | 调度账本 | ❌ 沙箱销毁即丢；仅会话内提及的部分可重建 | ⚠️ 需改造 |
| `boards/*/FINDINGS.md`、`plan/retro/war-report` | 跨代理情报/复盘交付物 | ❌ 文件本身随沙箱销毁；**内容若被 read/收尾进对话则保留** | ⚠️ 需改造 |
| guard/watch 本地日志 | 守护链健康 | 平台接管沙箱（一次性、无需 guard）；值守方改用 status/WS 实时看 | ✅ 角色替换 |
| 平台 API（status/leaderboard） | 得分/事件 | 同左 + score-timeline/result/matrix | ✅ 不变 |

**托管包必备改造（防止诊断能力降级）**：
1. **产物上移对话层**：SKILL 要求战报/复盘/计划以"最终收尾消息"输出（进会话记录），
   不能只写文件——文件带不出沙箱。发现板内容同理：关键情报要让工具结果回显。
2. **runner 审计收尾**：finish 前把 audit 摘要作为工具返回文本输出（已天然如此——
   工具返回即进会话）。
3. **花费账改用平台口径**：战后用 model-usage API 拉四元组，本地 jisi priceTable
   只做计价（计量不重复造轮子）。
4. **值守脚本换代**：本地 guard/watch 不出现在托管包；值守侧写一个"托管值守器"——
   轮询 status + WS logs + model-usage，告警规则沿用金柝语义（无重启职责）。
5. **LLM 地址改造**：DSH 配置里所有 provider baseURL 改 `<host>.tsecbench.gw`（白名单
   内域名才通）；沙箱无公网，任何直连公网的调用都会失败——打包前自检。
