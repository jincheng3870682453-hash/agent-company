# Shepherd 仓库研读报告

> 调研子代理 shepherd-deepread 产出 · 2026-11-22 · 来源:本地克隆 G:\AI agent\shepherd-repo(15 篇文档全读、src/ 30 个源文件头部与核心逻辑、19 个测试文件清单、CI 配置)。本文件为立项证据基础之一。
> 仓库地址:https://github.com/jincheng3870682453-hash/shepherd

**仓库概况**:TypeScript,Node ≥24,`--experimental-strip-types` 直跑(零构建),**零第三方运行时依赖**(devDeps 仅 typescript + @types/node)。`src/` 30 文件共 **3197 行**,`tests/` 19 个 spec 共 **1832 行**、**108 项测试**。定位:DeepSeek《Super-agent》文章四个"未来方向"(外部看门狗/投机执行/信用×诱饵/异步 ReAct 默认开启)的可运行最小闭环。研究原型,MIT,已完成 P0..P6 全部里程碑(2026-11-07/08 两天冲刺)。

---

## A. 仓库地图

### 文档(docs/)

| 文档 | 内容 |
|---|---|
| `README.md` | 30 秒论点:瓶颈=人的注意力,把会话管理/监督/验证从人手里拿走且**必须可测量**。四块空白逐块对账 + 三组对照数据表 + 谱系图(jinchen-halfdup 入口层 → HooH 出口层 → Shepherd 编排层 → 上游 fork 只读) |
| `docs/000-charter.md` | 立项章程:论点、范围(7 做/7 不做)、5 条毕业条件、7 条风险登记(R6 范围蔓延→"新增机制必须先补 ADR")、文档地图 |
| `docs/001-glossary.md` | 约 30 个术语,每词"定义+不是什么"。关键:事件/总线/会话/Ticket/Mandate/脊髓/皮层/怒气/诱饵/口吃/审计链 |
| `docs/002-architecture.md` | 架构总纲:事件总线(SQLite 追加写、seq 全序、caused_by 因果、物化视图、回放语义)、异步会话内核、Router/Worker 双层、Watchdog 独立回路、**core/adapters/apps 三层依赖规则(core 不 import adapters,core 零 I/O)**、单机三进程拓扑 |
| `docs/003-watchdog.md` | 旗舰一完整协议:7 信号遥测表、脊髓 YAML 规则(SafeEvaluator)、皮层三值判断(healthy/drifting/spinning)、怒气状态机门控(<50 放行/50-79 restart 需双证据/≥80 降级 observe)、看住看门狗、校准四剧本 |
| `docs/004-credit-bait.md` | 旗舰二:双层信用(base 慢变+cur 敏感,9 条加减分表)、三类诱饵(code/probe/environment)、采样率 `1-credit/120`、预算函数(投机预算/阈值乘数/直通资格)、**红线:信用永远不买"绕过边界"** |
| `docs/005-event-sources.md` | 事件信封 9 字段 + 三条铁律(幂等/因果/全序)、GitHub 轮询源、mock 源(flood/poison/lure/churn 故障注入)、入口过滤管线、Blob 2KB 上限与 Inline&Spill |
| `docs/006-cost-benchmark.md` | 分电表/总闸/熔断(熔断恢复只能人解)、模型分档表(Router/皮层 chat 档、Worker reasoner 档)、A/B/C 三组对照×7 剧本临床试验设计、5 条可执行成功判据 |
| `docs/007-roadmap.md` | P0..P6 每期"验收+退路",全部 ✅ 含 commit 号与实战裂缝记录;**规模预算表(core 2500 行/上限 3500,防蔓延)** |
| `docs/900-references.md` | 全部设计输入映射表:20 项本方资产→Shepherd 部件(jcf/HooH/aitoolkit/ace)与上游原型对照表 |
| `docs/910-conventions.md` | 工程纪律:编号文档按序阅读、裂缝编号不删、core 零 I/O、一处状态一个写者、白名单求值器禁 eval、错误分类(技术故障≠安全拒绝)、移植件 origin/behavior-diff 注释、node:test 零依赖立场(明确记载沙盒 EPERM 限制)、回放测试进 CI |
| `docs/ADR.md` + `docs/ADR-006.md` | 4 个已接受 ADR:ADR-001 事件溯源唯一事实源、**ADR-002 看门狗永不喂自由文本**(注入面收口)、ADR-003 乐观执行分级(只读白名单可抢跑)、ADR-004 纯算法移植 TS+执行边界留协议(HooH 一行不改);ADR-006 strip-types 语法禁令(参数属性等,CI grep 兜底) |
| `docs/promotion.md` | 发布物料:一句话介绍、推文、数据卡、awesome-list PR(#796 已提交) |
| `docs/upstream-issue.md` | 拟投上游仓库的英文 issue 草稿(上游已关闭 issues,未投) |
| `docs/announcements/benchmark-p6.md` | P6 基准报告:四张表 + 毕业条件 5/5 ✅ + 真模型冒烟 4/4 补录 + 局限声明 |

### 源码(src/)

- **core/(零 I/O 纯逻辑,~1900 行)**:`events.ts`(225 行,事件信封+37 类事件命名空间+强类型载荷)、`bus.ts`(语义核心:校验/封缄/链哈希/幂等/verifyChain)、`audit.ts`+`canonical.ts`(HMAC-SHA256 链)、`session.ts`(异步 ReAct 状态机:句柄/步间注入/取消竞态按 seq 裁决/归档)、`orchestrator.ts`(Router:受理三路/开票/订阅直通/workerDone)、`tickets.ts`(Ticket 物化视图从事件重建)、`pool.ts`(4 槽 Worker 池 FIFO+BenchmarkStats 四张表)、`filter/rule-gate.ts`(250 行,jcf 移植,SafeEvaluator 白名单求值器)、`filter/ambiguity.ts`(三路分流)、`credit/tier.ts`(双层信用,jcf 参数原样)、`credit/bait.ts`(诱饵后验+逃逸哈希检测)、`credit/budget.ts`(预算函数+红线)、`credit/link.ts`(信用→事件桥)、`watchdog/telemetry.ts`(七信号采集)、`watchdog/spinal.ts`(脊髓规则引擎)、`watchdog/anger.ts`(怒气门控表)、`watchdog/core.ts`(观察主入口:脊髓→皮层唤醒→怒气→执行)、`speculation/scheduler.ts`(投机三重门控)
- **adapters/(I/O,~770 行)**:`store.ts`(Node 24 内置 node:sqlite,WAL,BEGIN IMMEDIATE 事务)、`llm.ts`(ModelClient 抽象:ReplayModel 回放/缺记录即报错不编造)、`deepseek.ts`(真 API+录制器,录制格式=回放格式)、`cortex.ts`(皮层会话:sanitize 强类型+fail-close healthy)、`hooh.ts`(HooH-MCP stdio JSON-RPC 客户端,裁决四态 allow/deny/sandboxed/asked,denyStreak 遥测)、`github.ts`(轮询源+结构化反问)、`mock-gen.ts`(S1 剧本+判分断言)
- **apps/(组合根,~420 行)**:`runner.ts`(会话运行循环)、`dual.ts`(双层并发驱动)、`watchdog-proc.ts`(看门狗独立进程壳:独立总线+心跳)

---

## B. 机制清单

### 1. 看门狗(脊髓规则+皮层模型+怒气状态机)——完成度最高、最成体系
- **实现**:遥测采集 `src/core/watchdog/telemetry.ts`(7 信号:reportGapMs/sameErrorStreak/fileEditRepeat/tokenSlope/hoohDenyStreak/turnNoToolStreak/handleOrphans,读类不算实质进展继承 HooH STALL 规则)→ 脊髓 `spinal.ts`(确定性规则,默认五条:同错连击/无进展/往返编辑/边界试探→escalate/烧钱斜率;when 表达式走 SafeEvaluator 白名单禁 eval)→ 皮层 `src/adapters/cortex.ts`(独立会话,**输入只有强类型数字,schema 外字段直接丢弃**,解析失败 fail-close 返回 healthy 不放咬)→ 怒气门控 `anger.ts`(介入+30/正常-16/时间-10每30s;<50 全放行,50-79 restart 需脊髓+皮层双证据否则降级 nudge,≥80 一切降级 observe"咬怕了")→ 每次介入带 evidence[] 落 HMAC 审计链。
- **独立进程**:`src/apps/watchdog-proc.ts` 独立总线+60s 心跳,heartbeatHealthy 四象限测试。
- **实测数据**(mock LLM,见 `docs/announcements/benchmark-p6.md`):S2 死循环漏放 0;S4 深思考误杀 0;S3 偏题皮层兜底检出→nudge;自愈闭环 14 事件可回放(`scripts/self-heal-demo.mts`:死循环→nudge 阶梯六轮→怒气 100 门控→restart→新会话交付)。
- **关键约束**:皮层证据摘要只进审计链给人看,永不回流为任何 LLM 输入(ADR-002,防自我强化与注入)。

### 2. 信用×诱饵
- **实现**:`src/core/credit/tier.ts` 双层信用(base 慢变+cur 会话敏感,新实体 30,时间衰减向均值回归 MAX_DECAY=20 封顶)+ 9 条加减分表(谎报 ready **-25**、漏放 -20、重犯 -15);`bait.ts` 三类诱饵,**逃逸检测=断言文件 SHA256 比对**(删测试/改断言直接抓,AST 签名的语言无关等价);采样率 `1-credit/120`(信用 30→75% 全查);`budget.ts` 预算函数:投机预算 `50k×(credit/100)²`、Watchdog 阈值乘数 `1.6-0.9c/100`、直通资格 ≥60,**红线测试断言预算函数不含任何权限字段**。
- **实测**:S6 谎报捕获率 100%(诚实/谎报信用分化,`tests/s6-s7.spec.ts`);采样率自适应联动验证。
- **哲学根源**:Word体系"这套系统不依赖任何 AI 的诚实,只相信检测逻辑和人类的终审权"(`docs/004-credit-bait.md` §2)。

### 3. 投机执行
- **实现**:`src/core/speculation/scheduler.ts` 双通道触发(显式规则+评估器提议)、三重门控(预算/并发/池干涸即静默永不挤占正式工作)、交付门槛=诱饵/复现验证、would-do 日志+命中率仪表盘、结算未花完回池。
- **实测**:S7 命中率 0.50,低信用源 0 打扰(预算平方曲线自然淘汰)。
- **注意**:这套机制与新设计"不做常驻主动找活、触发只报不开工"**方向相反**,不建议搬。

### 4. 异步 ReAct 内核
- **实现**:`src/core/session.ts`+`src/apps/runner.ts`:start/stop 句柄、结果步间注入(永不打断生成)、取消竞态**先到总线者胜(按 seq)**,迟到事件标 superseded 入库不投递(事实不丢弃)、孤儿句柄宽限清理、归档压缩(SimHash 判重思想)、乐观执行只读白名单(ADR-003,模型声称乐观但工具不在白名单→不乐观,fail-close,有测试)。
- **实测**:3 工具乱序回/竞态端到端/**重放一致性(同剧本双跑逐字段相等)进 CI**(`tests/e2e.spec.ts` 第 3 项)。
- **短板**:`runner.ts:78` 构造 TurnRequest 时 `messages: []` ——真实 LLM 的上下文组装实际是空壳,Session.context 数组存在但没有接进提示词;这解释了为什么真 API 只做了皮层+受理冒烟,没跑完整 Worker 任务。

### 5. 事件总线与审计链
- **实现**:`src/core/bus.ts`(校验/封缄/链哈希,缺 causedBy 直接 EventRejected,调用方不可自带 auditHash)+ `src/adapters/store.ts`(node:sqlite 追加写,events/natural_keys/meta 三表,WAL)+ `src/core/audit.ts`(HMAC-SHA256 链式,改一条即断链)。
- **实测**:篡改历史→verifyChain 定位断点;500 事件压测 seq 连续全链完好;重开库链头延续。
- **语义价值**:一切状态=事件物化视图,崩溃后重放重建(`tickets.ts` rebuildTickets 有重放一致性测试)。这就是一套现成的**跨对话状态账本**底座。

### 6. HooH-MCP 对接
- **实现**:`src/adapters/hooh.ts` stdio JSON-RPC,协议对接 HooH 一行不改;每次调用产 `hooh.decided` 事件;PERMISSION_REQUEST→asked(headless fail-close 拒绝);安全拒绝计数上浮→Watchdog 遥测。真 server v1.2.1 联调全通(`scripts/hooh-smoke.mts`/`hooh-rollback-demo.mts` 破坏→恢复闭环)。
- **意义**:出口裁决/快照回滚/沙箱是外部 Python 服务,Shepherd 只留协议边界。

### 7. 入口过滤管线(纯规则,零 LLM)
- **实现**:`src/core/filter/rule-gate.ts`(jcf 移植:YAML 热更新纪律 validate-then-swap 失败保旧、SafeEvaluator 白名单表达式)+ `ambiguity.ts`(乱码→零成本静默/模糊→模板澄清/清晰→慢路径)。结构化源(GitHub Issue)旁路模糊分流。
- **实测**:S1 剧本 220 Trigger 中 200 噪音全部被滤,20 请求全受理;与 Python 原件逐字段对拍(含"asdfgh 匹配正则算有词"这种边界,`tests/rule-gate.spec.ts` 第 1 项)。

### 8. 成本护栏
- `meter.charge`/`meter.tripped` 事件进总线;分电表(超预算降速不杀)、总闸(熔断停新不开旧)、熔断恢复只能人解。mock 基准下逐 ticket 成本可核。

---

## C. 与 22 条设计的映射表

**图例**:✅强支撑(有可复用实现) / 🟡部分沾边(有相邻机制需改造) / ❌空白

| # | 设计条目 | 判定 | Shepherd 对应物 |
|---|---|---|---|
| 1 | 任务=持久容器(对话+记忆+状态账本,跨对话) | 🟡 | Ticket"在,会话可死可换"(`docs/001-glossary.md`)+事件总线重放重建(`src/core/tickets.ts`、`src/adapters/store.ts`)+归档蒸馏(`src/core/session.ts` ArchiveEntry)。**缺**:跨对话记忆容器、对话挂载 |
| 2 | 动态边界:申请扩张→审批→持久并入→可收缩 | ❌ | 无。HooH 权限档是静态的;Mandate 划权限但无申请-审批流程 |
| 3 | 分级审批链:上级代批,越界升人 | 🟡 | Watchdog 介入分级 observe→nudge→restart→escalate + boundary-probe 直接 escalate(`src/core/watchdog/anger.ts`)+人类终审权(`docs/910-conventions.md` §5)。**缺**:"上级代批"概念不存在 |
| 4 | 主 agent 派发子文件夹给写手 | 🟡 | `router.dispatched` 派发 Ticket+Worker 隔离目录(`docs/002-architecture.md` §4)。派发对象是 Ticket/mandate 不是文件夹 |
| 5 | 写手工具极简:只读+写,无执行无网络 | 🟡 | Worker 每次工具调用过 HooH,readonly 权限档(`src/adapters/hooh.ts:29`);Router 无写工具。**缺**:无"只有读写文件"的硬工具面 |
| 6 | 执行权集中:统一测试 agent 群唯一能跑代码 | 🟡 | HooH 沙箱=唯一执行口(sandboxed 裁决四态)。**缺**:独立测试 agent 群 |
| 7 | 二级 agent 层做整合分配,主 agent 只管战略 | 🟡 | Router/Worker 双层。**缺**:第三层;Router 兼管对外发言与受理,不是纯中层经理 |
| 8 | 边界审批按文件重要性分级(核心/共享/叶子) | ❌ | 无。最接近的是诱饵的断言文件哈希保护,但不是审批分级 |
| 9 | 云端审查 agent 生产链外独立验收 | 🟡 | Watchdog 独立进程+判分器只读总线/blob 不读自述(`docs/910-conventions.md` §6)。可复用"独立进程+结构化输入"架构;但 Watchdog 监督行为,不验收交付物质量 |
| 10 | 三级化:CEO/经理/手+分档省钱 | 🟡 | **模型分档表直接可用**(`docs/006-cost-benchmark.md` §1:Router/皮层 chat 档、Worker reasoner 档);层级只有两层 |
| 11 | git 式发布-同步链,零进程文件远程 | 🟡 | 事件总线=追加写/全序/因果/回放,语义同构于"提交→合入→全员同步";GitHub 轮询+API-only PR 工作流(跨机等价形态)已通。**缺**:本地 git 服务、merge 语义 |
| 12 | 主 agent 视察权:只读拉状态 JSON,蒸馏入账本 | 🟡 | `watchdog.observed` 状态 JSON 快照入总线(`src/apps/watchdog-proc.ts:56`)+归档压缩"一行摘要,原文在总线"。**缺**:"看完可丢/暂存"的交互流程 |
| 13 | 罢免权:证据+人类批准 | 🟡 | restart 自带 evidence[]+快照回滚+重开(`src/core/watchdog/core.ts`),escalate 给人。**差距**:Shepherd 的 restart 是**自动执行**不经人批;需把门控改为"主 agent 发起+人批" |
| 14 | 专家评审团(横向参谋部) | ❌ | 无 |
| 15 | 全局诚实宪法:不得隐瞒、抽查对账、隐瞒=罢免证据 | ✅ | **最强对应**。信用×诱饵后验整套(`src/core/credit/bait.ts`、`tier.ts`):谎报 -25、删测试逃逸检测、采样率随信用自适应抽查、"隐瞒/谎报→信用打穿→预算清零→自动淘汰"。哲学完全同源 |
| 16 | 按需激活:用完即散,上岗领状态下岗写回 | 🟡 | Worker 池按需+槽位释放(`src/core/pool.ts`);会话状态全在总线可重放="领状态/写回"雏形。**冲突**:Router 是常驻会话,与新设计"没任务=没进程零 token"相反 |
| 17 | 接口登记制度(契约先行/域内/跨域/专家团批) | ❌ | 无。Mandate 是授权范围不是接口契约 |
| 18 | 专家团对抗性义务(必须反驳+替代方案) | ❌ | 无。双证据门控有一点对抗味,但不是专家团 |
| 19 | 本地微型 git 服务器 | ❌ | 无(总线是 SQLite;git 仅以 GitHub API 外部形态出现) |
| 20 | 同步通知=MD 文件挂仓库边 | ❌ | 无(通知是 `reply.sent` 事件,无 MD 文件形态) |
| 21 | 横向通信需上级审批 | 🟡 | "Worker 不直接对外说话,一切输出经 Router"(`docs/001-glossary.md` Worker 词条)=横向通信收口。**差距**:Worker↔Worker 同 Thread 自行协调不审批,与新设计相悖 |
| 22 | base_commit 三路合并/存量限额/报废效应 | ❌ | 无 git 合并机制。唯一沾边:Worker 池并发上限(4)的槽位控制 |

**附带决策映射**:
- **滤网用规则引擎/白名单而非 LLM** → ✅ `src/core/filter/rule-gate.ts`(250 行,已与 Python 原件对拍)+`ambiguity.ts` 直接搬。
- **规则哨兵(默认关、触发只报不开工)** → 🟡 Watchdog 的 `observe` 级+脊髓规则表达式正是"只记录不动作"形态;事件类型命名空间已含全套钩子。
- **专家团限频(里程碑触发)** → 🟡 皮层唤醒模式("只在脊髓命中时唤醒+15min 兜底",`src/core/watchdog/core.ts:66`)是现成的限频样板。
- **没任务=没进程零 token** → 🟡 事件驱动+SQLite 状态天然支持冷启动;常驻 Router 是反例需改。

**统计**:✅ 强支撑 1 条(15);🟡 部分沾边 12 条(1,3,4,5,6,7,9,10,11,12,13,16,21);❌ 完全空白 9 条(2,8,14,17,18,19,20,22,及组织结构核心项)。**空白集中在"组织结构与同步"域,Shepherd 的强项集中在"监督与诚实"域——两者几乎不重叠。**

---

## D. 工程质量评估

**测试(优)**:108 项/19 文件,1832 行测试对 3197 行源码;全部走 Node 内置 node:test 零测试依赖(910 §6 记载了选型原因:沙盒内 esbuild 管道 EPERM);测试不打真 API,mock 回放优先;**重放一致性、审计链篡改检测、预算函数红线(不含权限字段)都进了断言**;判分器与被测对象隔离(只读总线/blob)。CI 三步:typecheck+test+ADR-006 参数属性 grep(`.github/workflows/ci.yml:24`)。

**依赖策略(优)**:运行时零第三方依赖——node:sqlite(内置)、fetch(内置)、crypto(内置);唯一 devDeps 是 typescript。零构建(strip-types 直跑),代价是 ADR-006 记载的语法禁令(参数属性等四次撞墙,靠 CI grep 兜底)。

**文档纪律(优,罕见级别)**:编号文档按序依赖、ADR 先于代码、裂缝编号不删历史、移植件文件头强制 `origin:` + `behavior-diff:` 注释并要求对拍、基准剧本无判分断言不许进主目录、"文档与实现冲突以文档评审为准"。`docs/900-references.md` 的部件溯源表可查每个机制的出处。风险登记 R6"范围蔓延"配了行数上限表(core 3500 上限,实际 3197 含 adapters/apps)。

**明显短板(实)**:
1. **真实模型验证未做**:全量 S1-S7 基准是 mock LLM;真 API 只有 4 项冒烟(皮层 spinning+受理三态)。README/promotion 明确承认。mock 数字只证机制成立,不证效果分布。
2. **上下文组装是空壳**:`src/apps/runner.ts:78` 每轮 `messages: []`,Session.context 没接进提示词——完整真实 Worker 任务链从未跑通。
3. **UI 缺失**:escalate 的落点是"面板置顶",但面板本身没建;人机交互面只有 CLI 脚本。
4. **简化实现与文档有距离**:错误指纹实际是 `tool:error` 字符串(文档说"类型+参数哈希");tokenSlope 是 3 点滑动均值(文档说斜率);Watchdog 遥测靠 apps 层手动投喂,生产接管未验证。
5. 单机三进程,存算分离仅协议预留;Watchdog 皮层兜底巡检简化为全局间隔。
6. 无授权/多人协作/权限体系——它默认单一人类操作者。

---

## E. 结论:抽取机制做新底座,不建议直接扩建

**建议:新建仓库,把 Shepherd 当"机制供应商+工程纪律模板",按清单移植件级搬运。**

理由:
1. **领域不重叠**。22 条中 Shepherd 强支撑的只有第 15 条(诚实宪法)和若干 🟡;而组织结构的核心条目(2 动态边界、8 重要性地图、14/18 专家团、17 接口登记、19-20 同步设施、22 三路合并)**全部空白**。扩建=在监督型编排层上长出组织型系统,两层概念的 Router/Worker/Ticket 骨架会持续打架(如第 7 条要求拆掉 Router 兼职、第 21 条要求废除 Worker 自行协调)。
2. **三处方向性冲突**:Shepherd 的 Router 常驻会话 vs 新设计"没任务=没进程零 token";投机执行主动开工 vs 新设计"触发只报不开工"(且投机执行与"写手无执行权"冲突);Worker 同 Thread 自行协调 vs 新设计"横向通信须上级批"。
3. **体量与成熟度**:3197 行研究原型,真实模型全链未验证,直接当新项目底座风险不对称;而它的机制件(core 层零 I/O 纯逻辑)拆出来移植成本极低——它们本来就是按"可移植、可对拍、零依赖"纪律写的。

**复用清单(按优先级)**:
- **P0 直搬(几乎零修改)**:
  - 事件总线三件套:`src/core/bus.ts` + `src/core/audit.ts` + `src/adapters/store.ts`(+`canonical.ts`)——幂等/因果/HMAC 审计链/回放,直接当新项目的**跨对话状态账本+罢免取证**底座(服务第 1、12、13、15 条)。
  - 规则滤网:`src/core/filter/rule-gate.ts` + `ambiguity.ts`——滤网纯规则+白名单求值器(服务"滤网不用 LLM"决策),已对拍。
  - 看门狗机制件:`src/core/watchdog/*`(telemetry/spinal/anger)+`src/adapters/cortex.ts`——脊髓规则哨兵(触发只报不开工→observe 级)、限频唤醒模式(专家团限频)、介入分级+双证据门控改造成**罢免审批链**(第 3、13 条)。
- **P1 改造后搬**:
  - 信用×诱饵:`src/core/credit/tier.ts` + `bait.ts` + `budget.ts`——诚实宪法现成实现(第 15 条),加减分表按新组织重定义。
  - 异步会话内核:`src/core/session.ts` + `src/apps/runner.ts`——任务驱动执行基座(第 16 条),删常驻 Router,补真实上下文组装(这是 shepherd 自己的欠账)。
  - Worker 池:`src/core/pool.ts`——按需激活+槽位限额骨架(第 16、22 条存量限额)。
- **P2 参考不搬码**:
  - 模型分档表(`docs/006-cost-benchmark.md` §1)→ 三级组织的思考档/手档定价(第 10 条)。
  - 判分器纪律(只读证据不读自述)→ 云端审查 agent 的验收基座(第 9 条)。
  - `docs/910-conventions.md` 全套工程纪律 + ADR 流程 + mock 回放测试法 → 新项目仓库规范直接以它为蓝本。
- **不搬**:投机执行(`src/core/speculation/`,方向相反)、Router 受理评估、订阅直通的自行协调、mock-gen 剧本体系(新项目需重做剧本)。

**一个外部依赖提醒**:Shepherd 的执行边界在外部 HooH·互(Python MCP 服务,裁决/快照/回滚/沙箱),不在本仓库。新项目若沿用"出口裁决独立服务"的架构(与第 5、6 条契合),需要单独评估 HooH 的接入,或按同样协议哲学重做轻量版。
