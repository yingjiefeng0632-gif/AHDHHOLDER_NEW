# 桌面副驾驶 Agent 开发框架

> 一个浮窗形态、键鼠活动驱动、零评判的 ADHD 专注副驾驶。
> 核心理念：**用户点选的子模块 = 唯一航线真相；键鼠节奏只用来决定"该不该问一句"。**

---

## 0. 三条不可妥协的设计约束

在写任何代码之前，先把这三件事钉死，它们决定整个架构形态。

1. **航线由人定，不由机器猜。** 用户点击子模块（人偶坐上去）就是声明"我现在要做这件事"。Agent 永远不去猜测用户在干嘛，只追踪用户**已声明**的航线。
2. **感知是节奏，不是内容。** 权限只读"此刻键盘/鼠标有没有动作"，不读键码、不读窗口、不读屏幕。因此——
   - 能**强力检测**：停滞 / 卡住 / 离开（一段时间没动作）。
   - **几乎无法检测**：切去干别的（手在动，但不知道在动什么）。
   - 对策：把"离任务"的判断，交给**航线锚 + 周期性轻量确认**，而不是试图从键鼠数据里推断。
3. **干预极简，一击即止。** 设定时间 → 提示音响起 → 回复后停止。不搞多层分级、不追问、不唠叨。一次提醒就够了。

---

## 1. 系统分层

```
┌─────────────────────────────────────────────┐
│  浮窗 UI 层      人偶 + 3 任务 × 子模块         │  ← 用户在这里声明航线
├─────────────────────────────────────────────┤
│  感知层          键鼠活动 + 点选事件（只读节奏） │
├─────────────────────────────────────────────┤
│  状态层          航线 + 活动节奏 + 历史(Episode) │
├─────────────────────────────────────────────┤
│  决策层          FSM 状态机 + 分级干预策略 L0–L3 │ ←→ LLM 推理层（仅干预时）
├─────────────────────────────────────────────┤
│  执行层          人偶动作 · 高亮 · 气泡 · 语音    │  → 回到 UI
└─────────────────────────────────────────────┘
          本地存储（SQLite / 本地文件，默认不出设备）
```

热循环（感知→状态→决策→执行）全部是**廉价的确定性逻辑**，每秒可跑很多次也不费钱。LLM 只在"要开口说话/要理解用户回复"的那几个瞬间才被调用。

---

## 2. 数据模型

```ts
// 一个任务（用户最多同时开 3 个）
interface Task {
  id: string;
  title: string;            // "任务 A"
  color: string;            // 用于 UI 区分
  submodules: SubModule[];  // A1, A2, A3...
  status: 'active' | 'done' | 'paused';
}

interface SubModule {
  id: string;
  title: string;            // "A2：写第一部分"
  status: 'todo' | 'active' | 'done';
  estMinutes?: number;      // 可选：预估时长
}

// 当前会话
interface Session {
  startedAt: number;
  tasks: Task[];                          // 长度 ≤ 3
  focus: { taskId: string; subId: string } | null;  // 航线 = 人偶位置
  fsm: FocusState;
}

// 决策层的运行时状态
interface FocusState {
  state: FsmState;          // 见第 4 节
  enteredAt: number;        // 进入当前状态的时间戳
  interventionLevel: 0|1|2|3;
  lastInterventionAt: number;
  backoffUntil: number;     // 在此时间前不再主动打扰
}

// 感知层产出的活动信号（只有节奏，没有内容）
interface ActivitySignal {
  lastInputAt: number;      // 最近一次键/鼠动作的时间戳
  idleSeconds: number;      // 距今静止秒数
  burstRate: number;        // 近 N 秒内的动作频次（用于识别"焦躁切换"）
}

// 历史，用于复盘 + 自适应阈值
interface Episode {
  ts: number;
  type: 'anchor' | 'stall' | 'intervene' | 'reply' | 'complete' | 'snooze';
  payload: Record<string, unknown>;
}
```

设计要点：`ActivitySignal` 里**永远不出现键码或文本**——只存时间戳和计数。这既是隐私底线，也是营销卖点。

---

## 3. 感知层（键鼠活动监测）

唯一职责：把操作系统的全局输入事件，压缩成一个不断更新的 `ActivitySignal`。

```ts
// 伪代码：监听器只更新时间戳，不记录内容
onGlobalKeyOrMouseEvent(() => {
  signal.lastInputAt = now();
});

// 每秒 tick 一次，重算派生量
setInterval(() => {
  signal.idleSeconds = (now() - signal.lastInputAt) / 1000;
  signal.burstRate = countEventsInLastWindow(10_000); // 近 10s 动作数
}, 1000);
```

实现要点：
- **Tauri（推荐）**：Rust 侧用 `rdev` 或 `device_query` 做全局钩子，只取"有事件发生"这个布尔，丢弃 keycode。
- **Electron（更快出原型）**：用 `uiohook-napi`，同样只保留时间戳。
- 两个平台都需要在 macOS 上申请"辅助功能/输入监控"权限。**在权限申请文案里直接写明"我们只检测您是否在操作，不记录任何按键内容"**——这是信任的关键一句话。

---

## 4. Agent 核心：状态机 + 决策策略

### 4.1 有限状态机（FSM）

这是整个 Agent 的心脏，纯确定性、不调 LLM。

| 状态 | 进入条件 | Agent 行为 |
|---|---|---|
| `IDLE_NO_TASK` | 没有航线（没点子模块） | 安静。人偶待机姿势。 |
| `FOCUSED_ACTIVE` | 有航线 且 `idleSeconds < 45` | L0 环境正反馈："在推进中" |
| `FOCUSED_QUIET` | 有航线 且 `45 ≤ idleSeconds < 180` | **不打扰**（可能在思考/阅读）。人偶"侧耳倾听"姿势。 |
| `STALL_SUSPECTED` | 有航线 且 `idleSeconds ≥ 180` | 触发 L1→L2 轻量确认（"还在 A2 吗？卡哪了？"） |
| `DRIFT_CHECK` | 即使在动，距上次确认 ≥ `T_drift`(默认 18min) | 轻量 check-in（活动无法证明"在做对的事"，所以周期性问一次；可关） |
| `INTERVENING` | 已发出提问，等用户回应 | 等待回复 → 分类 → 路由 |
| `REVIEW` | 子模块完成 / 会话结束 | 短复盘，更新历史 |

阈值（`45s / 180s / 18min`）全部可配置，且应在 Phase 2 做**自适应**——学习每个用户"正常静止多久"，因为有人写代码会盯屏 5 分钟不动，有人 30 秒不动就是走神了。

### 4.2 分级干预阶梯（L0–L3）

**核心规则：永远从能匹配当前状态的最低档起步；无回应才升级；用户说"我没事"就退避（backoff）。**

| 档位 | 形式 | 例子 |
|---|---|---|
| **L0 环境** | 人偶姿势 + 柔和指示，无文字 | 人偶"专注工作"姿态 + 一个不打扰的"推进中"微光 |
| **L1 微文字** | 航线锚轻微脉动 + 一句短话 | "还在写 A2？" |
| **L2 对话提问** | 一个具体诊断问题 + 快捷标签 | "卡住了？" → 标签：[开始难] [被打断] [在推进] [想休息] |
| **L3 语音** | 仅在用户开启 且 L1/L2 被忽略时 | 一句简短语音轻推 |

退避逻辑（防唠叨，ADHD 用户的头号卸载原因）：
```ts
function onUserReply(reply) {
  if (reply === '在推进' || reply === '我没事') {
    state.backoffUntil = now() + 15 * 60_000;  // 15 分钟内不再主动问
    raiseThresholdsTemporarily();
  }
  if (reply === '想休息') startBreakTimer();
  if (reply === '被打断' || reply === '开始难') {
    callLLMForHelp(reply);  // 进入 LLM 推理层，给具体帮助
  }
}
```

### 4.3 决策主循环（伪代码）

```ts
function tick() {                       // 每秒调用
  const a = perception.signal;
  const f = session.focus;

  if (!f) return setState('IDLE_NO_TASK');
  if (now() < state.backoffUntil) return; // 退避期内，静音

  if (a.idleSeconds < 45)       return setState('FOCUSED_ACTIVE');   // L0
  if (a.idleSeconds < 180)      return setState('FOCUSED_QUIET');    // 不打扰
  if (state.state !== 'INTERVENING') {
    setState('STALL_SUSPECTED');
    intervene(level = pickLevel());   // 从 L1 起，无回应再升 L2
  }

  if (timeSince(state.lastDriftCheck) > T_drift && a.idleSeconds < 45) {
    softCheckIn();                    // DRIFT_CHECK：动着也偶尔确认一次
  }
}
```

推荐用 **XState** 来落地这个 FSM——状态、转移、守卫条件都显式可视化，比一堆 `if` 好维护得多。

---

## 5. LLM 推理层（仅在干预点调用）

LLM **不在热循环里**。它只在四个时刻被唤醒：(a) 生成贴合上下文的干预话术；(b) 理解用户的自由文本回复；(c) 当用户说"开始难"时把子模块拆成更小的第一步（FOCO 式）；(d) 生成复盘。

模型选型：分类用快而便宜的（如 Haiku），对话/复盘/拆解用强一点的（如 Sonnet）。

### 5.1 人格系统提示（副驾驶 persona）

```
你是用户桌面上的专注副驾驶，服务于一位 ADHD 用户。

铁律：
- 零评判。永远不说"你又分心了""你怎么还没做"这类话。
- 卡点是被诊断的对象，不是被指责的对象。
- 极简。一次只说一句话或问一个问题。
- 你不替用户做任务，你帮他重新进入任务。

你能看到：用户当前声明的航线（哪个子模块）、他在这条航线上停了多久、
最近的历史。你看不到他屏幕上具体在做什么——所以当不确定时，
用一个温和的具体问题去问，而不是假设。
```

### 5.2 回复分类（结构化输出，供程序确定性路由）

```
用户当前航线："{submodule_title}"，已停滞 {idle_min} 分钟。
用户刚回复："{user_text}"

只返回 JSON，不要任何其他文字：
{
  "intent": "progressing" | "interrupted" | "stuck_start" | "stuck_mid" | "need_break" | "switch_task",
  "reply": "一句 ≤20 字的温和回应",
  "next_action": "none" | "split_task" | "start_break" | "refocus_anchor"
}
```

程序拿到 `intent` 和 `next_action` 后走确定性分支，`reply` 直接显示在气泡里。

### 5.3 任务拆解（当 intent = stuck_start）

```
用户卡在子模块"{submodule_title}"，迟迟无法开始。
把它拆成"第一步小到 2 分钟内能开始"的动作。
只给 1 个起步动作，不要给完整清单（清单会再次压垮 ADHD 用户）。
≤25 字。
```

### 5.4 复盘（进入 REVIEW 时）

```
这段专注里：航线 {anchor}，完成 {done} 个子模块，
被打断 {interrupt_count} 次，平均重新进入用时 {recover_min} 分钟。
用 2 句话给一个温暖、非评判的小结，并指出一个"下次可以更轻松"的点。
```

---

## 6. 浮窗 UI 与人偶

### 6.1 两种形态

- **收起态**（默认常驻）：只显示人偶 + 当前航线一行字。极小，置顶，不挡视线。
- **展开态**（hover 或点击）：三列任务，每列下面是子模块。

### 6.2 人偶绑定航线

```
点击子模块 A2
  → session.focus = { A, A2 }
  → 触发 Episode{type:'anchor'}
  → 人偶动画"跳/坐"到 A2 那一行
  → A2 高亮
```

人偶的位置 = 当前航线的可视锚。它的**姿势**反映 FSM 状态：
- `FOCUSED_ACTIVE` → 专注工作姿
- `FOCUSED_QUIET` → 侧耳倾听姿
- `STALL_SUSPECTED` → 抬头看你 / 举牌姿
- `REVIEW` → 鼓掌 / 比心姿

推荐用 **Rive** 做人偶——它原生支持"状态机驱动的角色动画"，FSM 状态可以直接映射成 Rive 的 state machine inputs，省掉大量手写动画逻辑。

### 6.3 窗口行为
- `always-on-top`、透明背景、收起态尽量小。
- 静默时**鼠标穿透**（click-through），不挡用户操作；需要交互时才接管鼠标。

---

## 7. 隐私与本地优先（把它做成卖点）

- 感知层只存时间戳/计数，**永不存键码、文本、窗口标题**。
- 历史（Episode）、偏好默认存**本地 SQLite**，不出设备。
- 唯一离开设备的数据：干预时发给 LLM 的最小上下文——也只包含**航线标题 + 状态 + 时长 + 用户主动打的字**，绝无后台采集的内容。
- 在设置页用一句话讲清楚这套边界，并提供"完全本地模式"（用本地小模型跑分类，彻底不联网）作为 Phase 3 选项。

> 行业数据显示，相当比例的 ADHD app 用户因隐私政策不清晰而流失——把"内容盲、本地优先"摆到台面上，是差异化也是留存。

---

## 8. 技术栈选型

| 层 | 推荐 | 备选 / 说明 |
|---|---|---|
| 桌面外壳 | **Tauri**（Rust + Web 前端，浮窗内存占用小，适合常驻） | Electron（出原型更快，内存更大） |
| 前端 | React + TypeScript | — |
| 人偶动画 | **Rive**（状态机驱动角色） | Framer Motion（更轻但需手写状态） |
| 全局键鼠 | Tauri: `rdev` / `device_query`；Electron: `uiohook-napi` | 只取时间戳 |
| 状态机 | **XState** | 显式 FSM，易维护 |
| LLM | Anthropic API：Haiku（分类）+ Sonnet（对话/复盘/拆解） | 流式输出供 L3 语音 |
| 语音(L3) | 系统原生 TTS | 云 TTS（更自然但需联网） |
| 本地存储 | SQLite（Tauri）/ lowdb | 默认不上传 |

---

## 9. 开发路线图

**Phase 0 — MVP（验证核心价值，可不接 LLM）**
浮窗 + 3 任务/子模块 UI + 人偶跟随点选移动 + 键鼠静止/停滞检测 + L0–L2 干预（用模板话术，先不接 LLM）+ 本地存储。
👉 这一版就已经交付了核心价值："被打断后有人轻轻喊你一声"。

**Phase 1 — 接入 LLM**
上下文化干预话术 + 回复分类 + 任务拆解（stuck_start）+ 复盘。

**Phase 2 — 自适应与语音**
学习用户个人节奏、动态调整阈值 + L3 语音 + 偏好/勿扰时段。

**Phase 3 — 升级与可选增强**
（可选、需用户额外授权）窗口标题上下文 → 大幅增强"离任务"检测；全本地模式；数据看板；多设备同步。

---

## 10. 一句话收尾

整个产品的胜负手不在 LLM 多聪明，而在三件事：**误报控制**（FOCUSED_QUIET 这一档别乱打扰）、**分级轻打扰**（永远从 L0/L1 起步 + backoff）、**内容盲的本地优先**。这三件做对了，它是解药；做错了，你自己就成了那个新的扰动。
