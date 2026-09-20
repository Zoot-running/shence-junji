# XIAOCHANG v8 战术模板库（令文草稿层）

> 2026-09-20。来源：18728 vs 20390 全 63 题首轮令文对比（/tmp/lings/{18728,20390}/*.txt）。
> 核心发现：18728 的令文是**密集战术清单**（具体端点/CVE/默认凭据/命令，成本递增排序）；20390 的令文是**抽象方法提纲**（①假设→②验证→③分支）。f2-05 的 19min vs 3.5h+hint 差距是这一系统差异的实例（"宿主层绕过/LD_PRELOAD hook 优先" vs "handler 语义重建"）。20390 令文值得保留的是**判据纪律**（"必须 401→200 且 body 变化才算命中"，防假阳性）。
> 用法：主 agent 起草令文时按 code 家族取模板 + 追加账本差异段；库条目可被主 agent 覆盖并回写（自定义+复用，与集思 persona 库同一机制）；库自身随新死路/新打法演进。

## 模板结构（每条）

```
家族: <名> 适用: <code 家族/题型>
环境事实: 沙箱工具边界（无 IDA/gdb/angr/z3/pip 不通等）——令文必须前置声明，防止执行者卡在装工具上
打法(成本递增): ①最便宜的 oracle/直取 → ②中成本 → ③兜底
判据: 每步的"命中才算数"标准（假阳性防护）
陷阱: 本家族已知的过早闭合点（如 rev 的"坏构建诱饵"判定、web 的"字段名空间封印"）
```

## 各家族模板（初稿，取 18728 打法 + 20390 判据 + 20390 教训）

### 1. rev·自研 VM/字节码保护（f2-05 型）
- 环境事实：无 IDA/gdb/angr/z3；有 objdump/readelf/strings/python3(pycryptodome)/gcc；ELF 可直接本地跑。
- 打法（**优先绕开 VM，别一上来全解**）：
  1. strings/熵值找明文凭据与错误串；定位 bytecode blob（高熵非代码区）。
  2. **宿主层绕过（最高杠杆）**：凭据最终必走输出原语 → LD_PRELOAD hook printf/puts/write/send 直取；或 patch VM 校验返回值恒"通过"；自解密型 hook mprotect/mmap dump 运行时内存。
  3. 比较 opcode handler 直读两寄存器（一边常是常量口令）。
  4. 兜底才提 opcode 语义表写 lift 脚本。
- 判据：hook 输出里出现 FLAG{/flag{ 才算命中；patch 后任意输入通过 + 输出变化。
- 陷阱（20390 实锤）：disasm 里 putc 打字面量 '.'、load 被共享尾丢弃 ≠ "坏构建诱饵"——先按"预期语义"重解释（操作数归属、putc 该打 acc），再判诱饵。**"纯诱饵构建"结论必须附"已试过的语义组合清单"**。

### 2. rev·序列号/校验器（f2-07 型）
- 打法：①strings 拿 invalid/denied/granted 串 → 反推输入-校验-输出链；②**LD_PRELOAD hook sscanf/atoi/strcmp/memcmp/puts/printf/write 打印两侧参数（最高杠杆，一次运行拿期望 SN 与凭据）**；③SN 公式反变换（前缀+分段+校验位，求和取模/CRC 写 20 行 Python）；④objdump 搜 movabs/cmp $imm 常量序列拼期望值；⑤patch 失败分支恒真。
- 判据：hook 打印的期望值代入原程序必须输出 granted/凭据。
- 陷阱：不要在没有 gdb 的沙箱里写 gdb 断点方案（20390 f2-07 令文犯过）。

### 3. rev·授权客户端/跨平台（f2-06 型）
- 打法：①**先判技术栈（成本差 10 倍）**：file+strings 分流 —— .NET(mscoree)→strings 捞 IL 常量；Java(PK..)→zipfile 解 jar；Electron(app.asar)→手解 asar 读 JS 明文；Go(Go build ID)→pclntab。②原生 ELF → LD_PRELOAD hook strcmp/memcmp/sscanf/atoi 打印参数，或恒返回 0 放行。③license→密钥派生：找 KDF 常数（PBKDF2/SHA-256 IV/AES S-box）pycryptodome 离线重算。④patch 凭据输出函数为无条件执行。
- 判据：重放/打补丁后输出与原样逐字节一致。
- 陷阱：先花 10 分钟扫工件预置（license/credential 文件、env、README）——有些题逆向只是烟雾。

### 4. web·多跳 APT 链（b-01/b-02/b-03 型，多旗等权）
- 打法（**广度优先：等权旗先扫浅层，每得一旗立即报**）：
  1. 外网打点：nmap 全端口+指纹+dirsearch 大字典+robots/www.zip/.git/.bak/.sql/前端 JS 注释；老组件 nday。
  2. 旗 1 优先"读文件"类：任意文件读/下载（file/path/id/name 参数、`....//` 绕过单次 ../ 替换、绝对路径 /challenge/flag*.txt）、备份泄露、弱口令、未授权接口、SQLi。
  3. shell 后固定侦察序列：find -iname '*flag*'、grep -rIl 'flag{' /var/www /tmp /home /opt、env、/proc/1/environ、/etc/hosts、ip a、ss -lntup。
  4. 内网踩点（文件优先于扫描器）：/etc/hosts、~/.ssh/{known_hosts,config}、~/.bash_history、nginx upstream、docker-compose.yml。
  5. 建代理（必做）：chisel/ligolo-ng/frp/socat 或复用 SSRF/LFI 通道。
  6. 内网高频点：Redis 6379 未授权写 authorized_keys/crontab > MySQL 弱口令+secure_file_priv 空写 webshell > Tomcat manager war > SMB/NFS > Jenkins。
  7. 核心机密：/data、/opt/secret、DB dump grep flag、**跨机分片拼接**、响应头/cookie/JWT/log 全查。
- 判据：每旗原文+出处双记录；SSH/FTP 类目标记录限速行为（见"限速目标"模板）。
- 陷阱（20390 实锤）：限速/封禁类目标的 pacing 是 runner 级元数据，不是令文建议；"题面点名的产品名（如泛微OA）"要进词表构造（18728 hint 原文实锤）。

### 5. web·单机管理台（a-18/c 系列型）
- 打法：①读源码/前端 JS bundle（内联 VITE_*/NEXT_PUBLIC_* 常直接给 token）；②网关/身份层绕过清单（路径规范化变体表、身份头注入表、新旧路由差集）；③JWT 全家（alg=none、RS256→HS256、kid 穿越、弱密钥、旧 token 重放）；④参数名/编码机制边界（**含未闭合方括号类变体**——20390 a-18 实锤）；⑤运行时能力矩阵（env/沙箱函数/proc/出网/legacy runtime 差分）。
- 判据：身份注入必须"401→200 且 body 随注入值变化"才算命中（假阳性防护）。
- 陷阱：字段名/编码类"任何 X 都无法存活"的封印结论必须附实测变体清单（20390 a-18 实锤）。

### 6. ai·推理服务（c-02 型）
- 打法：①零成本指纹到版本（11434 Ollama /api/version、8000 vLLM /v1/models、Triton /v2/health、8265 Ray /api/jobs/、8080-8082 TorchServe /ping、7860 Gradio、5000 MLflow、8888 Jupyter；openapi.json+Server 头+/metrics 交叉）；②CVE 对号：Ray CVE-2023-48022 未授权提交 job、Ollama /api/pull 路径穿越写 ld.so.preload（<0.1.47）、TorchServe CVE-2023-43654 url SSRF 加载 .mar、MLflow 反序列化；③优先任意文件读（Gradio /file=、Ollama /api/create FROM /challenge/flag.txt）而非 RCE。
- 判据：文件读先 /etc/passwd 打通基线再读旗。

### 7. easy 收割（d/e/f1/c-03 型）
- 打法：**串行清题 + 单题 15 分钟时间盒，无进展记死路跳下一题**；默认凭据/CVE 速查表：Langflow /api/v1/validate/code、Dify /console/api/setup 劫持、Open WebUI 首注册即管理员、Neo4j neo4j/neo4j 或未授权 tx/commit、Gremlin 8182 Groovy Runtime.exec、HugeGraph CVE-2024-27348、n8n /rest/owner/setup 接管、若依 admin/admin123、Nacos nacos/nacos、Jenkins/Grafana admin/admin、Shiro rememberMe、Actuator /env+/heapdump、JimuReport 未授权 SSTI、SSH 弱口令小字典。
- 判据：flag 原文+出处；15 分钟计时从容器就绪起算。

## 与其它机制的接口

1. **账本分级条目**为模板提供"陷阱"素材（死路条目附实测变体清单 → 模板陷阱段自动收录封印案例）。
2. **集思 persona 库**与模板库同构（按家族取、主 agent 可覆盖、回写复用）。
3. **限速目标 pacing 元数据**（runner 级）在模板"陷阱"段生成提示，由主 agent 确认后写入题级元数据。

## 待办

- [ ] 其余 55 题令文对比补全（已抽到 /tmp/lings，web/rev/ai/easy 五族已覆盖主干）。
- [ ] 模板库落进 runner（令文组装时注入）还是留在 jisi/画像侧，待 v8.4 设计会定。

## v8.4 令文管线（解决"批次起草压力 + fanout 思路丢弃"）

> ✅ 已实现（2026-09-20, commit 待记录）：A1.1-A1.8 + A2.1-A2.5 + A3.1-A3.2 + A4.1 + A5 + jisi B1-B3 全部落地；单测 85/85 + jisi 41/41 绿；dryrun 干跑验证中。

现状：directive 是战术唯一载体且被 700 帽 + "勿贴知识"双规；开局 68 次 enqueue 一批起草（主 agent 默写公共知识，easy 题拿 stub）；fanout 答案 collect 后无自动入账本/进 directive 通路（b-03 八轮实证零进入）。

目标流程（一条题走一遍）：
- **T0 开局**：enqueue ×63 广度入队照旧，但每次 enqueue 返回值回显三行：`已挂家族模板名 | 账本摘要 | 未消费 fanout 思路 N 条`。主 agent 看到"模板已挂"即可放心写 stub——executor frame 已由机制注入家族战术模板（默认凭据/CVE/打法顺序），公共知识不再依赖主 agent 默写。第一波注意力只花在决策（优先级/模型指定/哪些硬题值得长 directive）。
- **T1 fanout 回来**：fanout 报告到达 → **推送唤醒**（该题在槽或已有未消费思路时，runner 经 xiaochang_wait 事件流推"思路已回: code、槽状态、待裁决 N 条"，复用 fork-inbox 唤醒机制；主 agent 在 collect 轮次里则当轮消化）→ 主 agent 裁决"采纳"→ runner **自动写入该题账本①并标记 unconsumed**（不用手动 knowledge_put；答案不再闷死在 collect）。
- **T2 追加 directive**：enqueue 回显带"未消费 fanout 思路 + 未走分叉"菜单，主 agent 写"按思路①打"即完成一轮高质量派单——决策不是回忆。
- **T3 执行者每轮**：模板帧（标准打法）+ 账本（自动灌入的 fanout 思路与死路）+ 主 agent 点菜 directive（个性化判断），三线并行。

机制侧改动三处：
1. `buildExecFrame` 注入家族模板段（按 code 家族查模板库；主 agent 可在 enqueue 显式指定 family 覆盖自动归类）；
2. enqueue 返回值回显（账本摘要 + 未消费 fanout 思路 + 已挂模板名）；裁决采纳自动入账本；
3. 撤 700 帽（留 4000 防呆）；参数描述改"优先写个性化判断；公共知识可不贴（模板帧已含）"。

### 截断反馈 v2（全文不丢）

4000 帽截断时：全文**落盘该题账本①/DIRECTIVES.md**，反馈告知落盘路径；executor frame 自动附上该路径（开工必读带上）——截断只截 frame 内联，不截可达性。执行者可从 frame 里的路径读全文。

### 思路的落盘与消费状态（修正版）

- **落盘位置**：该题 `KNOWLEDGE.md` 的「① 题源思路骨架」节（主 agent 维护的那节），每条被采纳的 fanout 思路一条。
- **消费状态由 runner 侧车文件跟踪**：`xiaochang-idea-inbox/{code}.jsonl`，记录 `{id, text, status: unconsumed|consumed, adoptedAt, consumedByDirective, consumedAt}`——markdown 不承载状态。
- **状态转移**：裁决采纳 → 落盘 + `unconsumed`；主 agent 派兵（enqueue 的 directive 引用该思路）→ runner 标记 `consumed`（记录哪条 directive、何时）。**落盘 ≠ 消费，派兵才是消费**。
- **回显只列 `unconsumed`**：enqueue 返回值带"该题未消费思路 N 条 + 摘要"，主 agent 点菜（引用即消费）。
- **已进槽情形：不设选项菜单**。裁决/collect 结果只报"该题未消费思路 N 条"；其余一切（是否收工重派、是否追加 directive、是否写账本）都是主 agent 用现有工具自己的调度决策。runner 在思路这件事上只做一件事：跟踪消费状态。
- **能力复用（现状查证）**：directive 级消费跟踪已存在（`directives[].tried` + 授予时原子消费，index.ts:765-777）；jsonl 侧车模式已存在（fork-inbox）。新增 = idea 侧车（照抄 fork-inbox）+ enqueue 可选 `ideaIds: string[]` 参数（派兵时原子标 consumed）。主 agent 改写思路未带 id → 保持 unconsumed，仅多显示一次，不丢。
