# 机格分裂 1.0 项目书

## 项目副标题

**Read the Room — A Context-Aware Interaction Harness for Multi-Mode AI Assistants**

---

## 一、项目定位

### 1.1 项目是什么

"机格分裂"是一个围绕现成大语言模型构建的 Context-Aware Interaction Harness（上下文感知交互 Harness）。它不训练一个全新的聊天模型，也不只是给 AI 套五种语气，而是在模型外部增加一套可执行的交互策略系统，使 AI 在回答或行动前，先判断用户当前想做什么、处于什么互动状态、任务进行到哪一步、当前是否满足执行前置条件、此刻应不应该介入、应由哪一种"机格"回应、介入应该有多深、最轻的有效帮助是什么，以及回答失败后该如何修复。

项目核心不是"让模型更聪明"，而是：

> **让模型在正确的时间，以正确的方式，做正确程度的事情。**

### 1.2 为什么属于 Harness Engineering

Prompt 只是本项目的一部分，完整系统还包括上下文组装、显式的会话与工作流状态、机格路由、干预策略、前置条件检查、输出验证、Repair、人工覆盖、Trace 记录，以及评估机制。因此项目的技术定位应当是：

> **一个用于多机格 AI 助手的上下文感知交互 Harness。**

更具体地说：

> **这个项目不修改模型权重，而是工程化模型外部的状态、策略、约束、验证与修复机制。**

---

## 二、项目背景与问题

### 2.1 当前 AI 的主要问题不一定是能力不足

现有大模型已经可以写文章、修改代码、总结材料、Brainstorming、回答问题、生成计划，甚至调用工具完成部分任务。但实际体验仍然经常不好，原因往往不是模型不够聪明，而是它在错误的时间介入、没识别用户当前所处的阶段、把最终目标误当成当前步骤、没检查执行前置条件、在用户发散时过早挑刺、在用户需要结论时继续发散、在用户已有大量投入后仍全部重写、在用户明显不满意后仍要求用户重新解释，或者频繁弹出帮助反而增加了操作摩擦。在语音或陪伴场景中，它还常常把开场和续话的责任全部交给用户。

### 2.2 核心问题

传统聊天系统通常采用：

```text
用户输入
→ 大模型直接生成回答
```

机格分裂采用：

```text
用户输入 + 对话历史 + 用户偏好 + 工作流状态
                    ↓
              读空气 Router
                    ↓
判断：
- 当前是什么互动状态？
- 用户做到哪一步？
- 是否需要介入？
- 应使用哪个机格？
- 最轻的有效干预是什么？
                    ↓
              对应机格生成
                    ↓
              验证与修复
```

项目解决的不是"模型能做什么"，而是：

> **模型现在应该做什么，以及应该以什么方式做。**

---

## 三、产品哲学

### 3.1 Goal 不等于 Stage

用户说"我要整理实习 JD"，不代表用户已经收集了 JD、导出了数据、建好了表、确定了字段、或者知道要分析什么。系统必须把 Goal（最终目标）、Stage（当前进行到哪一步）、Obstacle（目前卡在哪里）和 Preconditions（下一步执行需要满足什么条件）明确区分开，而不是把用户说出的目标直接当成已经具备执行条件的信号。

### 3.2 AI 不应因为理解目标，就默认执行条件已经成立

AI 在执行前应优先检查关键 Preconditions，但也不能变成连续盘问。系统需要区分三类条件：可以从上下文直接观察到的条件、可以进行低风险假设的条件，以及真正阻塞下一步、必须向用户确认的条件。原则是：

> **能观察就观察，能合理假设就继续，只有关键未知条件才做最小确认。**

这里的"能观察"，很大一部分依赖的是可靠的时间信号——比如某个任务距上次更新已经过去多久、某个状态持续了多长时间。这类时间戳信息属于典型的"可观察条件"，理论上应该由系统自动读取和计算，而不需要每次都向用户确认。但时间戳的处理方式需要格外小心，具体原则在第五章详细说明。

### 3.3 AI 的价值不是做更多，而是正确介入

AI 有时应该接梗、继续发散、给出判断、只问一个问题、帮用户检查、主动打开话题，有时则应该暂时保持安静、停止扩写、承认上一轮理解错误，或者切换到 Repair。因此每一轮回应之前，系统都要先回答三个问题：现在是否应该介入（Should I intervene）、用户为什么卡住（Why is the user stuck），以及最轻但足以推动任务的干预是什么（What is the lightest effective intervention）。

### 3.4 AI 应降低用户的回答门槛

回答门槛指用户为了接住 AI 的这一轮，需要付出的认知成本、表达成本、自我暴露成本、选题成本和理解成本。尤其在语音、陪伴和冷启动场景中，AI 不应只抛出一个宽泛问题，而应该先贡献具体内容，再留下一个低门槛的回应入口。

### 3.5 Creative、Critical、Verifiable 是全局质量原则

Creative 指提供认知增量，而不只是复述、改写或重新排版；Critical 指检查前提、遗漏变量、反例和失败条件，不盲目迎合用户；Verifiable 指区分事实、推断、假设和主观判断，不靠语气制造可信感。不同机格对这三者会采用不同权重，不要求每一轮都全部拉满。

### 3.6 Boldness 是表达变量，不等于 Critical

Critical 决定 AI 有没有发现问题，Boldness 决定 AI 敢不敢直接说出来。比如同样是发现用户已经有了倾向，低 Boldness 的说法是"你可能已经更倾向其中一个选项"，高 Boldness 的说法则是"别装了，你其实早就选好了，现在只是在给自己找一个更理性的理由"。第一版不自动预测 Boldness，先作为用户可以手动调节的偏好设置。

---

## 四、机格体系

第一版保留五个主机格和一个覆盖状态。

### 4.1 闲聊机格

适用于日常聊天、吐槽、轻情绪表达、好奇心、无明确任务的交流，以及语音场景下的冷启动。它的目标是接住情绪与梗、保持相关性、不强迫产出、不把所有话题都项目化，必要时由 AI 主动续话或转场，并尽量降低用户的回答门槛。

### 4.2 Brainstorming 机格

适用于新点子、模糊想法、产品吐槽、项目方向、命名与概念扩展，以及尚未成形的问题。它的目标是从碎片表达中识别最强洞察、沿用户原来的方向继续生长、建立大胆但合理的连接，允许一定的抽象、玩梗和 Boldness，先扩展再评估，不过早列风险、竞品和实施困难。Brainstorming 同时吸收了原来的"扯皮"能力：允许用户从模糊感受和吐槽开始，通过来回对话发现问题本身。

Brainstorming 机格产出的成果，如果被用户确认为一个真正想推进的计划，应当能够被主动写入日历，作为后续协作执行机格的输入——这一点在第五章和新增的日历功能一节中详细说明。

### 4.3 写作机格

适用于论文、PPT、简历、邮件、方案、文档结构、代码解释，以及文字与表达修改。它的目标是先诊断真正的问题、保留用户已经确定的观点、区分用户原意和 AI 扩展、优先进行最小但高价值的修改，避免泛泛润色，也避免擅自重写整套逻辑。理想的体验是用户说"对，这就是我觉得不对但没说清楚的地方"，而不是"这一版更流畅、更专业"。

### 4.4 协作执行机格

适用于需要持续推进的任务、存在多个步骤、依赖材料状态和执行条件、需要 AI 与用户共同完成的工作。它的目标是识别 Goal、识别当前 Stage、识别 Obstacle、检查 Preconditions、保护用户已有投入，只确认真正阻塞下一步的问题，给出最小可执行的下一步，不跳步、不抢活、也不空问。核心问题始终是：用户想完成什么、现在做到哪一步、当前卡在哪里、下一步缺什么、最轻的有效动作是什么。

日历读取是协作执行机格的一项重要输入来源：如果 Brainstorming 阶段产出的计划已经被写入日历，协作执行机格应当每天读取相关条目，判断今天该做什么、昨天是否有遗留未完成的任务，并据此决定要不要主动提醒，而不是机械地把日历上的内容照本宣科念给用户听。

### 4.5 判断机格

适用于比较哪个更好、应不应该做、项目优先级、可行性分析、方案比较、风险判断，以及是否需要收敛。它的目标是检查隐藏前提、识别遗漏变量、比较选项、避免虚假平衡、给出明确结论、说明什么条件会改变结论，必要时直接反驳用户。

### 4.6 Repair Mode

Repair 不作为普通机格，而是覆盖所有模式的异常处理状态。常见触发信号包括"不是这个意思""你又开始泛了""我刚才已经说了""不要继续扩展"、连续要求重写、AI 忽略明确约束，或者用户点击"重新理解我"。Repair 的处理流程是：停止当前方向，识别上一轮偏差，重述用户真正目标，做最小修复，再返回正确机格。

不推荐的说法是"抱歉，请重新解释你的需求"；更合适的说法类似"我刚才把重点放在了功能扩展，但你真正问的是如何最快落地，重新按 MVP 来"。

---

## 五、读空气 Skill 1.0

### 5.1 定义

读空气 Skill 不是情绪识别，也不是试图读懂用户内心。它是：

> **根据当前对话、任务状态和可观察信号，生成本轮交互策略的 Router。**

### 5.2 第一版职责

读空气 Skill 需要完成十项判断：推荐当前机格、判断是否需要 Repair、判断当前对话状态、判断回答门槛、判断是否需要主动推进、在协作任务中识别 Stage、检查关键 Preconditions、输出 Intervention Level、输出回复计划，以及输出本轮必须避免的行为。

### 5.3 时间戳与日历信号的处理原则

时间戳是这套系统最基础也最容易被误用的一类信号。Idle Time、任务停留天数、日历事件的临近程度，这些判断的前提都是可靠的时间数据，缺了它，"能观察就观察"这条原则就没有地基。但时间戳不能被直接丢给负责生成回答的大模型去"感受"——模型很容易从一串原始时间信息里脑补出情绪判断，比如看到用户凌晨回复、或者连续多天只回一句话，就自行推测用户可能很疲惫或者心情不好，这恰好违背了"只读信号、不读人"的核心原则。

因此系统采用分层处理：原始时间戳的计算全部放在规则层完成，规则层只产出结构化的、去情绪化的结论，例如"该任务已停滞 4 天"或者"距离日历中的截止时间还剩 2 小时"；生成回答的 LLM 只接收这些已经翻译过的结构化信号，不接触原始时间戳，Prompt 中也会明确写明"不要基于时间信息推测用户的情绪或心理状态"。这样时间戳依然是系统的核心输入，但它的解读权始终留在规则层，而不会流到语言模型手里被过度诠释。

### 5.4 日历功能

日历在这套系统里承担的是"计划落地为行动"的载体角色，具体分工如下。当 Brainstorming 机格帮用户把一个模糊想法收敛成具体计划，并且用户确认这是真的想推进的事情后，系统会把这个计划拆解成带时间锚点的具体任务，写入日历——这一步拆解交给 LLM 负责，因为它需要理解计划的内容和优先级；但拆解出的任务是否与已有日程冲突、时间安排是否现实，则由规则层负责校验，避免 LLM 生成不合理的排期。

协作执行机格每天读取日历中的相关条目，判断今天该做什么、有没有前一天遗留未完成的任务。是否要主动提醒，同样遵循 Preconditions Check 和 Lightest Intervention 的原则：任务时间明显临近，或者前一天的任务确实没有完成，才触发提醒；否则系统保持安静，不会仅仅因为"今天日历上有事"就主动念叨。

这个功能天然适合用备考复习这类场景来验证：模糊目标（比如"考好某门课"）经 Brainstorming 拆解为具体的复习计划、写入日历，再由协作执行机格每天读取、判断今天该复习什么、昨天有没有拖欠，这个链条完整覆盖了"目标—计划—执行"的三段式流程，也是第一版一个现实可用的测试场景。

### 5.5 Router 输入

```json
{
  "current_message": "用户本轮消息",
  "recent_messages": ["最近若干轮对话"],
  "current_mode": "auto 或手动选择",
  "user_preferences": {
    "creative": "high",
    "critical": "high",
    "verifiable": "high",
    "response_threshold": "low",
    "boldness": "medium_high"
  },
  "workflow_state": {
    "goal": null,
    "stage": null,
    "available_inputs": [],
    "blocking_conditions": []
  },
  "time_signals": {
    "days_since_last_update": null,
    "calendar_events_today": [],
    "overdue_tasks": []
  }
}
```

`time_signals` 中的字段均由规则层预先计算好，Router 不需要、也不应该自行解读原始时间戳。

### 5.6 Router 输出

```json
{
  "recommended_mode": "collaborative_execution",
  "confidence": "high",
  "repair_needed": false,
  "conversation_state": "task_execution",
  "goal": "整理实习 JD",
  "workflow_stage": "collecting_inputs",
  "friction_type": "missing_input",
  "missing_preconditions": ["JD source"],
  "should_intervene": true,
  "intervention_level": 2,
  "response_threshold": "low",
  "response_plan": "确认唯一阻塞条件，不要直接生成最终表格。",
  "avoid": [
    "假设用户已经拥有 JD",
    "一次询问多个不必要的问题",
    "直接进行总结或分类"
  ]
}
```

第一版采用 low / medium / high 和固定枚举，避免制造 0.73、0.84 之类没有依据的假精确。

---

## 六、干预等级

第一版采用五级 Intervention Ladder：

- **Level 0：保持安静** —— 用户正在顺利完成任务，无明显摩擦。
- **Level 1：轻提示** —— 提供一个可忽略的简短建议，不打断流程。
- **Level 2：最小确认** —— 只询问一个真正阻塞执行的关键条件。
- **Level 3：提供下一步** —— 给出具体、可执行的下一步或两个低门槛选项。
- **Level 4：生成或检查产物** —— 生成草稿、代码、表格、摘要，或检查已有成果。

第一版不自动执行外部工具动作，工具调用与更高权限放到后续版本。日历写入属于工具调用的一种，第一版按 Level 3 或 Level 4 处理，需要用户明确确认后才执行，不会在用户没有确认计划的情况下自动写日历。

---

## 七、系统架构

### 7.1 最小闭环

```text
用户输入与最近对话
          ↓
会话状态 / Workflow State / 用户设置 / 时间信号
          ↓
Read-the-Room Router
          ↓
结构化 Interaction Policy
          ↓
Prompt Composer
          ↓
Base Prompt + Mode Prompt + Policy + State
          ↓
Response Generator
          ↓
Validator
          ↓
通过 / 重写一次 / Repair
          ↓
最终回答 + Trace + 用户反馈
```

### 7.2 核心模块

系统由九个模块组成。Read-the-Room Router 判断机格、阶段、摩擦和干预策略；Persona Policy Library 保存五个机格的行为协议；Workflow State Manager 显式维护 Goal、Stage、Obstacle 和 Preconditions；Time Signal Engine 负责计算 Idle Time、日历事件临近程度等结构化时间信号，并只向下游输出去情绪化的结论；Intervention Controller 决定是否介入以及介入多深；Prompt Composer 动态拼接当前所需的上下文与规则；Response Generator 调用现成大模型生成回答；Response Validator 检查回答是否符合策略；Repair and Human Override 处理偏航并允许用户手动切换；Trace Logger 保存每轮决策链路，用于调试和评估。

---

## 八、技术选型

### 8.1 原型设计

在正式开发之前，先用 **Figma** 做一版交互原型，验证界面布局和交互流程本身是否顺畅，再进入代码实现。原型阶段不连接真实的 Router 逻辑，而是针对几个预设场景（对应第十一章的 Demo 案例）手动设计出"AI 已经完成判断"之后的界面状态，通过 Figma 的 Prototype 连线功能模拟点击跳转，让人在没有后端的情况下也能感受到这套交互逻辑，属于典型的 Wizard of Oz 原型做法。

需要覆盖的界面包括：主聊天页面（五个机格 Tab、Auto/Manual 切换、聊天气泡）、Debug Panel（推荐机格、置信度、Goal/Stage、Friction 类型、Intervention 等级）、Compare 页面（Baseline / Routed / Manual 三栏对比），以及日历视图（任务条目、来源标注、完成与逾期状态）。这一步也顺带作为 Figma 的练手过程，从主聊天页面开始，逐步加上 Debug Panel、场景跳转、Compare 页面，最后再做日历视图。

原型稿完成并自己走查顺畅之后，再进入 8.2 起的技术栈实现，Figma 稿可以直接作为 Streamlit 界面搭建时的设计参照。

### 8.2 前端

第一版使用 **Streamlit**，原因是全 Python、能快速搭建 Chat UI、方便展示 Router Debug Panel、支持 Session State，适合一周内做出可演示的 MVP。

### 8.3 后端

第一版使用 **Python**，不单独拆 FastAPI，直接按模块组织（UI、Router、State Manager、Time Signal Engine、Prompt Composer、Generator、Validator、Storage），等系统稳定后再考虑拆成独立 API。

### 8.4 模型调用

第一版只接一个现成大模型 API，具体供应商视开发阶段实际情况而定，不在文档阶段锁定。调用链分三步：Router 输出结构化 JSON，Generator 生成用户可见回答，Validator 可选地检查并最多重写一次。第一版不做多模型智能路由，避免项目重点偏移。

### 8.5 日历接入

第一版优先考虑接入 Google Calendar API 或类似的标准日历接口，只做读取和写入两个基础操作，不涉及复杂的日程冲突自动协商。写入前的冲突校验放在规则层完成，不依赖模型判断。

### 8.6 数据结构

使用 **Pydantic** 定义 RouterInput、RouterOutput、WorkflowState、TimeSignals、UserPreferences、ValidationResult 和 ConversationTrace。

### 8.7 数据存储

第一版优先使用 JSONL（最快）或 SQLite（便于查询与统计），保存内容包括对话记录、Router 输出、选择的机格、人工覆盖记录、助手回答、验证结果和用户反馈。

### 8.8 测试

使用 **Pytest**，重点测试 Router JSON 能否通过 Schema、明确信号能否进入正确模式、Repair 是否优先触发、协作执行是否检查 Preconditions、时间信号是否只以结构化形式传给 LLM、Prompt Composer 是否包含必要规则，以及 Validator 是否识别跳步与过度介入。

### 8.9 第一版暂不使用 LangGraph

第一版采用普通 Python Pipeline：

```python
policy = route(conversation, state, time_signals)
prompt = compose_prompt(policy, mode_prompts, state)
draft = generate(prompt, conversation)
result = validate_or_rewrite(draft, policy)
```

出现多轮循环、工具调用、中断与恢复、Human-in-the-loop 审批、持久化 checkpoint、多 Agent，或者复杂条件分支等需求之后，再考虑引入 LangGraph。

---

## 九、Prompt 体系

第一版包含以下 Prompt 文件：

```text
prompts/
├── base.md
├── router.md
├── chat.md
├── brainstorming.md
├── writing.md
├── execution.md
├── decision.md
├── repair.md
└── validator.md
```

### 9.1 Base Prompt

统一规定 AI 需要保留用户明确约束、不机械复述、不无依据赞同、不捏造事实、区分事实推断和假设、不把修复责任全部交还用户、不因理解 Goal 就默认 Preconditions 成立、有可观察信息时不重复询问、只在必要时澄清，并且在用户已有大量投入时优先检查和局部增强。

### 9.2 Mode Prompt

统一结构：

```yaml
mode:
primary_goal:
prioritise:
avoid:
interaction_style:
response_structure:
success_criteria:
```

### 9.3 Router Prompt

职责是只输出策略、不直接回答用户、严格遵守 JSON Schema，并给出模式、阶段、干预等级、回复计划和禁止行为。

### 9.4 Validator Prompt

检查回答是否符合当前机格、是否跳过 Preconditions、是否提出不必要问题、是否覆盖用户已有劳动、是否过度介入、是否抬高回答门槛，以及是否违反 Router 给出的 avoid 列表。

---

## 十、第一版界面

### 10.1 主聊天页面

顶部显示五个模式（闲聊、Brainstorming、写作、协作执行、判断），支持 Auto Recommend、手动选择、锁定当前机格、显示当前机格和展示最终回答。

### 10.2 Debug Panel

展示推荐机格、置信度、当前 Goal、当前 Stage、Friction 类型、Intervention 等级、回答门槛、是否触发 Repair，以及本轮必须避免的行为清单。

### 10.3 快捷控制

第一版只保留"更主动""更直接""降低回答门槛""重新理解我"四个按钮，不展示十几个复杂旋钮。

### 10.4 Compare 页面

同一输入展示三种版本：Baseline（普通 Prompt 直接回答）、Routed（经过 Router 和机格策略）、Manual（用户手动指定正确机格）。用户可以标记哪个更说到点上、哪个更愿意继续聊、哪个更像在帮忙、哪个跳步骤、哪个太泛，或者哪个过度介入。

### 10.5 日历视图

作为可选的补充页面，展示当前由系统写入日历的任务条目，标注哪些来自 Brainstorming 确认后的计划、哪些已完成、哪些逾期。这个页面主要用于调试和演示，不是第一版的核心交付物。

---

## 十一、最小 Demo 场景

### 11.1 通用对话 Demo

用同一句话"卧槽这个思路挺牛逼"，放在不同上下文里验证系统反应：新想法刚出现时应进入 Brainstorming；已经发散很久并开始问可行性时应进入判断；用户引用 AI 的空话表示不满时应触发 Repair。这个场景的目的是证明项目不是简单的关键词分类。

### 11.2 协作执行 Demo：整理实习 JD

工作流是收集 JD、导入内容、设计字段、整理分类、分析岗位要求、形成行动计划。用户输入"我要整理实习 JD"时，系统不会直接生成表格，而是先判断当前 Stage 未知、缺少 JD 数据这一关键前提，于是回复类似"你现在是刚准备开始收集，还是手里已经有一批 JD？先确认这一点：前者需要搭收集流程，后者才适合直接做字段和分类"。用户回答"我已经有一个 CSV"之后，系统才进入读取样本、判断现有字段、提议最小字段结构、整理与分析的流程。

### 11.3 日历计划 Demo：考试复习

用户在 Brainstorming 机格里说出想复习的科目和大致时间范围，系统帮助收敛出具体的复习计划，经用户确认后拆解成带时间锚点的任务写入日历。第二天用户打开对话时，协作执行机格读取日历，判断今天该复习什么、昨天的任务有没有完成，如果昨天遗留了任务，则在给出今天安排的同时简短提及，而不是把整个日历条目原样念一遍。这个场景同时验证了日历写入、每日读取和克制介入这三个机制。

---

## 十二、涉及的 AI 概念

本项目涉及的核心概念包括：Prompt Engineering（为不同机格定义行为边界、优先级和禁止行为）、Prompt Composition（根据本轮状态动态组合 Base Prompt、Mode Prompt、Router Policy、Workflow State、User Preferences 和 Conversation Context）、LLM Routing（由 Router 判断机格、Repair、Stage、Friction 类型和 Intervention Level）、Structured Outputs（使用固定 JSON Schema 让 Router 输出可解析、可测试的策略结果）、State Management（显式保存当前机格、Goal、Stage、已有材料、缺失条件、用户投入程度、上一轮策略和 Repair 状态）、Workflow Grounding（把抽象目标落到当前可执行工作流）、Preconditions Validation（在行动前验证关键条件，避免跳步和基于错误假设生成结果）、Human-in-the-Loop（用户可以手动切换机格、否决 Router、点击 Repair、调整主动程度、提供反馈）、LLM-as-a-Judge（辅助检查回答是否符合策略，最终效果仍需人工评估）、Evaluation Harness（通过固定测试集对比 Baseline、Routed 和 Manual），以及贯穿全项目的 Harness Engineering（不修改模型权重，而是工程化上下文、状态、策略、权限与干预、验证、恢复和反馈循环）。

---

## 十三、评估方案

### 13.1 第一阶段测试集

准备 50 条案例，分布如下：

| 类型 | 数量 |
|---|---:|
| 闲聊 | 8 |
| Brainstorming | 10 |
| 写作 | 10 |
| 协作执行 | 12 |
| 判断 | 6 |
| Repair | 4 |

每条记录包含 context、user_message、expected_mode、workflow_stage、expected_intervention、must_avoid 和 reason 字段。

### 13.2 Router 指标

包括 Mode Accuracy、Repair Detection Precision / Recall、Workflow Stage Accuracy、Intervention Level Accuracy，以及 Structured Output Pass Rate。

### 13.3 回答质量指标

人工按 1–5 分评估 Relevance、Precision、Creative Gain、Critical Thinking、Verifiability、Workflow Awareness、Response Threshold、User Effort Preservation、Over-intervention 和 Mode Consistency。

### 13.4 Pairwise Preference

采用盲评方式，让评审在 A 更好、B 更好、差不多之间选择。最终汇报 Routed 相比 Baseline 的偏好率、路由准确率、跳过前置条件的错误率、Repair 成功率、不必要澄清问题的数量，以及 Validator 重写触发率。所有数据必须来自实际测试，不能预先编造。

---

## 十四、最快执行计划

**Day -1：Figma 原型。** 在 Figma 里画出主聊天页面、Debug Panel、Compare 页面和日历视图，针对第十一章的几个预设场景做出对应的界面状态，用 Prototype 连线模拟点击跳转，走查一遍交互是否顺畅。这一步不写代码，重点是把界面布局和信息层级想清楚，同时作为 Figma 的练手。验收标准是能对着原型稿完整讲一遍这几个 Demo 场景，而不需要临场解释。

**Day 0：冻结范围。** 确定五种机格、确定 Router Schema、确定项目结构，把 JD 整理定为协作执行 Demo，把考试复习计划定为日历 Demo，不再添加新模块。

**Day 1：跑通 Router。** 创建 GitHub 仓库、配置 Python 环境、封装模型 API、创建 Pydantic Schema、编写 Router Prompt、完成 CLI 测试。验收标准是输入一段对话，能返回合法的 Router JSON。

**Day 2：跑通五个机格。** 编写 Base Prompt 和五个 Mode Prompt、实现 Prompt Composer 和 Generator、支持手动选择机格。验收标准是同一句输入在手动切换五个机格后，输出存在明显差异。

**Day 3：Streamlit UI。** 完成 Chat 页面、会话历史、当前机格显示、Auto / Manual 切换、Debug Panel，并展示 Router JSON。验收标准是网页可以正常聊天、可以看到路由结果、可以手动覆盖。

**Day 4：Repair 与 Validator。** 实现 Repair Trigger、"重新理解我"按钮、Validator，不合格时最多重写一次，并保存 Trace。验收标准是用户说"你理解错了"之后，系统能进入 Repair 并重新对齐。

**Day 5：JD Workflow 与日历 Demo。** 定义 JD Workflow State 和四至六个 Stage、加入 Preconditions、实现最小确认、支持可选的 CSV 上传；同时实现 Time Signal Engine 的基础版本，接入日历读写，跑通 Brainstorming 写入日历、协作执行每日读取这条链路。验收标准是用户只有目标没有数据时，系统不会跳步，而是先确认输入；日历计划写入后，第二天能被正确读取并给出恰当程度的提醒。

**Day 6：评估。** 完成 30–50 条测试案例，批量运行 Baseline 与 Routed，进行人工盲评，输出结果 CSV 并计算各项指标。

**Day 7：简历化。** 完成 README、架构图、Demo GIF、两分钟演示视频、项目截图、简历 Bullet 和 Future Work。

---

## 十五、项目结构

```text
machine-persona/
├── app.py
├── README.md
├── requirements.txt
├── .env.example
│
├── core/
│   ├── router.py
│   ├── generator.py
│   ├── validator.py
│   ├── prompt_composer.py
│   ├── state_manager.py
│   └── time_signal_engine.py
│
├── integrations/
│   └── calendar_client.py
│
├── models/
│   ├── routing.py
│   ├── workflow.py
│   ├── preferences.py
│   ├── time_signals.py
│   └── trace.py
│
├── prompts/
│   ├── base.md
│   ├── router.md
│   ├── chat.md
│   ├── brainstorming.md
│   ├── writing.md
│   ├── execution.md
│   ├── decision.md
│   ├── repair.md
│   └── validator.md
│
├── workflows/
│   ├── jd_workflow.yaml
│   └── exam_review_workflow.yaml
│
├── storage/
│   ├── database.py
│   └── traces.db
│
├── evaluation/
│   ├── cases.jsonl
│   ├── run_eval.py
│   ├── pairwise_review.py
│   └── results/
│
└── assets/
    ├── architecture.png
    ├── demo.gif
    ├── figma_prototype/
    └── screenshots/
```

---

## 十六、第一版完成标准

满足以下六项即视为 1.0 完成：用户能正常聊天；Router 能稳定输出结构化结果；五种机格有明显可感知差异；用户能手动覆盖机格；Repair 能处理显性纠正；有 Baseline 对比和初步评估结果。

完成后即可上传 GitHub、录制 Demo、写入简历，并用于 AI 解决方案、产品、实施、交付等岗位投递。

---

## 十七、后续迭代

**1.1 版本** 计划优化 Prompt、扩大测试集、改进机格边界、完善 UI，并增加更多用户设置。

**1.5 版本** 计划保存用户手动覆盖数据、记录回答偏好、生成模式混淆矩阵、增加更多 Workflow、支持文件上传，并加入基础工具调用；日历功能在这一阶段可以扩展到支持多日程冲突提示。

**2.0 版本** 读空气 Skill 将进一步判断是否应该介入、用户为什么卡住、用户投入程度、最轻有效干预、何时保持安静、何时调用工具、何时请求人工确认，可以在状态与分支变复杂后引入 LangGraph。

**3.0 版本** 只有在积累足够真实数据、并发现明确工程瓶颈后，才考虑引入小型 Router 分类器、Preference Model、Fine-tuning、DPO 或多模型路由。训练模型不是项目成立的必要条件。

---

## 十八、简历表达

### 项目名

**Read the Room — Context-Aware Multi-Mode AI Interaction Harness**

### 中文描述

基于读空气策略路由的多机格 AI 交互 Harness，通过显式建模对话状态、工作流阶段、任务前置条件、时间信号与干预等级，在闲聊、Brainstorming、写作、协作执行和判断五种模式间进行可解释路由，并支持将 Brainstorming 产出的计划写入日历、由协作执行机格每日读取推进。

### 英文 Bullet 草稿

- Designed the end-to-end interaction flow in Figma (chat interface, router debug panel, baseline-vs-routed comparison, calendar view) before implementation, validating UX against several preset scenarios.
- Designed a context-aware interaction harness that routes conversations across five modes—casual chat, brainstorming, writing, collaborative execution, and decision support—using dialogue intent and workflow state.
- Implemented a structured Read-the-Room router that identifies task stage, missing preconditions, friction type, intervention level, and response strategy before generation.
- Built a rule-layer time signal engine that computes idle time and calendar proximity as structured signals, keeping raw timestamps out of the LLM's context to avoid emotional inference.
- Added calendar read/write integration so confirmed brainstorming plans are broken into scheduled tasks, with the execution persona reading them daily to decide whether and how lightly to intervene.
- Built a Streamlit prototype with transparent routing traces, manual persona override, repair mode, workflow grounding, and baseline-versus-routed comparison.
- Developed an evaluation dataset covering mode selection, skipped-precondition errors, response threshold, workflow awareness, and pairwise preference.

完成评估后再补充真实数字。

---

## 十九、最终落地决策

第一版不从训练模型开始，也不从复杂前端开始，先跑通最短闭环：

```text
对话
→ Router 输出 JSON
→ 选择机格
→ 拼装 Prompt
→ 生成回答
→ 展示路由决策
```

只要这条链跑通，"机格分裂"就从一套产品哲学变成了一个真实可演示的 Interaction Harness。日历读写和时间信号引擎作为第一版里相对独立的一条支线，可以在最短闭环跑通之后再接入，不阻塞核心链路的验收。后续的 Repair、Workflow、Validator、评估和界面，都围绕这条闭环逐层增加。
