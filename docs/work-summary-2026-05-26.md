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
- 最新验证通过的 tag：`v3.14.1-test16`

## 晚间补充进展

### 实际排查结果

1. 复核远端后确认：`v3.14.1-test13` 标签存在，但并没有对应的 GitHub Actions workflow run。
2. 随后补跑了 `v3.14.1-test14`、`v3.14.1-test15`：
   - 两次都失败在 `Build Tauri App (macOS)`
   - 失败耗时已从此前的 40 到 60 分钟收敛到约 9 到 10 分钟
   - 说明新的简化工作流已经生效，但仍有更早的真实错误未解决
3. 本地复现 `pnpm tauri build --bundles app` 后确认：
   - 前端构建正常
   - Rust / Tauri 编译正常
   - `.app` 与 macOS `.tar.gz` updater artifact 可以产出
   - 最终失败点为 updater 签名：检测到 updater 公钥，但缺少 `TAURI_SIGNING_PRIVATE_KEY`

### 新确认的根因

- Windows 之所以能继续发布，是因为在缺少 `TAURI_SIGNING_PRIVATE_KEY` 时，会切到 `src-tauri/tauri.windows.no-updater.conf.json`，关闭 updater artifact 继续构建。
- macOS 之前没有对应的降级分支。
- 因此 macOS 构建虽然可以完成 `.app` 打包，但会在生成 updater artifact 的签名阶段失败。

### 本次新增修复

1. 将 macOS build 阶段收敛为只构建 `.app`
2. 在缺少 `TAURI_SIGNING_PRIVATE_KEY` 时，为 macOS 增加 `no-updater` 降级路径
3. 新增配置文件：`src-tauri/tauri.macos.no-updater.conf.json`
4. 保留自定义 DMG 生成逻辑，由 `Prepare macOS Assets` 继续产出 `.dmg` 和 `.zip`

对应提交：

- `e423b6f` `ci: build macos app bundle before custom dmg`
- `47ec8fe` `ci: skip mac updater artifacts when signing key is missing`

### 最终验证结果

- `v3.14.1-test16` 发布流程已在 GitHub Actions 成功跑通
- workflow run：`26455356840`
- macOS `Build Tauri App (macOS)` 成功
- macOS `Prepare macOS Assets` 成功
- Windows 发布链路同时成功
- `Publish GitHub Release` 成功
- `Assemble latest.json` 成功

### 当前已发布产物

`v3.14.1-test16` 已确认发布：

- `CC-Switch-v3.14.1-test16-macOS.dmg`
- `CC-Switch-v3.14.1-test16-macOS.zip`
- `CC-Switch-v3.14.1-test16-Windows-Setup.exe`
- `CC-Switch-v3.14.1-test16-Windows-Portable.zip`
- `latest.json`

### 当前仍需注意

1. 本次成功的 macOS 发布属于“缺少 updater 私钥时继续发布安装包”的降级路径。
2. 因此本次未生成 macOS updater 用的 `.tar.gz.sig`，`latest.json` 也不会包含 macOS 自动更新平台条目。
3. `Notarize macOS DMG` 与 `Verify macOS code signing and notarization` 仍处于临时关闭状态，后续若要恢复正式 macOS 分发质量，还需继续补齐 notarization 流程。

### 下一步建议

1. 若目标是“先可下载使用”，当前 `test16` 已满足。
2. 若目标是“恢复 macOS 自动更新”：
   - 补齐有效的 `TAURI_SIGNING_PRIVATE_KEY`
   - 恢复 macOS updater artifact 与 `.sig`
   - 验证 `latest.json` 中出现 `darwin-aarch64` / `darwin-x86_64`
3. 若目标是“恢复正式 macOS 分发”：
   - 重新开启 `Notarize macOS DMG`
   - 重新开启签名与 notarization 校验
