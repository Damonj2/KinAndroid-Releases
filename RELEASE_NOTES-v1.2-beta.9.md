# Kin Android v1.2-beta.9 — 诊断插桩：gate 拆分归因

**versionCode:** 20 · **versionName:** 1.2-beta.9 · **HEAD:** `593277b`

---

## 概述

本版本是**纯诊断插桩版本**，不改变任何执行语义。

目标：让「GKD 规则在 Kin 里 0 命中」的排查能**读数据**而不是猜。
上一版（beta.8）修了 Host query/root 对账，本版补上「谁挡住了事件」的观测。

---

## 改动

### 1. gate 拆分计数

`EntryDiagnostics` 原先只有一个笼统的 `disabledSkip` 计数器，总开关关闭
和暂停期都会 +1，无法区分是谁挡的。本版拆成两个独立字段：

| 字段 | 触发条件 |
|---|---|
| `eventsDisabledMasterSkip` | 总开关关闭（master disabled） |
| `eventsDisabledPausedSkip` | 一键暂停（paused） |

优先级 `MASTER_DISABLED > PAUSED`，与 `UiCleanerService` 中
`if (!currentMasterEnabled || currentPaused)` 的短路顺序一致。
两个都关时记 `MASTER_DISABLED`，保证 reason 文案与计数永远对得上。

### 2. UI_CLEANER_GATE_STATE（新增诊断）

三个来源统一落盘 gate 状态：

```
SERVICE_CONNECTED   无障碍服务连接瞬间
DATASTORE_UPDATE    masterEnabled / paused 被改写时
DISABLED_SKIP       事件被 gate 拦下时
```

字段：`masterEnabled / paused / managerEnabled / accessibilityEnabled /
engineMode / enabledSubscriptions / indexedRules / reason / source`

`reason ∈ RUNNING | MASTER_DISABLED | PAUSED`

### 3. APP_RULE_STATS_TRACE（新增诊断）

应用规则页「订阅 / 可执行 / 不兼容」三个数显示 0 时，用这条对账三层口径：

```
storedGroups / storedRules      落盘层
parsedGroups / parsedRules      解析层
statsSubscribed / statsExecutable / statsIncompatible   RuleStats 计算层
```

用于区分：落盘丢组、读取丢 rule，还是 RuleStats 计算差异。

### 4. 引擎状态区显示真实订阅名

原先 `EngineStatusSection` 写死显示 `"Lin-arm / ganlinte"`，与实际加载的
订阅无关，属误导性展示。现改为读订阅 JSON 的真实 `name`，只显示已启用项；
无订阅时显示「未添加订阅」。

### 5. 其他

- `GkdParser` 历史注释由厂商代号改为通用表述（非功能性）
- `CleanerViewModel` 补加载态刷新，避免订阅加载完成前误显示「0 条订阅」

---

## 未改动（明确边界）

GKD parser、selector semantics、NodeAdapter、CandidateIndex、
runtime timing semantics、ActionBridge、subscription enable 逻辑、
pipeline 执行路径、Host reconciler。

**本版本不声称修复任何命中问题**——它只提供观测能力。

---

## 测试

```
app            全量单测 BUILD SUCCESSFUL
kin-gkd-core   debug + release 全量单测 BUILD SUCCESSFUL
```

新增测试：
- `master and paused gate skips are counted separately` — 两 gate 独立计数 + 独立渲染
- `flush clears both gate skip counters` — 窗口落盘后归零
- 修正 `render contains all入口字段` 的过时断言（`disabledSkip=` → 两个新字段）

---

## 构建信息

| 项 | 值 |
|---|---|
| versionName / versionCode | 1.2-beta.9 / 20 |
| 源码 commit | `593277b` |
| 构建类型 | debug（Android Debug 证书） |
| minSdk / targetSdk | 24 / 35 |

---

## 真机验证步骤

1. App 内更新到 beta.9
2. 开启 Kin 无障碍
3. **先确认 gate 开着**：设置里总开关开启、未暂停
4. 冷启动目标 App（如小米商城）3 次
5. 导出 Kin Diagnostic Bundle

期望读法：

```
先看 UI_CLEANER_GATE_STATE.reason
  RUNNING          → gate 不是问题，往 selector 层查
  MASTER_DISABLED  → 总开关被关了（看 source 是哪个来源改的）
  PAUSED           → 暂停未解除

再看 disabledMasterSkip / disabledPausedSkip
  非 0 且 reason=RUNNING  → 状态在运行中被改写，看 DATASTORE_UPDATE 时间线

再看 APP_RULE_STATS_TRACE
  三个 stats* 全 -1        → pkg 不在 stats 索引里（key 口径不一致）
  statsSubscribed=0 但 stored 有 → RuleStats 计算层问题
  stored 就是 0            → 落盘/读取层问题
```

---

## 校验

```
sha256sum -c SHA256SUMS.txt
```

| 文件 | SHA-256 |
|---|---|
| Kin-v1.2-beta.9-gate-split-subscription-names-debug.apk | `282f2b986df59e13b800d9eb5853e96904272f8f0c3d02996529073cb33ea831` |
| Kin-Android-1.2-beta.9-source.tar.gz | `89e9f82e75c8419323b3f6bfd63bb09f2d6a61800cb27ffe43ba8ceda180bb95` |

---

## 对应源码（GPL-3.0 合规）

本发行物包含官方 GKD 核心（GPL-3.0），按 GPLv3 提供对应源码：

- `Kin-Android-1.2-beta.9-source.tar.gz` — 对应 `593277b` 的源码快照
- GKD 上游：<https://github.com/gkd-kit/gkd>，commit `a16036644cd761dd8306ef93f9abb0783a4dc9d8`
- sing-box（libbox 依赖）：<https://github.com/SagerNet/sing-box>，固定 tag `v1.14.2`

## 安装说明

- debug 签名覆盖安装，无需卸载
- 覆盖安装后请确认无障碍服务重新开启，否则不会执行任何 Action
