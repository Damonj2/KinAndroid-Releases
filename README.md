# KinAndroid-Releases

Kin Android 官方发布通道。

本仓库**只**用于分发：

- APK 发行产物
- 更新元数据（`latest.json`）
- 校验和（`SHA256SUMS.txt`）
- Release notes
- 对应版本的源码快照与许可证文件

日常开发源码不在本仓库同步，开发仓库为私有仓库。

## 检查更新

Kin 客户端读取仓库根目录的 `latest.json`，通过比较 `versionCode` 整数大小
判断是否有新版。**不要**用 `versionName` 字符串比较大小。

本通道使用自建的 `latest.json` 判断更新，因此不依赖 GitHub `/releases/latest`
对 prerelease 的特殊行为。

## 校验下载产物

```bash
sha256sum -c SHA256SUMS.txt
```

## 对应源码（GPL-3.0）

Kin 发行物包含官方 GKD 核心（GPL-3.0）。每个 Release 均附带对应版本的源码快照
`Kin-Android-<version>-source.tar.gz`，其中包含 vendored GKD core 及其上游
LICENSE、抽取边界说明、第三方归属声明与构建脚本。

- GKD 上游：<https://github.com/gkd-kit/gkd>
- sing-box：<https://github.com/SagerNet/sing-box>（固定 tag `v1.14.2`）

## Releases

见本仓库的 [Releases](../../releases) 页面。
