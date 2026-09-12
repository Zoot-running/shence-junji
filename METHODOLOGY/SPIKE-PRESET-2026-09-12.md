# Spike 报告：DSH Agent Preset（模式）机制验证 —— 2026-09-12

> 方法论新项目架构讨论的验证 spike。三个验证点全部实测（dev checkout + headless profile，
> 用户根 `<dshHome>/.agent-presets` 自建预设 + `--patch` 挂 roster + 修改 headless bundle 的
> setup 做 join——补丁已存 `shence-jintuo/patches/headless-preset-join.patch`，checkout 侧已生效）。

## 验证结论

### ① persona 叠加（complete: false）—— ✓ 叠加生效
- 实测：host persona（"You are an AI agent powered by DeepSeek Harness"）保留，预设 persona
  文本（SPIKE-MARKER-42）以附加段注入上下文（模型逐字引用了 marker 文本，且把它归类为
  "项目级预设描述"而非身份本体）。**无替换、无冲突**。
- 设计含义：模式 persona 用 `complete: false` 叠加；范围写窄（只注入方法论纪律，不重复身份）。

### ② 挂载语义 —— 两个决定性发现
1. **预设内挂 jisi/hufu = 冲突拒绝**：`service "jisi" has been registered at <shence-jisi>`
   —— headless profile 的主机层已挂 jisi/hufu，预设再挂即双注册冲突，会话拒绝启动。
   → **架构定案：模式预设不挂 jisi/hufu**；它们留在主机（profile bundles）。
2. **主机服务/工具对预设会话天然可见**：预设会话内直接调用 `jisi_usage` 成功（返回真实账本）——
   预设 agent 经 scope 链可见主机注册的工具与服务。
   → 依赖设计 = "主机侧安装 + 自家插件 lazy `ctx.get` + 响亮回退"（沿用 F8 模式），
     与 jisi/hufu 的耦合比预设直挂更松、且兼容"未安装即回系统接口"。
3. **缺失插件 = 启动前拒绝**：`row "shence-missing" names a plugin that cannot be resolved`
   —— 半组合会话不可能出现，且拒绝发生在任何模型调用之前（零花费）。

### ③ headless 选模式 —— ✓ 通路打通
- 机制：roster 插件经 `--patch` 挂入（或 profile patch）；headless bundle 的 `agents.create`
  setup 里 `agentPresets.mount(agentCtx, id)` 完成 join；预设 id 经 env `DSH_PRESET` 传入；
  用户根自动扫描。两个测试均走此通路成功。
- 设计含义：新项目的模式对 headless 战役可用（run 11 起）；web 侧会话创建 header 已有
  `agentPreset` 字段（选择器现成）。

## 对架构讨论的修正

- 用户设想"模式直接依赖集思和虎符"→ 实测修正为 **"主机挂载 + 会话可见 + 自家插件软依赖"**：
  预设直挂会与 headless profile 的主机层冲突；而主机挂载时预设会话天然可见，效果等价且更稳。
- "没安装就回系统接口"的实现点 = 新项目插件自身的 lazy get（不是预设层、也不是 jisi 改动）。

## 残留/待办

- [ ] headless 侧修改仅存于 dev checkout 源码+lib（patch 已存 jintuo/patches/），正式固化时
      随新项目交付（或按 patches/check.sh 模式纳入检查清单）。
- [ ] spike 用的 scratch 预设已删除；复现步骤见本文件与 patch。
