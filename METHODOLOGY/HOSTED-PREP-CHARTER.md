# 立项书:托管模式前置三件套(clean-room 打包管线 / SKILL 改造 / 会话回放重建器)

> 2026-09-12,主调度草拟,用户已确认开工("可以,开工吧")。
> 背景与探索事实:见 `VALIDATION/HOSTED-MODE-EXPLORATION.md`(§5 合规影响、§7 诊断迁移)。

## 1. 目标(字典序:唯一硬指标在前)

- **硬目标(唯一)**:三件套全部落地、可运行、各自通过最小验证——
  使 run 12 托管局具备:合规镜像可打包可审计、战役 agent 知道托管规矩、
  赛后能用平台会话重建本地诊断工作流。
- 次要 1:时间(并行三路,今天内收口);次要 2:不破坏现有本地模式流程。
- **非目标**:不打托管局(另立项);不做镜像本体构建(管线先行,镜像待 run 12 立项);
  不改平台行为假设未经实测的部分。

## 2. 约束

- 硬:托管规则 6 红线(禁内置题解/记忆)、凭据只走 env、协议 7.2(不逆向)。
  审计脚本自身不得引入私知内容;公开仓库不落密钥/flag。
- 软:复用现有件(governance.ts 规则、jintuo 脚本风格、SKILL 既有合规节);
  "靠配置不靠纪律"——审计进打包流程,重建器进赛后流程,SKILL 进开战令。

## 3. 验收口径(done 定义)

- **W1 clean-room 打包管线**(yebushou):`packaging/hosted-pack-audit.py` 可对目录
  全量扫描(flag 值/密钥/已知凭据/私知档案名与内容标记/历史题解路径),退出码
  非 0 即阻断打包;`packaging/HOSTED-PACK-MANIFEST.md` 白名单/黑名单成文;
  自测(构造违规样例目录 → 审计命中 → clean 目录通过)。
- **W2 SKILL 改造**(yebushou xiaochang SKILL):新增"托管模式须知"节——
  ①产物上移对话层铁律(战报/复盘/计划以收尾消息输出,发现板关键情报回显)
  ②大模型地址 .gw 自检清单 ③托管红线与平台自动注入变量说明。文本可被
  agent 直接引用执行(行为级验收留给 run 12)。
- **W3 会话回放重建器**(jintuo):`hosted-replay.py` 从平台 sessions API 拉全量
  会话,产出 timeline.jsonl / file-tree(按 write/edit 工具调用重建)/ 交付物摘要 /
  usage-summary;用合成 fixture 单测通过(不拉真实题解内容,clean-room)。

## 4. 规则域(红线前置)

- 红线:见 §2。W3 的 fixture 测试**不得**拉取/引用榜上其他 agent 的解题内容;
  schema 取证已完成(键名级),实现只按 schema 解析。
- 变更条款:目标/红线变更 → 基于当前完成工作重新排计划。

## 5. 假设日志

- 假设 1:托管局 run 12 走"官方 DeepSeek 通道(.gw 代理 api.deepseek.com)",与本地同价——已查证。
- 假设 2:平台 sessions API 的 schema 稳定(已实测键名),重建器按防御式解析实现。
- 假设 3:托管镜像不含 jintuo guard(沙箱一次性,无重启语义)——已裁定(探索档案 §5)。
- 假设 4:SKILL 行为级验收延到 run 12(文本级验收在本立项内)——用户方向认可。

## 6. 计划(DAG + 并行度)

```
W1 打包管线(yebushou)  ─┐
W2 SKILL 改造(yebushou) ─┼─ 三路独立,并行;共享同一个 yebushou 仓,提交分离
W3 会话重建器(jintuo)   ─┘
W0 立项书(junji,本文档) —— 已完成先行
W4 三仓 commit+push 收口 —— 汇聚点(依赖 W1/W2/W3)
```

- 关键路径:W1/W2/W3 并行 → W4。执行分工:W1/W2 主 agent 亲做;W3 委托子代理
  (spec 已备:schema 键名 + 端点 + 输出契约 + fixture 测试要求)。
- 并行度上限:3(两仓无共享状态,冲突面仅 yebushou 同仓不同文件)。

## 7. 复盘回填区(战后)

## 7. 复盘回填区（2026-09-12 当日回填）

- **预期 vs 实际**：三件套全数交付且各自通过最小验证（W1 审计自测 4/4 命中、
  W2 文本落地、W3 29 单测全绿 + 本地 http.server 全链路 wire 级验证）。
  比预期多出的产出：W3 的 `--dry-run`/`--only-summary`/`--public` 三模式与
  契约发现 7 条（防御式解析已消化）；总用时 ≈1 小时（三路并行 + 委托）。
- **决策-结果对照**：
  - D1 三路并行、W3 委托子代理（spec 前置：schema 取证已完成）→ 一次交付零返工 ✅；
  - D2 W3 测试用合成 fixture + 本地 http.server，**不拉真实题解内容** → clean-room 保住 ✅；
  - D3 真实凭证连测延到 run 12 首用（--only-summary 探活）→ 风险显式挂账 ✅；
  - D4 打包审计复用 governance 规则 + 新增路径级黑名单 → 一次成型 ✅。
- **根因**：无失败项。平台契约增量（详情层 `steps` 键、from/to 依赖时间戳字段、
  edit 完整内容判定、working_directory 依赖）由 W3 防御式解析消化并写入 README。
- **教训分类**：
  - A 通用方法论："先取证 schema 再委托子代理"（spec 完备 → 一次成）——升格候选；
  - B 机制先验：平台 sessions API 的 7 条契约细节 → 进
    `VALIDATION/HOSTED-MODE-EXPLORATION.md`（已附指针）；
  - C 一次性事实：路径清洗启发式/分页退化判定，已在 jintuo README；
  - D 平台缺陷：无结论（未做真实凭证连测，留待 run 12）。
- **收口**：junji `50383cf`（立项）+ yebushou `174622c`（W1+W2）+ jintuo `d173cc6`（W3）。
  三件套就绪，run 12 托管局可立项。

- (W1/W2/W3 完成后回填:预期 vs 实际 / 决策-结果 / 根因 / 教训分类)
