# Root Cause & Fix Summary

## 原因分析 (Root Cause)
上个 commit (`1815afa`) 在 `packages/macos-ax-helper/Sources/MacosAxHelper/SafariController.swift` 的 `executeJavaScript` 中引入了如下判断：

```swift
let isBlockStatement = normalized.contains("\n")
    || normalized.contains(";")
    || normalized.hasPrefix("return ")
    ...
let body = isBlockStatement
    ? "\(normalized);return 'ok';"
    : "var __r=(\(normalized));return typeof __r==='string'?__r:(JSON.stringify(__r===undefined?null:__r)||'null');"
```

`networkRecorderClient.ts` 中的探针、注入、轮询等脚本（如 `buildNetworkRecorderProbeScript()`、`buildNetworkRecorderInstallScript()`、`buildNetworkRecorderPollScript()`）全都是多行 IIFE `(function(){ ... return { ... }; })()`。由于它们内部包含换行 `\n` 和分号 `;`，被误判为 `isBlockStatement = true`，导致执行体变成：

```js
(function(){ ... return { installed: ... }; })();return 'ok';
```

IIFE 的返回值被直接丢弃，固定返回字符串 `"ok"`，随后 `networkRecorderClient.ts` 执行 `JSON.parse(res.result || "{}")` 时便抛出 `JSON Parse error: Unexpected identifier "ok"`，前端显示 `Injection Failed`。

## 修复内容 (Fix)
修改 [`SafariController.swift`](file:///workspace/agyPreviewInstant/packages/macos-ax-helper/Sources/MacosAxHelper/SafariController.swift)：
1. 新增 `isBlockOrMultiStatement(_:)` 词法级扫描器，在忽略字符串（单引号、双引号、模板字符串）和注释（`//`、`/* */`）以及括号嵌套深度（`()`、`{}`、`[]`）后，仅在顶层出现分号或以语句关键字开头时才判定为 block/multi-statement。
2. 对单表达式（包括多行 IIFE、带尾部分号的表达式等），先剥离末尾多余分号，再通过 `var __r=(\n...\n)` 保留并序列化返回值。
3. 对真正的 block/multi-statement 脚本，包裹在内层 `(function(){\n...\n})()` 中执行并序列化其返回值，既保留返回值又避免 AppleScript `OSStatus -2753`。

## 验证结果 (Verification)
- `pnpm run env:check`：通过（OverlayFS 8 MB <= 100 MB）。
- `pnpm --filter p2p-remote-controller check`（`tsc --noEmit && eslint src`）：通过。
- `pnpm --filter p2p-remote-controller test`（`vitest run`）：11 个测试文件、42 个用例全部通过。
- 注：当前沙箱为 Ubuntu 24.04，无 Xcode/Swift 工具链，请在 macOS 上运行 `pnpm ax-helper:build` 重新编译 `macos-ax-helper` 二进制以生效。

## 提交信息 (Commit)
- 分支：`a2`
- Commit：`7fd33f2f64a243f87102f6b48664cb4e71f90e46`（Verified Bot Commit，`signature.isValid = true`）
- 链接：https://github.com/Antigravi-ty/agyPreviewInstant/commit/7fd33f2f64a243f87102f6b48664cb4e71f90e46
