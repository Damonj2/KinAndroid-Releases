# Kin 1.2 Beta

新 GKD Core Beta 接管真实 Action 执行。本版为 **Beta**，已知问题见下。

## 主要变化

- **新 GKD Core Beta 接管** — kin-gkd-core pipeline 成为默认真实 Action 执行引擎
- **官方 GKD selector/runtime 核心集成** — 依据上游 commit `a16036644cd761dd8306ef93f9abb0783a4dc9d8`
- **新 UI Cleaner engine** — 新/旧引擎互斥，任意时刻仅一个真实 Action executor
- **DecisionLog 诊断基础** — 规则决策记录，支持按 package 过滤与导出

## Known issues

本轮仅记录，不修复：

- Accessibility permission may unexpectedly turn off
- Saving local UI-cleaner rules may crash
- Navigation back to Home may occasionally fail

## 构建信息

| 项 | 值 |
|---|---|
| versionName / versionCode | 1.2-beta / 11 |
| 源码 commit | `36db6dac54ee11fb6a67b1d8540002c4139185ed` |
| 构建类型 | debug（Android Debug 证书） |
| minSdk / targetSdk | 24 / 35 |

## 校验

发布前请用 `SHA256SUMS.txt` 校验：

```
sha256sum -c SHA256SUMS.txt
```

| 文件 | SHA-256 |
|---|---|
| Kin-1.2-beta.apk | `1c50dfe4fc4e669e94d18d9bd63c08db81956f82e46696536b4fad9c2ce975e7` |
| Kin-Android-1.2-beta-source.tar.gz | `15e93af461e901e383bf3e46e0d82403cf9e337bef68ff1b27f74ad51764deb1` |

## 对应源码（GPL-3.0 合规）

本发行物包含官方 GKD 核心（GPL-3.0），按 GPLv3 提供对应源码：

- `Kin-Android-1.2-beta-source.tar.gz` — 对应上述发布 commit 的源码快照
- 包含 vendored GKD core 与其上游 LICENSE、抽取边界说明、第三方归属声明、构建脚本
- 排除项及各自的公开获取方式见包内 `.excluded/EXCLUDED.md`
- GKD 上游：<https://github.com/gkd-kit/gkd>，commit `a16036644cd761dd8306ef93f9abb0783a4dc9d8`
- sing-box（libbox 依赖）：<https://github.com/SagerNet/sing-box>，固定 tag `v1.14.2`

本版不做法律审查声明，仅按项目当前 GPL 使用方式提供源码与许可证分发基础。

## 安装说明

- debug 签名覆盖安装，无需卸载
- 覆盖安装后请确认无障碍服务重新开启，否则不会执行任何 Action
