# DSH 记忆插件关键路径核查

访问与核查日期：2026 年 10 月 29 日，约 05:00（Asia/Shanghai）。本专题追踪 dsh-fish-memory 的写入、注入、更新、跨会话和删除路径，目的是明确适用边界，而非给项目打总分。

版本固定为 `Yuuyuko-uu/dsh-fish-memory@8b3571da94d8bafd12f746ad4b2035186f5c014c`，提交时间 2026-10-28T01:59:21Z。本轮通过 GitHub 连接器取得 `lib/index.js`、`lib/store-merge.mjs`、`tests/store-merge.test.mjs`、`tests/purge-generation.test.mjs`。重点阅读下述控制流程与测试断言，并检索主文件中的调用位置；未审计全部宿主或其他插件，未安装、执行项目或运行测试。下文“会进入某分支”是给定输入条件下的静态判断，不是发生频率或实际聊天效果的测量。

## 1. 常驻事实仍受召回入口门槛影响

主文件中 `recall()` 除定义外只有一处调用，位于 `systemPrompt.context` 的 memory 注入回调。该回调获取当前输入，调用 `recall()`，然后把 `selected` 交给 `render()`；空列表渲染为空字符串。另一处 `memory-hint` 提供写入指引和称呼，不读取保存的常驻事实。[注入回调](https://github.com/Yuuyuko-uu/dsh-fish-memory/blob/8b3571da94d8bafd12f746ad4b2035186f5c014c/lib/index.js#L3769)、[常驻提示内容](https://github.com/Yuuyuko-uu/dsh-fish-memory/blob/8b3571da94d8bafd12f746ad4b2035186f5c014c/lib/index.js#L3728)、[渲染](https://github.com/Yuuyuko-uu/dsh-fish-memory/blob/8b3571da94d8bafd12f746ad4b2035186f5c014c/lib/index.js#L3002)

信息量门槛默认 0.8。用于统计说话习惯的样本少于 20 时返回无穷大，暂不拦截；达到该条件后，低于门槛会直接返回空列表，早于常驻事实的选择。因此，“每轮注入长期事实”在所读插件内存在明确条件：低信息输入被门槛拦截时，这条路径不会补入它们。[信息量计算](https://github.com/Yuuyuko-uu/dsh-fish-memory/blob/8b3571da94d8bafd12f746ad4b2035186f5c014c/lib/index.js#L2420)、[提前返回](https://github.com/Yuuyuko-uu/dsh-fish-memory/blob/8b3571da94d8bafd12f746ad4b2035186f5c014c/lib/index.js#L2773)、[后续常驻事实选择](https://github.com/Yuuyuko-uu/dsh-fish-memory/blob/8b3571da94d8bafd12f746ad4b2035186f5c014c/lib/index.js#L2908)

**解决的问题与代价：**门槛可以避免“嗯”“哈哈”等输入也翻出一堆无关往事，并节约评分开销；把基本事实也放在门后，会牺牲这些轮次的重新注入。不能由此推断角色必然忘记名字：历史上下文、人物设定或其他插件可能仍提供信息。后续应分别检验“是否注入”和“最终回答是否知道”，不要混为一个指标。

## 2. 重要度和长期有效性在写入时被绑定

`remember` 的参数把重要度从随口一说到非常重要分级；写入记录时，`importance >= 1` 直接设为 `stable_profile`，其余为 `event`。该工具没有独立的类别参数。稳定类别在鲜活度计算中不随时间衰减。[工具参数](https://github.com/Yuuyuko-uu/dsh-fish-memory/blob/8b3571da94d8bafd12f746ad4b2035186f5c014c/lib/index.js#L3038)、[类别赋值](https://github.com/Yuuyuko-uu/dsh-fish-memory/blob/8b3571da94d8bafd12f746ad4b2035186f5c014c/lib/index.js#L3236)、[鲜活度](https://github.com/Yuuyuko-uu/dsh-fish-memory/blob/8b3571da94d8bafd12f746ad4b2035186f5c014c/lib/index.js#L2466)

**适合与边界：**用一个数表达“特别值得记住”很容易接入，但特别重要的临时事件也可能被当成稳定事实。例如近期住院可能极其重要，却不应永久表示当前健康状态。项目其他编辑或导入入口可以有不同处理；这里只确认模型工具的赋值规则。值得研究的是把重要程度、记录类型、有效期限分开，而非简单调低重要度。

## 3. 系统记录时间不能自动代替事件发生时间

`remember` 没有事件时间参数，新记录的 `at` 与 `createdAt` 都取调用时的系统时间。注入时却把 `at` 转成相对时间前缀，并指引模型按该时间叙述往事。[写入时间](https://github.com/Yuuyuko-uu/dsh-fish-memory/blob/8b3571da94d8bafd12f746ad4b2035186f5c014c/lib/index.js#L3236)、[时间渲染与指引](https://github.com/Yuuyuko-uu/dsh-fish-memory/blob/8b3571da94d8bafd12f746ad4b2035186f5c014c/lib/index.js#L3002)

由此构成一个待测反例：今天补记“去年暑假一起去过海边”，正文可能保留去年，但结构化前缀指向今天。避免模型随手编造日期是合理目标，代价是该路径没有表达“现在得知过去的事”。手动导入等路径另外存在时间识别，不能把这一结论推广成所有记录都没有事件时间。

同事实键收敛优先按 `at`、其次按 `createdAt` 排新旧，并将较旧条目标为被取代；时间相同或无法确定的记录有保护，不随意覆盖。因此，如果模型给今天回忆的旧住址和当前住址同一个事实键，旧事被赋予更晚记录时间后，可能错误取代当前状态。这是需要特定输入和键分配的条件性反例，尚未观察实际模型触发率。[收敛规则](https://github.com/Yuuyuko-uu/dsh-fish-memory/blob/8b3571da94d8bafd12f746ad4b2035186f5c014c/lib/index.js#L2695)

**研究启发：**至少区分发生时间、得知时间和事实有效区间。对“当时住哪里”和“现在住哪里”采用不同查询语义，才能保留生活变化。

## 4. 配额检查先于纠正，子串去重也可能先于纠正

每日记录配额默认 8 条。`remember` 先计算当天创建的记录数，达到配额直接返回失败；之后才进入去重、显式取代和事实键处理。该入口没有重要纠正的配额豁免。它会明确报告失败，不能描述成静默成功，但用户当天恰好需要纠正旧事实时仍可能受阻。[配额与后续顺序](https://github.com/Yuuyuko-uu/dsh-fish-memory/blob/8b3571da94d8bafd12f746ad4b2035186f5c014c/lib/index.js#L3137)

去重去掉空白和部分标点后，若文本相等，或双方长度大于 8 且一方包含另一方，就强化原条目并提前返回；这发生在处理 `取代` 和 `事实键` 之前。例如以下两个合成文本满足子串条件：

- 旧：“她每天晚上都喜欢喝一杯咖啡”
- 新：“她每天晚上都喜欢喝一杯咖啡但最近已经戒了”

按该控制流程，新文本会走强化旧条目的分支，而非写入后半句的状态变化，即使本次还携带取代字段。实际模型可能选择完全不同的表述，所以此处证明的是可构造边界，不是日常普遍失败。[去重和提前返回](https://github.com/Yuuyuko-uu/dsh-fish-memory/blob/8b3571da94d8bafd12f746ad4b2035186f5c014c/lib/index.js#L3144)、[后续取代处理](https://github.com/Yuuyuko-uu/dsh-fish-memory/blob/8b3571da94d8bafd12f746ad4b2035186f5c014c/lib/index.js#L3181)

**解决的问题与代价：**配额限制噪声增长，去重避免重复占空间；但新增、重复、补充和纠错应有不同规则。积极的一点是，单纯召回不会自动强化记录，代码将强化放在重复写入路径，避免仅因系统自己频繁想起而不断提高重要度。仍需检查模型发起写入的依据是否真的来自用户。

## 5. 跨会话复制是快照，不等于持续同步的身份记忆

存储按会话 ID 分文件。分支继承检查父会话、排除子代理，并仅在目标没有记忆时复制；复制后重置召回轨迹。显式复制入口保留目标已有记录，按去除首尾空白后的文本去重，必要时重分配 ID。[分文件路径](https://github.com/Yuuyuko-uu/dsh-fish-memory/blob/8b3571da94d8bafd12f746ad4b2035186f5c014c/lib/index.js#L225)、[分支继承](https://github.com/Yuuyuko-uu/dsh-fish-memory/blob/8b3571da94d8bafd12f746ad4b2035186f5c014c/lib/index.js#L3876)、[显式复制](https://github.com/Yuuyuko-uu/dsh-fish-memory/blob/8b3571da94d8bafd12f746ad4b2035186f5c014c/lib/index.js#L6075)

这能帮助新分支延续已有经历，但所读实现不是持续共享一个人的统一记录。来源后来纠正或删除，不能推定所有副本也同步变更。显式复制路径本身未调用事实键收敛；两个不同文本若表达相冲突的同键事实，可能同时进入目标活动记录，直到其他路径处理冲突。不能把“可跨会话复制”直接等同于“跨设备、群聊、身份一致”。

## 6. 防止删除复活已有实质实现，但边界仍需保留

合并函数以记录 ID 合并，删除标记优先；旧快照即使带有更晚的扩词时间，也不能自动撤销删除。只有明确恢复入口列出的 ID，且写入内容确有该记录，才撤销对应删除标记。保存流程还检查清理代际，旧任务在库已清理后写回会被拒绝。源码明确提示这不构成跨进程事务。[合并函数](https://github.com/Yuuyuko-uu/dsh-fish-memory/blob/8b3571da94d8bafd12f746ad4b2035186f5c014c/lib/store-merge.mjs)、[保存与代际检查](https://github.com/Yuuyuko-uu/dsh-fish-memory/blob/8b3571da94d8bafd12f746ad4b2035186f5c014c/lib/index.js#L398)

本轮读到的合并测试包含：旧快照不复活删除、只恢复指定记录、保留无关删除标记、保留较新修改、新增记录与合并幂等性。清理代际测试使用模拟宿主与模型响应，安排异步任务跨越删除/清理，再检查旧记录未复活。这比 README 口头承诺提供了更明确的工程证据；**本轮没有运行这些测试，不能报告测试通过，更不能保证所有并发情况安全。**[合并测试](https://github.com/Yuuyuko-uu/dsh-fish-memory/blob/8b3571da94d8bafd12f746ad4b2035186f5c014c/tests/store-merge.test.mjs)、[清理代际测试](https://github.com/Yuuyuko-uu/dsh-fish-memory/blob/8b3571da94d8bafd12f746ad4b2035186f5c014c/tests/purge-generation.test.mjs)

保存逻辑仍保留普通、每日及销毁前等备份；销毁前备份有数量限制。这有恢复价值，但主库不再使用某条记忆，不等于磁盘所有副本都消失。跨会话已复制的副本也需要单独定义删除范围。[备份代码](https://github.com/Yuuyuko-uu/dsh-fish-memory/blob/8b3571da94d8bafd12f746ad4b2035186f5c014c/lib/index.js#L330)

## 后续验证设计：仅设计，未执行

| 合成场景 | 需要控制的条件 | 应分别观察什么 |
|---|---|---|
| “在吗”时是否保留基本约定 | 说话统计样本达到 20；已有稳定事实；门槛开启/关闭 | 注入内容、已有对话上下文、最终回答，三者分开 |
| 重要的临时住院 | 相同事实分别赋重要度 0.9 与 1 | 类别变化、后续出院更新、是否继续描述为当前住院 |
| 今天补记去年旅行 | 正文有历史时间，工具没有事件时间 | 结构字段、相对时间前缀、最终叙述冲突 |
| 补述旧住址 | 当前住址已记录；同事实键；之后回忆旧址 | 哪条被取代；询问当时与现在是否分别正确 |
| 第 9 次记录是纠正 | 当天配额已用完；显式取代字段 | 工具是否拒绝、旧事实是否保留、模型是否告诉用户未更新 |
| 长文本末尾追加否定 | 新旧文本满足子串条件 | 是强化旧条目，还是保存否定与修订关系 |
| 两个会话分别变化 | 先复制，再在来源纠正/删除 | 两库状态、冲突事实、删除是否跨副本传播 |
| 后台整理跨越清理 | 合成数据，延迟返回的旧任务 | 活动库、删除标记、备份、错误反馈分别核查 |

这组反例帮助区分“中文词法召回是否够用”和“事实生命周期是否可靠”。提高检索相似度不会自动修复时间混淆、错误去重或副本更新。对单会话、近期经历、可解释检索的需求，该插件有直接价值；对多年状态变迁与跨身份同步的需求，还需要另外核查这些边界。
