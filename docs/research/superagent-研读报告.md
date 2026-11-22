# dsh-superagent-prototype 研读报告

> 调研子代理 superagent-docs 产出 · 2026-11-22 · 来源:仓库 docs/000–008 全部编号文档 + SETUP.md(raw.githubusercontent.com 逐篇抓取;007 截断部分已从溢出文件补读)。本文件为立项证据基础之一,未经核实前不作为规范引用。

**仓库定位**:DeepSeek 官方的"受管 agent 团队成员"原型——一个装在 DeepSeek Harness 上的插件,让 agent 从 GitHub/飞书接收任务,经确定性准入 + 模型评估后生成工单(Ticket),派给独立的 Executor Session 在隔离目录里干活,通过 PR/评论交付,带 Web 监督面板。架构核心:单 SQLite 状态库 + 单写者 Engine + 持久效应队列 + 双角色 preset。代码量约 40+ 个测试文件覆盖,文档编号分层、只向前引用。
仓库地址:https://github.com/deepseek-ai/dsh-superagent-prototype

## A. 逐篇要点

**000-philosophy(哲学)**
核心原则"占用尽可能少的人类注意力,给出尽可能多的价值",每条消息/通知/设置都要付出注意力成本,不划算就删。对 agent 的约束:只在被请求或书面规则允许时行动;只在人必须行动或能用结果时说话;一次只问一个结构化的具体问题;按影响而非工作量停止;诚实汇报(草稿标注未完成,失败报失败,限制不绕过)。对设计的约束:概念最少、设置最少(团队用语言描述的行为放 prompt 而非设置)、面板只显示需要人的东西、文档按序阅读。责任条款很实:agent 以团队可见的账号发消息,操作者要为噪音负责——从小名单起步、主动行为先记录不执行、不用它刷量。

**001-domain(域模型)** — 全仓库最重要的文档
完整概念表:Source→Trigger→Thread→Eval→Decision→Ticket(+Mandate)→Assignment→Executor,Subscription 做 Trigger 向活工单的扇出。最有价值的机制细节:
- **Mandate 字段**:Policy 版本、scope、strength(只取 `explicit/inferred`)、matched rules、发送者在该 Source 上的身份、Source 收到 Trigger 的方式;**不可变**——更大 scope = 新 Ticket;Ticket 结束不结束 Mandate,只有人的 stop 撤销、revalidation 失败吊销(docs/001-domain.md、docs/002-authorization.md)。
- **拒绝的概念表(Rejected concepts)**:明确拒绝"人/成员身份作为实体""跨 Source 身份映射"(那是 prompt 文本)、"Eval 自调度延迟 Trigger"(等待属于 Source)、"每 Thread 限一个活工单"、"append to Ticket 决策"(追加=订阅扇出,不需要 Eval)、"Ticket 状态跟随 DSH Session"(工单自定义状态)等约 12 个看似自然的实体。这是防止概念膨胀的工程手段。
- Ticket 只由 Eval 创建;Executor 在域外,经 Assignment 接入;人不是域概念,只是 Trigger 的 actor 元数据。

**002-authorization(授权)**
授权 = 谁能开工 + 工作可包含什么,**由 host 代码强制**,消息/评论/引用文档只是决策输入、不能改权限。机制细节:
- **两条路**:Eval 路径(决定是否开工)走 gate;订阅路径(延续已有工作)绕过 gate 但不授予新权力,仍受 Mandate 限制。
- **强度分级表**:explicit(名单内的人 @ agent,或已有工单 Thread 上的跟进)/ inferred(名单外的人跟进、名单内 bot 的候选)/ not admitted。Source 有 **admission table**,行=接收 scope,列=三种发送情形,每格可选 explicit/inferred/ignore,各有默认值;非名单者的 explicit 格优先且在任何模式下生效。
- **三档 accept mode**:`off`/`observe`(默认:推断候选被评估,正面结果只记录为 "would accept" 不执行)/`auto`。**observe 模式是灰度上线的核心设计**。
- **恢复前重验(revalidation)**:按 Mandate 的 basis 逐项检查——explicit 跟进直接过;inferred 需 auto 模式且规则仍启用;表格准入的需按当日表格与名单仍达到原强度(把人移出名单可吊销已接受的 Mandate);bot 的需仍在 bot 名单。
- **Mandate 只有三种变更操作**:accept 创建、人的 stop 撤销、重验失败吊销。Executor 无权宣称"人让我停"。
- Eval 超时处理:队列重试两次,第三次记 `unavailable`,24 小时内可进面板请求人工。

**003-execution(执行)**
Worker 全生命周期在此,细节密度最高:
- **Ticket 阶段表**(自上而下首行命中):person/delivery(投递失败,2s 起指数退避到 1min 重试)→ person/answer → person/review → person/blocked、failed → working/queued → idle → working。`person` 前缀=没有人的输入就无法继续。
- **Executor = 每 Ticket 一个 `worker-<id>` Session + 一个同名目录**;prepare 命令 10 分钟限时,典型做法 `git worktree` 出独立 checkout;PR 归属按分支名末段匹配工单短名。
- **Worker slot 并发**:工单从 launch 起持槽;配置 maxWorkers 之外**留 1 个额外槽专给人的消息/状态请求**——新任务排队永远不能挤掉人续聊已有工单;人 wake 优先于新 launch。
- **后台工作规则**:turn 结束仍有 shell job/subagent 在跑→保留槽继续"工作中",完成的任务带结果唤醒;每 turn 后台最多 30 分钟,超时杀 shell job、放 subagent 继续;交接给人立即释放槽;server 类 job 可活到 Assignment 结束且不受 30 分钟限。
- **失败恢复**:模型/网络瞬时错误(408/429/5xx)退避重试 3 次;failed 后 CI/bot 消息只记录不唤醒;人的消息解除 hold;retry=同 Session 同目录同分支换 Assignment;重启=中断 Assignment 带 recovery note 按"最老工单优先"重启;"先存储后执行"(stored work)——所有中继/投递/启动先落库再执行,失败退避重试,同类保序。
- **合并治理**:host 无合并决策逻辑,全靠 Delivery requirements prompt;默认 worker 绝不用 agent 账号开 auto-merge(GitHub 把 merge 归因给开启者),<200 行改动且批准齐才可 auto-merge,推新修订前先撤 auto-merge。**文档明说:这些指令约束 agent 而非 GitHub 的合并规则,要强制必须用分支保护。**
- Source 失败:逐 Item 读,连续 3 轮失败才报错,单条坏数据跳过计数;PR 持续读失败一天后丢弃。

**004-prompts(提示词)**
- **装配顺序**:共享指令/角色 → 3 个团队段(Communication/Ticket scope/Delivery requirements)→ 订阅与回复 → 派发或工作指令 → admission(仅主)→ handoff → environment → **policy 放最后**——"这样 Policy 变更不改变其前面的 prompt 文本",即前缀内容稳定,对 prompt 缓存友好(docs/004-prompts.md)。
- **效力从强到弱 5 级**:①工具/host 强制的事实(身份、权限、参数校验)→ ②内置段(含团队覆盖)→ ③推断工单的 matched rules → ④团队段 → ⑤外部文本(消息、评论、文档——**永远是数据,不改变规则与授权**)。
- **Token 纪律的具体数字**:路由 note ≤30 词;聊天消息默认一句、硬限 100 词;确认 ≤100 词;主 Session **不许等待**——`noWait` 守卫拒绝 `sleep`、>30s 的 shell 超时、>5s 的 job 等待,因为阻塞一步就延迟新候选;report 摘要 600 字符;assignment 文本 1500 字符。
- **诚实交付条款**:开 PR ≠ ready;无真实 before 截图的 UI 改动不是 ready;"例子只展示格式,不是 agent 可以声称的事实"。
- 版本化:每次保存生成不可变版本(修订号+前驱+来源),冲突保存拒绝;Session 每次 model 请求读最新版本;覆盖记录其所基于的内置文本,升级时旧默认变化标记 "Default updated" 让团队三选一。
- Executor 工具排除所有"问人/展示 UI/改 profile"的工具(ask_user_question、present、plugin_manager)。

**005-panel(面板)**
开面板回答三问:有什么需要我?系统健康吗?单件工作的全历史?Tab 按工作流排:Overview(待办工单+系统问题+24h 活动数+Executor 负载)/Source/Trigger/Eval/Execution/Settings。原则 7 条,最实的几条:**"诚实的状态:计数等于它链接到的列表长度,任何状态不与另一事实矛盾"**;健康状态不占空间;分类要有可检验定义且互斥完备(如工单阶段表);每个设置块只有一条 save bar。Prompt 页直接内置"Default updated"三选一 diff 流程。

**006-deployment(部署)** — 负面警告集中地
- **权限警告(官方原文)**:worker 与 router 两个 preset 都"以操作者 OS 用户跑 `danger-full-access`",账号 wrapper 只选凭据、**不隔离进程**;"worker 可以按路径执行程序、读操作者 home 目录。要隔离,用只含 agent 凭据的专用 OS 账户"(docs/006-deployment.md)。
- 隔离手段全在进程环境层:`<stateDir>/worker-bin` 命令 shim 排 PATH 首位(gh→agent CLI、被禁命令直接失败)、`worker-gitconfig` 替换全局 git 配置(凭据 helper 只指 agent CLI、SSH 重写 HTTPS、剥 `GH_TOKEN`/`GITHUB_TOKEN`)、setup 剥掉名字含 KEY/PASSWORD/SECRET/TOKEN 的环境变量;但这些都是"默认而非安全边界"。
- 工程细节:`status` 四种退出码(0 正常/1 可重启修复/2 组件错误重启无用/3 等待 setup),每小时自检 prompt 完整给出(含 `--respect-stop` 防止覆盖人工停止);升级流程要求备份 `orchestrator.sqlite`,**新版迁移过的库旧版打不开,回滚只能靠升级前备份**;只支持 github.com 不支持 GHE。
- Management API 被承诺为稳定契约("Agents may rely on these routes and fields"),带同源+回环校验、403 防 CSRF。

**007-architecture(架构)** — 开发者索引
- **单写者纪律**:Engine 是 Trigger/Thread/Decision/Ticket/Assignment 的唯一写者,所有方法同步、一命令一事务;端口(Repositories/Policy/SourceScope/Assessor/Executor/Outbox)在事务内 enqueue,提交后适配器才执行;Engine 内部五个单向依赖的 Part(Intake/Evals/Tickets/Assignments/Replies);调用方只拿到按角色的 `Pick`(EngineForSources/MainSession/People/Settings/Executor)——**视察权的只读性是类型层面的,不是约定**。
- **共享谓词保证规则一致**:`waitsForPerson`/`followupOwners`/`settled`/`ticketStage`,工单结束时的未读消息回收全部由谓词统一判定。
- **Effects 持久效应队列**(关键机制):每 kind/key 一行,重复 enqueue 同 key 替换 payload 并置为到期不改变队列顺序;退避重试;`Later` 异常延迟不计失败;六种 kind(assessment/launch/delivery/relay/notice/reply)各有并发上限与重试参数(如 relay 并发 1 保序,reply 需等同工单同会话前序消息成功或三连败)。
- **只读投影的授权**:Executor Session 只能读自己工单的资源,"分页里连它读不到的对象的 id 都不出现"——授权通过投影裁剪实现,而非访问报错。
- 主 Session 14 个工具、Executor 6 个工具;worker 子代理用 `standard` preset 但只保留 worker 的工具白名单,工单结束即失权。
- **官方承认的开放问题**:哪些 Trigger 字段可信(actor 可信,正文/引用不可信)、部署隔离(独立 pinned DSH 安装)都未解决。
- **数据无限增长**:"还没有任何删除旧记录的逻辑,trigger/disposition/decision 行、已结束工单、outbox 回执、设置与 prompt 每个版本都无限增长"(docs/007-architecture.md)。

**008-cookbook(扩展手册)**
以一个虚构 webhook Source 为例给出接入 SourceKind 的完整配方,价值在抽象接口面的设计:翻译(稳定 id——重放安全,编辑作新事件要进 id)、scope(只依赖 meta,重验自动正确)、channel(幂等键发送、`gone` 仅用于永久不可达)、threadContext(评估上下文窗口)、policy part(allowed+admission 行,进 Policy version hash)、execution variables(无凭据)、插件循环(等 setup、每轮读配置、游标在全部 Trigger 落库后才推进、逐项容错、连续 3 败才报 error、abort signal 贯穿)。测试配方与检查清单 11 条。

**SETUP.md(附)**
面向"由 agent 替操作者安装"的流程:一次只问一个问题、每条 <300 词、带推荐答案、先收集全部答案再执行、永不索要秘密(操作者手工步骤单独发)。体现了 000 的注意力哲学如何落到流程。

## B. 与 22 条决策的三桶对照

### 「可直接抄」——已实现且兼容

| 决策 | superagent 对应物 | 可抄的具体机制 |
|---|---|---|
| 1 任务=持久容器 | Ticket(001/003) | 挂对话(Thread 订阅+Inputs)、状态账本(Report/Assignment/stage 表)、重启存活、人的消息唤醒;补一点:它的"记忆"是 DB 结构化记录,我们的"任务记忆"可加一个蒸馏字段 |
| 4 主 agent 显式派发 | accept→Ticket→Assignment | 派发单位=工单+独立目录;prepare 命令 `git worktree` 出 checkout(003/006) |
| 12 视察权 | `orchestrator_read` 资源分页 + `request_status` + `GET state` | 只读性靠角色化 `Pick` 接口在类型层保证;蒸馏入记忆可用 `orchestrator_ticket_note`(≤100 词、只授事实不授权) |
| 13 罢免权 | Mandate 三操作原则 | accept 创建 / **人的** stop 撤销 / 重验失败吊销;Executor 无权宣称人令停止(002) |
| 16 按需激活 | 整条 Trigger→Eval→Ticket 链 | 无任务无进程;闲置工单 idle 等 Trigger;"主动找活"先过 observe 模式只记录不执行 |
| 21 横向通信需审批 | 已经是这么做的 | worker 间**没有**通信工具(007 明确排除"给其他 Session 发消息/开会话");协调只经订阅+主 Session note(不授权) |
| 22 之写手存量 | WorkerSlots + maxWorkers | 限并发+排队+人槽优先(003) |
| 其他:任务驱动触发 | 即其整个 Trigger/Eval 模型 | — |
| 其他:滤网=规则引擎 | gate 确定性层 + admission table | 规则引擎放 gate,模型放评估,两层分离 |
| 其他:专家团限频 | automated reviewer 规则 | 每 PR 每 reviewer 最多 3 轮、有回答即 resolve(004)——现成的限频先例 |

### 「要改造」——有雏形,按设计改

- **决策 3 分级审批链**:superagent 只有两级(确定性 gate + 主 Session 评估)+ 人。改法:把 `EngineForMainSession.decide` 的可代批范围参数化(上级 Mandate scope 内=代批,越界=自动生成 Attention 结构化提问,`human_action` 机制现成),三级链=把"上级"从主 Session 泛化为任意持 Mandate 的管理层 agent(docs/002-authorization.md、docs/004-prompts.md)。
- **决策 5 写手无执行权**:方向相反——worker preset 带 shell/job/web 且 danger-full-access。改法:自定义 preset,WORK_TOOLS 白名单砍到 read/write/orchestrator_*;注意 006 自己承认 wrapper"不是安全边界",所以必须 preset 白名单+专用 OS 账户双保险,这正是本设计优于原型的点。
- **决策 8 重要性地图**:现有最接近的是 <200 行 auto-merge 阈值与 scope 级 admission table。改法:把 admission table 思路移植成"文件路径模式→重要性等级→所需审批档"的映射表,放进 Policy version 参与 hash,让边界审批有版本可重验。
- **决策 9 云端独立验收**:无内置审查 agent,但 follow-up/自动审查者的接入面现成。改法:审查 agent 实现为链外 bot 或独立 Source,验收结论作为 Trigger 进 Eval,不进执行链授权;限频规则直接搬 3 轮上限。
- **决策 11/19 git 同步链**:保留 worktree-per-task 与"任务边界即分支",但 PR 流程强绑 github.com(明确不支持 GHE)。改法:本地微型 git 服务替换 GitHub Source——008 的 SourceKind 抽象(channel.send、bindThread/threadClosed 决定工单完成)正好是插入位,PR 的"Thread 关闭=工单结束"语义映射到"本地分支合并"。
- **决策 20 MD 通知**:superagent 通知是 Outbox 结构化消息(回执+幂等键+保序)。改法:保留 Outbox 的持久化/顺序/幂等语义,把 Channel 实现换成 MD 文件写入器(008 的 channel.send 接口即抽象位);另注意其团队配置/技能全是 MD 文件加载,可参照其组织方式。
- **决策 22 之三路合并**:无 base_commit 概念,靠 PR diff。改法:任务卡加 base_commit 字段,prepare 时记录 worktree 基点,同步点做 three-way merge;worker slot 语义保留。
- **决策 15 诚实宪法**:prompt 层已有(诚实交付、披露行、"draft is marked unfinished")但无强制。改法:把"隐瞒"定义为规则引擎可检测的事实(如 Report=ready 但验收未过),在 gate/eval 层记 Reason 落库,作为决策 13 罢免的证据链输入——正好复用 `Reason(code,values)` 体系。

### 「它没有」——设计独有,需自建

- **决策 2 动态边界对象**(申请扩张→审批→持久并入→可收缩):Mandate scope 在 accept 时固定,"wider scope means a new Ticket"(docs/001-domain.md)——原型明确拒绝动态扩张。需要全新建边界对象与审批流。
- **决策 6 统一测试 agent 群**:Executor 自实现自验证("implements and verifies"),无独立验收执行层。
- **决策 7 二级整合层 / 决策 10 三级化**:只有主 Session+worker 两级;但它的纪律值得保留——"主 Session 只路由:转发原始 Trigger 与路由事实,**不做技术决策**"(docs/004-prompts.md),这直接支持"主 agent 只管战略"的定位,中层可在 worker 之上插入整合角色。
- **决策 14/18 专家评审团与对抗性找茬**:完全没有;最近的替代品是外部 automated reviewer bot。
- **决策 17 契约登记**:policy version hash + revalidation 是"配置契约"的先例,但没有跨 agent 接口契约登记表。

## C. 官方承认的负面经验

1. **danger-full-access 是最大安全债**:两个 agent preset 都以操作者 OS 用户全权限跑,官方明说 wrapper"建立默认,不是安全边界",补救建议只有"专用 OS 账户"(docs/006-deployment.md);007 开放问题里部署隔离仍未解决。本设计的决策 5/6(收执行权)是对它最直接的修正。
2. **数据无限增长**:trigger/disposition/decision、已结束工单、outbox 回执、每个设置与 prompt 版本"还没有任何删除逻辑,无限增长"(docs/007-architecture.md)。
3. **Prompt 级规则≠强制**:auto-merge 条款"约束 agent 而非 GitHub 的合并规则";"避免读代码是 prompt 规则而非工具限制"。教训:要紧的规则必须下沉到代码强制层(决策 15 应据此设计)。
4. **Token 开销被当成产品约束管理**:存在按角色的 prompt 表面积预算测试(`model-surface.spec.ts` "within a size budget per role");大量硬性预算数字:路由 note ≤30 词、聊天 ≤100 词、评估上下文=4000 字符 Trigger+前 5 后 5 条消息(邻居 1200 字符)+8 个相关工单、存根 50 字符、列表 20 条/页 4000–8000 字符——多层级系统逐层放大信息,这些预算值得逐层复制。
5. **升级回滚单向**:新版迁移过的 SQLite 旧版打不开,回滚只靠升级前备份;Docker 式快照应成为部署默认。
6. **接入面限制**:只支持 github.com;编辑 Issue 正文加 @ 不会触发(必须新评论);Feishu 个人配置与 bot 配置共享 keychain 密钥、误跑 `config remove` 会伤及个人配置。
7. **信任边界未定**:007 开放问题承认"哪些 Trigger 字段可信"没有答案(当前实践:actor 可信、正文/引用不可信)。

## D. 对新项目立项的 5 条最重要启示

1. **授权做成可计算对象,不做 prompt 承诺**。Mandate(版本+scope+强度+matched rules+via)+ 恢复前重验 + 只有人的 stop 可撤销,全部落库可审计。决策 3/8/13 应直接建在这个模型上:审批=生成或扩张 Mandate,罢免=撤销 Mandate,外部文本永远是数据不是授权(docs/002-authorization.md)。
2. **判定必须两层分离 + observe 灰度开关**。确定性 gate 管"谁有资格"(fast path),模型评估管"该不该做"(slow path),外加从卡片直接忽略的 `ignoredFromCard` 捷径;observe 模式把"会接受"只记录不执行,跑一段时间再切 auto。滤网规则引擎与分级审批链应原样套用,observe 是最重要的上线安全阀。
3. **"先存储后执行"的持久效应队列是一切可靠性的地基**。Outbox+Effects:每 kind/key 一行、指数退避、`Later` 不计失败、同类保序、幂等键、health 按 kind 上报。"任务=持久容器+状态账本"(决策 1/22)应以此为内核,崩溃恢复、顺序、去重全部免费获得(docs/007-architecture.md)。
4. **立项时先写"拒绝的概念"表**。001 用一张 Rejected concepts 表拒绝了约 12 个看似自然的实体(人、成员、跨源身份映射、Eval 自调度……),这是它概念数能压到最小的原因。本 22 条决策引入至少 6 个新概念(边界对象、重要性地图、专家团、契约登记、三路合并卡、MD 通知)——每个都要先回答"为什么不能是已有概念的属性",否则概念膨胀会吃掉实现预算。
5. **每一层都要有字数预算与"只读自己"的投影**。Executor 的资源分页连读不到的对象 id 都不出现(授权即裁剪);主 Session 被 `noWait` 禁止等待;各角色消息有 30/100/600/1500 等硬预算。多层级组织里信息逐级放大,没有逐层预算一定会爆 token、拖垮审批链(docs/004-prompts.md、docs/007-architecture.md)。

**一句话总结**:dsh-superagent 验证了本设计中最难的三件事——任务容器(决策 1/16)、审批与罢免的可计算化(3/13)、任务驱动无进程(16)——已被官方原型跑通且细节成熟;它没有的(动态边界、中层组织、专家团、契约登记)正是本项目的增量,而它自己承认的 danger-full-access 安全债,恰是"写手无执行权+统一测试群"决策的最佳立项论据。
