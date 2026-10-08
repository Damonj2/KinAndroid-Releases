# Kin Android v1.2-beta.8 — Host query/root 对账修复

**versionCode:** 19 · **versionName:** 1.2-beta.8 · **HEAD:** `e6881ac`

---

## 概述

本次修复 **Host query/root 时间错配**，它已被真机证据确认为
「GKD 规则在 Kin 里 0 命中、而官方 GKD 能跳过」的直接原因。

同时补齐 Android 位置权限模型（独立兼容性修正）。

---

## 根因（真机确认）

```
com.xiaomi.shop 冷启动，21/21 条 HOST_ROOT_TRACE 全部 queryRootMatch=false
  queryPkg = com.xiaomi.shop
  rootPkg  = com.miui.home
  topPkg   = com.xiaomi.shop

event 路径 74 条 / 49 条 mismatch
reentry 路径 3 条 / 0 条 mismatch

全库统计：FIRST_ZERO_TRACE 192 条，selectorMatches = 0，actionSuccesses = 0
```

机制：

```
旧 AccessibilityEvent(pkg=A)
   ↓ 事件延迟/积压（真机洪峰 1207–1391 事件 / 10 秒）
   ↓ 处理时 rootInActiveWindow 已变成 B
   ↓ Kin 仍按 A 取 candidate rules
   ↓ A 的 selector 去查 B 的节点树
   ↓ selectorMatches 恒为 0
```

AccessibilityEvent 被当成了「root 一定属于 event.packageName」的证明，
它在积压下并不成立。

---

## 修复

Host 现在把 **fresh root 当作执行真值**，每次进入 `pipeline.process()` 前对账：

| 条件 | decision | 行为 |
|---|---|---|
| queryPkg == rootPkg | ALIGNED | 正常执行 |
| queryPkg != rootPkg | REBOUND_TO_ROOT | 用 rootPkg 重建查询 |
| root.packageName 为空 | DROPPED_NO_ROOT_PKG | 安全退出 |
| rootPkg == Kin 自身 | DROPPED_SELF | 不执行规则 |
| rootPkg ∈ block 名单 | DROPPED_BLOCKED | 不执行规则 |

activity 重建规则（不伪造）：

- tracker topPkg == rootPkg → 沿用 tracker 的 activity
- topPkg != rootPkg → **activity = null**（绝不把 stale event 的 activity 搬到别的 App 上）

**核心不变量：**

```
任何进入 pipeline.process(root, appId=X) 的调用
必须满足 X == root.packageName
```

Host helper 违反该不变量时 fail-closed（记录并拒绝执行）。

event 与 reentry 两条路径走**同一套** reconciler。reentry 复用既有已验证路径
（真机 3/3 queryRootMatch=true），未新造第二套 scheduler。

---

## 新增诊断：HOST_RECONCILE

```
trigger / eventPkg / originalQueryPkg / rootPkg / topPkg
decision / effectiveQueryPkg / effectiveActivity
```

decision ∈ `ALIGNED | REBOUND_TO_ROOT | REENTRY_SCHEDULED | DROPPED_NO_ROOT_PKG | DROPPED_SELF | DROPPED_BLOCKED`

---

## block 名单（最小补齐）

```
com.miui.home
com.android.systemui
```

理由：修复 mismatch 后 queryPkg 会改用 rootPkg，而真机 mismatch 经常落到
launcher / systemui。若不 block，就会开始对系统桌面大量执行全局规则。

这两个包本就是 system app，官方语义经 `systemAppIds` + `!matchSystemApp` 已应排除。
本列表是 Host 侧最小兜底，**不是**复制 GKD 全部 block 列表。

> 附带发现（本轮未修，仅记录）：生产侧 `RuleRuntime()` 无参构造，
> `environment()` 恒为空（`launcherAppId=""`, `systemAppIds={}`），
> 且 `KinSubscriptionBridge` 中 `isSystem` 硬编码 `false`。
> 真机可见 `com.android.systemui global=7` × 64 次，即全局规则未按官方语义排除系统 App。
> 这是独立于本轮修复的第二处偏差，需后续单独评估。

---

## 位置权限（独立兼容性修正）

- 声明 `ACCESS_COARSE_LOCATION` + `ACCESS_FINE_LOCATION`
- 设置 → 权限 新增「位置权限」行，点击走标准 runtime launcher 同时请求两者
- 新增只读诊断 `LOCATION_PERMISSION_STATE`（fine / coarse / mockLocationAllowed）
- **不声明** `ACCESS_BACKGROUND_LOCATION`
- 保留 mock location 授权机制（开发者选项 → 选择模拟位置信息应用 → Kin，AppOps `android.mock_location`）
- **未修改** `MockLocationService` 定位注入逻辑
- 不宣称普通位置权限是虚拟定位失效的根因，仅补齐权限模型

---

## 未改动（明确边界）

GKD parser、selector semantics、NodeAdapter、CandidateIndex 数据结构、
runtime timing semantics、ActionBridge、subscription enable 逻辑、Golden Sources。

---

## 测试

```
kin-gkd-core  141 / 141
app           331 / 331   （其中新增 10 条 reconciler 测试）
失败 0 · 错误 0 · 跳过 0
```

新增测试覆盖：aligned / stale event / stale tracker / null root / self /
blocked / reentry 不回归 / 核心不变量。

---

## 构建信息

| 项 | 值 |
|---|---|
| versionName / versionCode | 1.2-beta.8 / 19 |
| 源码 commit | `e6881ac` |
| 构建类型 | debug（Android Debug 证书） |
| minSdk / targetSdk | 24 / 35 |

---

## 真机验证步骤

1. App 内更新到 beta.8
2. 开启 Kin 无障碍
3. 冷启动小米商城 3 次
4. 观察是否跳过开屏
5. 立即导出 Kin Diagnostic Bundle

期望（若修复生效）：

```
queryRootMatch=false 的 selector 执行数 → 0
actionAttempts / actionSuccesses 出现 > 0
```

若仍 0 命中，先看新导出日志：

```
queryRootMatch 是否已全部对齐
selectorMatches 是否仍为 0
```

只有出现 `queryRootMatch=true` 但 `selectorMatches=0`，才进入下一层
（raw node tree vs 官方 GKD 1.12.1 vs Kin NodeAdapter/selector）。

---

## 校验

```
sha256sum -c SHA256SUMS.txt
```

| 文件 | SHA-256 |
|---|---|
| Kin-v1.2-beta.8-Host-query-root-reconcile-debug.apk | `bb4bbee91c1e8a0702089d45c9c29e8a5d6886143eede2e14072cd8f17a95f4e` |
| Kin-Android-1.2-beta.8-source.tar.gz | `0c4f1305a4842e1f3679f6d99ef0dd7f66a351fd6ee806275d31e788a2b044ce` |

---

## 对应源码（GPL-3.0 合规）

本发行物包含官方 GKD 核心（GPL-3.0），按 GPLv3 提供对应源码：

- `Kin-Android-1.2-beta.8-source.tar.gz` — 对应 `e6881ac` 的源码快照
- 含 vendored GKD core 与其上游 LICENSE、抽取边界说明、第三方归属声明、构建脚本
- GKD 上游：<https://github.com/gkd-kit/gkd>，commit `a16036644cd761dd8306ef93f9abb0783a4dc9d8`
- sing-box（libbox 依赖）：<https://github.com/SagerNet/sing-box>，固定 tag `v1.14.2`

## 安装说明

- debug 签名覆盖安装，无需卸载
- 覆盖安装后请确认无障碍服务重新开启，否则不会执行任何 Action
