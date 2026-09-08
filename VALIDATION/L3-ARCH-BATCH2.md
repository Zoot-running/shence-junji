# L3 验收：架构改进批次二（2026-09-09，8 项全落地）

> 触发：run 6 复盘结论 + 用户方案讨论定稿（虎符归调度、fanout 先完先到、长思考进 lane、靠配置不靠纪律）。

## 交付清单（逐项：提交 / 测试 / 冒烟）

| # | 项 | 仓库 | 提交 | 验证 |
|---|---|---|---|---|
| ① | provider 无关 usage-meter（会话事件总线；行带 sid/seq 读取端去重；compat 停写防双计） | jisi | `7b03056` | 26/26 测试；**live 冒烟**：sidecar 首次出现 deepseek-official/deepseek-v4-pro 行（主 agent 自己的 native 用量，旧机制盲区），jisi_usage 聚合 ¥0.06 |
| ② | hufu campaign 全量快照（原子写）+ 稳定 id 幂等恢复（resetOpen：在途项 supersede+requeue 回队列，prompt 不丢）+ finish 归档 | hufu | `0a81e61` | 32/32 测试（新增 resetOpen 恢复语义测试） |
| ③ | fanout 重构：notify 缺省（立即返回票据）/ collect+timeout / 信封 [fanout:id][model][question] / jisi_fanout_drop（abort 未结算路） | jisi | `7b03056` | 26/26 测试；**live 冒烟**：notify 立即返回 "dispatched to 1 model(s)" ✓ |
| ④ | continuable 表述归位：虎符=调度语义（长/难/已派无果任务），校场=解题语义（hard 第 2 轮起） | hufu + yebushou | `0a81e61` / `ecb665e` | 文本审阅（边界检验：虎符描述无任何 CTF 概念） |
| ⑤ | runner：稳定 campaignId（tsecbench-run-<runId>）+ boardNamespace 按 run 隔离 + pre-run sweep（sweepLegacyWorkdir，含测试）+ 门禁扩扫 cwd 遗留 + finish 调 hufu 归档 | yebushou | `7abe5c8` | 11/11 测试（新增 sweep 测试：旧工件归档、工具链/启动脚本保留） |
| ⑥ | watcher campaign-idle 告警（容器未满 + 无派单/终态活动超时 → 提醒；审计轮换自动重置游标） | jintuo | `2048e7f` | bash -n + 函数离线冒烟（heartbeat 过滤/增量/轮换均正确） |
| ⑦ | SKILL：fanout 先行 + 亲挖预算（≤1 轮/题）+ notify/drop 用法 + hard 第 2 轮起 continuable | yebushou | `ecb665e` | 文本审阅 |
| ⑧ | 一次性归档存量：boards-run6 / workdir-run6（49 条目）/ storages run6-15998（快照/审计/画像/sidecar/watch 日志）；workdir 只剩工具链+启动脚本 | 本地 | — | 已核对归档根目录命名统一 |

## 部署
- dev headless profile 已重装三插件（jisi/hufu/xiaochang-runner，00:49-00:50 构建）——下个 run 直接吃到新代码。

## 未验证项（需 run 7 实战确认）
1. 恢复闭环端到端：跑一半 kill -9 主 agent → guard 重拉 → setup 幂等恢复队列（prompt 不丢、在途项重派）。
2. fanout notify 多模型交错 + drop 中止（迟到通知按信封忽略）。
3. usage-meter 对子代理/continuable/kimi/智谱路由的全覆盖（冒烟只验了主 agent deepseek 与 fanout 派发）。
4. campaign-idle 告警在真实深挖漂移下触发（只告警不代决策）。
5. pre-run sweep 在 run 7 开局的实际效果（workdir 已人工清空，sweep 起兜底作用）。

## 设计要点（用户边界原则的落实）
- 调度权不变：一切新机制都是原语/约束/告警，没有任何"强制主 agent 怎么做"的代码。
- 虎符/校场边界：虎符只谈调度语义（continuable=长任务续战），解题判断（hard 第 2 轮）只在校场 SKILL。
- fanout 默认不阻塞：先完成先到，信封保证多次 fanout 交错可溯源，drop 停思考省 token。
- 主 agent 长思考：预算约束 + fanout 先行 + watcher 闲置告警三件套引导进 lane，不禁止。
