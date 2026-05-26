# 2026-05-26 工作总结

## 今日完成

1. 将项目内 Codex 和 Claude 的默认请求地址统一切换为 `https://api.tokenstore.me`。
2. 更新仓库相关配置，使项目发布信息指向 `zzktz/ts-switch`。
3. 配置并推送远端仓库 `git@github.com:zzktz/ts-switch.git`。
4. 调整 GitHub Actions 发布流程，完成 Windows 安装包产出链路修复。
5. 将 Windows 安装包方案从 `.msi` 调整为 `NSIS .exe`，并已验证可成功生成。
6. 重新启用 macOS 打包流程，并接入 Apple 证书 / API Key 相关 secrets。
7. 针对 macOS 构建失败问题进行了多轮排查与工作流收敛。

## 已确认结果

### Windows

- Windows 发布链路已跑通。
- 当前产物为 `.exe` 安装包。
- 相关成功版本已经验证过可由 GitHub Actions 生成。

### macOS

- Apple 证书、`.p12`、`.p8`、Team ID、Issuer ID、Key ID 等信息已梳理完成。
- GitHub Secrets 已按工作流所需名称接入。
- 之前多次失败均集中在 `Build Tauri App (macOS)`。
- 旧流程问题包括：
  - 使用 `universal-apple-darwin`
  - build 阶段自带重试与等待，导致一次失败耗时接近 40 到 60 分钟
  - 尚未进入稳定的独立公证阶段

## 今天最后一次调整

为尽快缩小 macOS 问题范围，已新增一版更保守的工作流调整：

1. macOS 构建由 `pnpm tauri build --target universal-apple-darwin` 改为 `pnpm tauri build`
2. 去掉 macOS build 的三次重试与等待
3. 将 macOS build 超时降为 30 分钟
4. 临时关闭 `Notarize macOS DMG`
5. 临时关闭 `Verify macOS code signing and notarization`

对应提交：

- `a723b855` `ci: simplify macos signed build`

对应测试标签：

- `v3.14.1-test13`

## 当前判断

- Windows 方向已经基本完成。
- macOS 当前问题仍聚焦在“签名构建阶段”，还没有稳定推进到独立公证阶段。
- `test13` 的目标是先验证“单架构签名构建”是否能通过。

## 下次继续建议

1. 先看 `test13` 结果。
2. 如果 `test13` 仍失败：
   - 重点检查 Tauri macOS 签名构建本身
   - 排查证书身份、keychain、codesign 或 Tauri bundle 配置
3. 如果 `test13` 成功：
   - 再逐步恢复 DMG 公证
   - 最后恢复 notarization 验证步骤

## 关键参考信息

- 仓库：`git@github.com:zzktz/ts-switch.git`
- macOS Team ID：`SC25G9G7G7`
- Apple API Key ID：`93QNL8376M`
- Apple API Issuer ID：`6e81cd2f-e8e1-4b44-8cfb-e7d944e219fc`
- 最新用于缩小问题范围的 tag：`v3.14.1-test13`
