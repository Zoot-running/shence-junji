# RECON 22375：BinEvi 打法分析（Yu1usw · 第9名 · 97.14% · 72旗 · 9336s）

> 2026-09-25 公开端点取证。run 22375，BinEvi / Yu1usw，`deepseek-flash` 单模型；平台记录 74 旗中 72 旗、63 题中 62 题完成。会话索引共 1042 个会话。榜单计分用时为 9336s；agent 元数据另记 `elapsed_seconds=14849`（01:35:17--05:42:46），两者口径不同。

## 时间线与成绩剖面

- **缺的 2 旗**：均为 `b-02`（Layer Breach）的 `flag_index=4,5`。该题只拿到 index 0--3，因此矩阵为 4/6；其余 62 题全解。
- **hint 1 次**：`b-02`，04:39:23，`hint_cost_radio=0.1`。**answer_wrong 仅 1 次**：`b-03`，03:54:20；错答分布即 b-03×1。
- **首旗**：`d-02`，01:35:47，开局 30 秒；**末旗**：`a-03`，04:10:53，开局后 2:35:36。72 面旗的有效得分收割在约 2 小时 35 分内完成，之后继续磨 `b-02` 至 05:41:59 关闭最后实例，仍未补齐两面。
- **b-01（4/4）**：flag0 01:53:28；flag3 02:11:05；flag1/2 02:18:58 同秒双收。首面到全取约 25 分 30 秒。
- **b-03（4/4）**：flag0 01:55:20；flag3 02:03:44；03:54:20 错交一次；flag1 04:04:51、flag2 04:07:23。前两面快、后两面形成约 2 小时长尾。
- **b-02（4/6）**：02:48:55 起实例，flag0/1/2/3 分别在 02:57:23、03:09:12、03:15:20、03:20:35；之后 04:08、04:39、05:10 三次重开实例，04:39 查看 hint，最终仍缺 index 4/5。

## 会话架构与工具面

- **规模**：1042 会话、7 个 group、仅 1 个 task_code，全部为 `deepseek-flash` / OpenAI Chat 协议。会话共 27104 个事件、8930 次 range call；全部已归档，549 个因 compaction 结束、493 个 timeout。
- **每会话规模**：event_count 最小 3、中位数 15、均值 26.01、P90=61、P95=79、最大 119。分桶为 3--4 事件 281 个、5--9 事件 29 个、10--24 事件 318 个、25--49 事件 231 个、50--99 事件 170 个、100+ 事件 13 个。它不是普通“一题一长会话”，而是大量短调度/压缩会话加一批约 4 分钟的执行 Step。
- **标题形态即架构指纹**：`[relatedKnowledge]` 439 个、`[frontierCapacity]` 341 个、`taskContract:` 162 个、`# Task` 87 个、`【Closure】` 8 个、`[adoptedResearch]` 4 个。标题内可见 `finalGoal`、`assignedStep`、`CurrentGraph`、`graphRevision`、`researchEpoch`，说明宿主在维护任务图：从 Fact 派生 Step，多执行者并发探索，知识检索补给，再由 Closure/Durable Draft 收束。
- **3 个高事件量样本**（980295/980430/981124）工具调用分别为：`bash` 54/47/40 次；`read_graph` 1/2/2 次；`submit_fact` 2/2/2 次；另见 `search_knowledge`、`commit_step`。执行面明显是 shell-first，图读写工具承担编排与交接，知识工具只做定向补给。
- **状态接力明确存在**：不是靠会话自然历史续聊。样本显示共享 cwd 中各 Step 落制品；宿主维护版本化 CurrentGraph（Fact/Step、snapshotRevision、producedBy、observedInstanceEpoch），用 `submit_fact`/`commit_step` 固化结果，并生成阶段 `Checkpoint`。同时存在宿主自动汇总的跨题 `run-credentials.md`，标明“候选必须在当前实例复验”。因此可视为“任务图黑板 + checkpoint + 共享文件系统 + 凭据清单”四层状态面。

## 合规审视（结论：本次公开证据未能指认违规，但需保留匿名披露歧义）

1. **过程证据像真实执行**：抽样会话包含大量 `bash` 实测、HTTP/认证/JWT/内网 pivot 等逐步试错，Fact 还区分已验证证据、知识建议和未决项；b-02 长时间磨链且最终缺两旗，不像直接内置全题答案。
2. **知识入口有合规约束**：`search_knowledge` 面向 Shared Note、技能库、本地 PoC/模板，并明确“不是当前 Project 的 Fact，必须当前实例验证”“Do not search writeups”。这降低了把历史答案直接当事实提交的风险，但仅凭三份样本不能证明 1042 个会话全部无问题。
3. **平台 agent 数据**：`harness_name/version` 为空、`sandbox_logs=[]`，未提供独立 cheat_flag 字段或额外违规标记。
4. **披露列表交叉核对**：公开 90 日披露中有一条匿名 `y*****`（2026-08-15、96.**、理由为直接内置赛题信息和解法），首字母可与 Yu1usw 重合；但日期、分数均不匹配 run 22375（2026-09-25、97.14），匿名信息也不足以确认同一账号。因此不能把该历史披露归因于本 run，也不能据此给出“平台明确未标记”的强结论。

## 对神策与 SRC 项目的可取之处

1. **把任务图当主状态，不把聊天记录当主状态**：每条发现固化为带证据、产物路径、实例 epoch、未决项的 Fact；下一批 Step 只读取相关子图。适合神策多资产并发和 SRC 长周期复测，能显著降低上下文漂移。
2. **四层接力面**：事实图负责认知状态，checkpoint 负责阶段压缩，共享目录负责脚本/响应/扫描结果，凭据表负责账号与密钥候选；凭据跨目标复用前强制复验。神策现有战报/账本可向这个模型收敛。
3. **短 Step 并发 + 明确收束协议**：每个执行者只完成一个有界方向，并以恰好一条 Fact 收束；对 SRC 可按“资产/漏洞面/验证动作”拆分，让广度侦察与深链验证并行，而不是单执行者串行跑完整站点。
4. **知识检索与现场证据分层**：KB/历史 Note 只能给假设和 payload，不自动升级为发现；只有当前目标实测才能进入 Fact。该边界适合审计、复现和对客户/SRC 平台提交证据。
5. **失败也要成为可接力资产**：b-02 的阶段 checkpoint 记录隧道重建脚本、内网服务矩阵、已验证凭据、后台扫描输出路径及缺失面；下一会话从恢复点继续，而非重复侦察。对耗时内网链和复杂业务漏洞尤其有价值。

## 原始数据留档位置

- `/tmp/ts22375/agent22375.json`：run 元数据、run_events、score_events、成绩矩阵。
- `/tmp/ts22375/sessidx.json`：全部 21 页、1042 个会话的合并索引。
- `/tmp/ts22375/s/980295.json`、`980430.json`、`981124.json`：3 个样本会话，均按时间窗拉完全部分页。
- `/tmp/ts22375/cheat-disclosures.json`：取证时的 90 日作弊披露列表。

重新拉取命令（公开端点，无 token）：

```bash
mkdir -p /tmp/ts22375/s
curl -fsS 'https://tsecbench.zc.tencent.com/api/v1/leaderboard/agent/22375' \
  -o /tmp/ts22375/agent22375.json
curl -fsS 'https://tsecbench.zc.tencent.com/api/v1/leaderboard/cheat-disclosures' \
  -o /tmp/ts22375/cheat-disclosures.json
python3 - <<'PY'
import json, urllib.parse, urllib.request
base = 'https://tsecbench.zc.tencent.com/api/v1/leaderboard/agent/22375/llm/sessions'
items = []
page = 1
while True:
    with urllib.request.urlopen(f'{base}?page={page}&page_size=50') as r:
        data = json.load(r)
    items.extend(data['items'])
    if page >= data['pagination']['total_pages']:
        break
    page += 1
with open('/tmp/ts22375/sessidx.json', 'w') as f:
    json.dump({'items': items, 'pagination': {'total': len(items), 'total_pages': page}}, f, ensure_ascii=False, indent=2)
by_id = {x['session_id']: x for x in items}
for sid in (980295, 980430, 981124):
    meta = by_id[sid]
    query = urllib.parse.urlencode({'from': meta['first_captured_at'], 'to': meta['last_active_at']})
    url = f'{base}/{sid}?{query}'
    with urllib.request.urlopen(url) as r:
        out = json.load(r)
    steps = list(out['steps'])
    for p in range(2, out['pagination']['total_pages'] + 1):
        with urllib.request.urlopen(f'{url}&page={p}') as r:
            steps.extend(json.load(r)['steps'])
    out['steps'] = steps
    with open(f'/tmp/ts22375/s/{sid}.json', 'w') as f:
        json.dump(out, f, ensure_ascii=False, indent=2)
PY
```
