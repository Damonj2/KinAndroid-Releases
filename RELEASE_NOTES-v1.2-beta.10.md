# Kin Android v1.2-beta.10 — blocked 转场重入 + 运行时凭据

**versionCode:** 21 · **versionName:** 1.2-beta.10 · **HEAD:** `e3886e7`

---

## 概述

本版本两条**互相独立**的改动，分别修一个真机暴露的问题：

1. `ui-cleaner`：blocked root 是转场假象时不再直接丢弃（这是「0 命中」排查主线的下一步）
2. `diagnostics`：上传凭据从构建期 shared secret 改为运行时 Android Keystore

两者无耦合，可分开验证、分开回滚。

---

## 改动一：blocked 转场不再直接丢弃

### 真机证据

`com.coolapk.market` 上报 51/51、57/57 全部 `DROPPED_BLOCKED`。

### 根因

不是 root 选错，而是**转场瞬间**的时序问题：

```
eventPkg = com.coolapk.market      ← 事件已在目标 App
topPkg   = com.coolapk.market      ← tracker 已是目标 App
rootPkg  = com.miui.home           ← 但 rootInActiveWindow 还是 launcher
```

原逻辑判定 `rootPkg in BLOCKED_PKGS` → 直接 drop。
后果：本次转场**彻底失去机会** —— selector 永不执行，
`FIRST_ZERO_TRACE` 恒为 0，看起来就像「规则不生效」。

### 修法

复用既有 reentry scheduler，加一个新 decision：

```
rootPkg in blockedPkgs && queryPkg != null && queryPkg != rootPkg
  → DROPPED_BLOCKED_REENTRY_SCHEDULED
      shouldProcess          = false   （绝不拿 launcher root 查目标 App 规则）
      shouldScheduleReentry  = true    （300ms 后重读 fresh root）
```

### 保留的约束

| 约束 | 说明 |
|---|---|
| 稳态仍是纯 drop | `rootPkg == queryPkg`（自己就是 launcher）→ `DROPPED_BLOCKED`，否则会在 launcher 上无限重入 |
| 核心不变量未动 | `effectiveQueryPkg` 恒为 null，`invariantHolds` 一行未改 |
| 不新造调度 | 用的是既有 `scheduleQueryReEntry` |
| activity 传 null | 原事件的 activity 属转场前旧窗口，交给 reentry 时的官方 `matchActivity` 语义 |

### 涉及文件

- `uicleaner/HostRootReconciler.kt` — 新 decision + `requiresReentry()`
- `uicleaner/UiCleanerService.kt` — blocked 分支在 return 前处理 `shouldScheduleReentry`

---

## 改动二：上传凭据改为运行时

### 为什么

Release APK 公开可下载，把 shared secret 编进 `BuildConfig` 等于公开它。

### 做了什么

- 新增 `DiagnosticKeyStore`：AES-GCM，密钥由 Android Keystore 生成保管、
  进程外不可导出；密文与 IV 分开存 DataStore
- 移除 `buildConfigField` / `local.properties` / 环境变量三条注入路径
- `DiagnosticUploader.upload()` 改为 key 必传，本类不再读 BuildConfig、
  不持有密钥字段、不缓存 key
- 诊断页新增「诊断上传凭据」区（保存 / 清除 / 显隐）+ 一键上传
- `DiagnosticId` 区分两套校验：本地十六进制 vs 服务端 Crockford Base32
  （剔除易混淆 `I/O/L/0/1`）—— 用本地正则校验服务端 ID 会把合法值误判

### 不变量

任何异常一律收敛为 `Failure`，绝不 crash；401 时不回显 key。

---

## 测试

全量 **625/625** 通过（`app` + `kin-gkd-core`）。

`HostRootReconcilerTest` 13 例，其中新增 3 例：

- blocked transition 排重入（核心新行为）
- blocked 稳态纯 drop（防无限重入）
- 非 blocked drop 不排重入

---

## ⚠️ 未验证项（重要）

**诊断上传链路从未端到端跑通过。**

- R2 bucket `kin-diagnostics` 目前**为空**，从未有过成功上传
- `scripts/fetch-diagnostic.sh` 所用的对象键约定
  `diagnostics/YYYY-MM-DD/<ID>.zip` **未经真实上传验证**
- Worker `/health` 返回 200 正常，但 `/upload` 需要真实 key 才能测
- 需要 owner 在真机上配置真实 key 后，才能确认上传是否可用

**不要**因为「诊断页看起来正常」就认为上传可用 —— 未验证就是未验证。

---

## 安装

- 签名：`CN=Android Debug`
  SHA256 `3d7be6702e63436edc48c67c2abad5622024673985f69fab66ca066f7a1a55b8`
  （与历史版本同一证书，可覆盖安装）
- `versionCode 21 > beta.9 的 20`，OTA 可识别
