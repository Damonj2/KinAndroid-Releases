# Kin 1.3-beta.1 · UI Cleaner 收口

> 本轮从 1.2-beta.11 线起做了一轮三阶段收口：**Runtime 编排统一 → 命中统计统一 → UI 组件收口**。

## 本轮改动

### A. Runtime 收口（对齐 GKD 1.12 官方语义）

- **Runtime 编排迁移**：`KinRuleSchedulerHost` 收敛为官方 delays + scope + actionDispatcher 三件套；每 (rule, kind) 唯一调度；删除旧 `ForcedScheduler` / `runShadowPipeline` 路径
- **DIV-014 · CONTENT_CHANGED 节流**：移植官方三重条件节流，抑制同窗口内重复 query
- **DIV-013 · interruptKey / guardInterrupt**：移植官方中断四判据（含 `lastTriggerTime` / `appChangeTime` 3000ms 阈值 + 300ms delay）
- **query single-flight**：`querying` 闸门覆盖 event / delay-requery / reentry 三处触发源，丢弃语义与官方一致

### B. 统一命中统计口径

此前 `CleanerViewModel` 与 `UiCleanerViewModel` 各自 map/take/count 同一份 hitHistory，且规则数一处读硬编码、一处读订阅解析 → 数字必然对不上。

- 新增 `UiCleanerRuntimeUiState`（唯一聚合状态定义）
- 新增 `HitStatsProjection`（todayHits / totalHits / recentHits 唯一投影，`success` 字段透传）
- 两个 ViewModel 删除各自的统计口径，统一走投影
- 规则数口径统一为**订阅真实解析**（`refreshLoadedRules`），内置规则仅兜底

### C. UI 组件收口

- 新增 `CleanerCommonUi`：`SectionTitle(compact)` / `HitRow` / `CategoryRow`
- `UiCleanerScreen` 与 `CleanerScreen` 删除各自逐字重复的私有副本

### D. 构建修复

- **MockLocation lint 门禁**：`ACCESS_MOCK_LOCATION` 是虚拟定位必需声明（不声明则开发者选项「选择模拟位置应用」列表看不到本 App），release 构建被 `MockLocation` 致命检查拦截 → 加 `tools:ignore="MockLocation"` 并写明理由，不移入 debug manifest

## 验证

- `clean testDebugUnitTest assembleDebug -x lint` → `BUILD SUCCESSFUL in 5m 14s`
- `lintVitalRelease` → `BUILD SUCCESSFUL`
- 全量单元测试：0 failures / 0 errors
- APK：**zipalign 通过 · v2/v3 签名通过** · 签名指纹与历史版本一致（`3d7be670...`，可覆盖安装）

## 已知缺口（不谎报）

- **source-first / root-fallback instrumentation 未验**：本机无模拟器，未做真机验证，标 **NOT VERIFIED**
- `GkdSelectorEngine`（LEGACY-only）由 DIV-001 双闸门隔离，未删除

## 升级方式

设置 → 关于 → 检查更新，或直接下载 APK 覆盖安装（签名一致，无需卸载）。
