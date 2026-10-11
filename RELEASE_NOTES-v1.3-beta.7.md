## Kin 1.3-beta.7 · 诊断取证 + 进页面卡顿修复

本版集中修掉三个已确证的缺陷，均为「证据先行」——先取真机诊断包或源码证据，再动手。

### 诊断
- **FIRST_ZERO_TRACE 单开通道**：它此前写在 general.log（300 容量），而该通道有 14 个日常上报点、每次 process 又产一条 trace，点几下 App 就把窗口挤爆——诊断包里根本看不到「0 命中断在哪一环」。真机包实测：general.log 223 条里 192 条（86%）都是它。现改为独立通道 `first-zero.log`。
- **INDEX_REBUILD 的 appGroupsEnabled 不再硬编码 0**：该字段是判定「App 级订阅覆盖有没有进引擎」的直接证据，此前恒报 0，会把排查引向错误结论。现取索引里的真实 App 级组数。

### 性能
- **应用规则页主线程磁盘 I/O 移出**：该页四处读盘（含 610 App 全量 JSON 解析）此前都在主线程（`viewModelScope` 默认 `Main.immediate`），是进页面/拨开关卡顿的真因。现读盘移入 IO、纯计算移入 Default；批量分类改由 IO 线程异步加载。行为不变，仅调度线程变化。

### 验证
- `kin-gkd-core` 171 + `app` 427 项测试，0 failures
- 新增回归用例锁死 FIRST_ZERO_TRACE 的通道归属，防止回退

### 说明
- 本包为 debug 构建，用于内测验证。versionCode 29（> 线上 28）。
