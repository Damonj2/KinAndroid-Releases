## Kin 1.3-beta.10 · 移除 GKD 界面净化，广告过滤统一走 DNS 层

本版按 owner 决策做了一次**减法**：删掉整套 GKD 界面净化（无障碍点按），广告过滤只保留 sing-box DNS 域名拦截。

### 删除
- **`kin-gkd-core` 模块整体移除**（106 文件 / 47MB），含 vendored 的 GKD 选择器、规则引擎与运行时测试
- **无障碍服务全部下线**：`UiCleanerService`、`accessibility_service_config.xml`、manifest 声明、测试 Activity
- **相关 UI**：应用内净化、应用规则、界面净化订阅、命中记录四个页面及其路由
- **GKD 专属诊断通道**：`accessibility.log`、`local-rules.log`、`decision.log`、`first-zero.log` 收敛为 `navigation.log` / `crash.log` / `general.log`
- 相关单测与测试夹具

合计 `221 files changed, 71 insertions(+), 35523 deletions(-)`。

### 保留
广告过滤统一到既有 DNS 层能力，未新增自研拦截逻辑：
- **HaGeZi 三档规则**（轻量 / 标准 / 强力），互斥启用
- **白名单**：增删改查 + 搜索
- **规则更新**：下载校验后落盘，离线回退内置 `geosite-category-ads-all.srs`
- **真实 DNS 探针**：对广告域名与正常域名各做一次真实查询，判定是否真在 DNS 层拦截——不做假统计

### 入口收敛
- 工具页「过滤工具」4 项 → 1 项「广告过滤」
- 设置页移除「应用内净化」行
- 订阅页「广告过滤订阅」改为「广告过滤（DNS 级）」，卡片改读真实状态（档位 / 启用 / 规则版本 / 上次更新 / 白名单条数）

### 验证
- `compileDebugKotlin` 通过
- `testDebugUnitTest`：**193 项，0 failures**
- APK 实测：manifest 无 accessibility 声明；16 个 dex 中 GKD 类命中合计 **0**

### 说明
- 本包为 debug 构建，用于内测验证。versionCode **32**（> 线上 31）
- **能力变化（务必知悉）**：DNS 层只能拦**域名**。App 自身域名下发的开屏广告（如运营商 `StartPageActivity`）与正文同域名，DNS 拦不掉——这是移除无障碍点按后的**能力降级**，非缺陷。
- GKD 上游（GPL-3.0）已从本发行物中完全移除，故不再附带其源码快照；sing-box 归属声明保持
