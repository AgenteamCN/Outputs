# Research Report: Externalizing SagerNet/sing-box Outside the HarmonyOS NEXT .hap

## 1. Executive Summary

- **Can `sing-box` be externalized into `filesDir` (runtime-downloaded or user-imported `.so` / binary) and loaded via `dlopen` or `NativeChildProcess`?**
  - **No.** HarmonyOS NEXT enforces **mandatory code signing** (`hapsigner` + kernel `fs-verity` Merkle verification + `ownerId` checks) and **W^X / SEPolicy** restrictions on writable app sandbox directories (`filesDir`).
  - `dlopen()` on an unsigned `.so` in `filesDir` (`/data/storage/el2/base/files`) is blocked by code-signing / `PROT_EXEC` enforcement.
  - `OH_Ability_StartNativeChildProcessWithConfigs` only accepts a library name resolved from the **signed `libs/arm64-v8a/` bundle directory** (`bundleCodeDir`), not an arbitrary `filesDir` path.
  - Furthermore, even if runtime `dlopen()` were possible, dynamically loading a GPLv3 `.so` into the host app's address space **still** triggers GPLv3 copyleft under FSF's linking rules.
- **Can `sing-box` be externalized into a separate companion `.hap`?**
  - **Yes — this is the cleanest GPLv3 isolation boundary on HarmonyOS NEXT**, provided the companion `.hap` is distributed separately under GPLv3 and communicates with the closed-source host `.hap` solely via OS-level networking (`VpnExtensionAbility` or `127.0.0.1:<port>` HTTP/SOCKS5) without shared `.so` linking.
  - **Key platform constraint**: Third-party apps on tablets/phones cannot use `AppServiceExtensionAbility` (PC/2-in-1 + enterprise ACL only) and are frozen in the background unless using a valid `backgroundTaskManager` category or **`VpnExtensionAbility`**.
- **Practical Recommendation for MatePad Mini (HarmonyOS 7.0.x)**:
  - **If `sing-box` is required and host app must stay closed-source**: Ship a **separate GPLv3 companion `.hap`** using `VpnExtensionAbility` (such as `sing-box-for-harmonyos` / `NekoBox4Harmony` or a minimal proxy `.hap`) so it stays alive in the background, and let the host app's `ArkWeb` either route transparently through the VPN or connect to `127.0.0.1:<port>` via `webview.ProxyController.applyProxyOverride`.
  - **If a single-`.hap`, zero-external-app UX is required with a closed-source host**: Replace `sing-box` (GPLv3) with an **MPL-2.0 (`XTLS/Xray-core`)** or **MIT (`shadowsocks-rust` / `v2ray-core`)** core bundled as a signed `c-shared` `.so` inside `libs/arm64-v8a/`.

---

## 2. Runtime-Downloaded / User-Imported `.so` or Binary in `filesDir`

### 2.1 Mandatory Code Signing (`fs-verity`) & Sandbox Directory Rules

1. **Signed Bundle vs. Writable Data Directories**:
   - According to Huawei's official HarmonyOS sandbox specification (*应用沙箱目录*):
     - `bundleCodeDir` (`/data/storage/el1/bundle`, mapped from `/data/app/el1/bundle/public/<PACKAGENAME>`) is explicitly designated as the directory whose code resources can be dynamically loaded (*"此路径下存储的代码资源数据可以被用于动态加载"*).
     - `filesDir` (`/data/storage/el2/base/files`) is designated strictly for private user/application data (documents, media, logs), not executable code.
   - According to OpenHarmony's `developtools_hapsigner` and `hapsigner-rs` format specification:
     - All executable code (`.abc`, `.so`, `hnp`, `elf`) in a `.hap` is protected by **mandatory runtime code signing** (`signCode=1`).
     - During signing, `hapsigner` embeds `fs-verity` Merkle trees, CMS/PKCS#7 signatures, and `ownerId` metadata into the HAP signing block (`0x20000003` -> `0x30000001`).
     - At runtime, the kernel and dynamic linker (`ld-musl-aarch64.so.1`) verify `fs-verity` signatures and `ownerId` before allowing executable mappings (`mmap` with `PROT_EXEC`).

2. **Impact on Each Loading Mechanism from `filesDir`**:
   - **`dlopen("/data/storage/el2/base/files/libsingbox.so")`**:
     - **Blocked.** Files downloaded at runtime or imported by the user into `filesDir` lack valid `fs-verity` code-sign metadata tied to the app's `ownerId` and reside on an untrusted writable data path. Release/AppGallery builds enforce code signing (`signCode=1`) and reject unsigned `.so` loading.
   - **`OH_Ability_StartNativeChildProcessWithConfigs` / `CreateNativeChildProcessWithConfigs`**:
     - **Cannot target `filesDir`.** The API takes a library module name + symbol (e.g., `"libsingbox_child.so:Main"`), which `AppSpawn` resolves exclusively from the signed `libs/<abi>/` directory inside the installed `.hap` (`bundleCodeDir`).
   - **Standalone CLI binary in `filesDir` (`fork` + `execve`)**:
     - **Blocked.** Mobile/tablet SEPolicy forbids `execve` from writable app sandbox directories (`filesDir`), and Huawei's `executableBinaryPaths` (`bin` integration, API 24+) is restricted to **PC/2-in-1** devices and still requires bundling and signing inside the `.hap` (`binary-sign-tool`).

3. **GPLv3 Legal Note**:
   - Even if a debug/rooted environment allowed `dlopen()` from `filesDir`, **in-process `dlopen()` does NOT avoid GPLv3 copyleft**. Under FSF's GPLv3 interpretation, dynamically loading a GPLv3 `.so` plugin into the host process and calling its functions in the same virtual address space creates a single combined work.

---

## 3. External Companion App, System Extension, and IPC Alternatives

### 3.1 Separate GPLv3 Companion `.hap` + Local Proxy / VPN

Running `sing-box` in a **separate `.hap`** (distributed independently with full GPLv3 source compliance) while keeping the host browser/app in its own closed-source `.hap` provides clean legal and technical separation:

- **No binary/symbol coupling**: The host `.hap` contains zero `sing-box` code, zero `.so` dependencies on `sing-box`, and no N-API bindings to it.
- **How `ArkWeb` connects**:
  - **Via Loopback HTTP/SOCKS5 (`127.0.0.1:<port>`)**: The host app configures `webview.ProxyController.applyProxyOverride` (API 15+) to point `ArkWeb` traffic to `127.0.0.1:<port>`. If the local port is unreachable, the host app can prompt the user or launch the companion `.hap` via standard `Want` / Deep Link.
  - **Via System VPN (`VpnExtensionAbility`)**: If the companion `.hap` establishes a system VPN (with optional per-app allowlist for the host app), `ArkWeb` traffic is routed automatically without needing `ProxyController` or custom IPC.

### 3.2 Background Execution Constraints for a Companion `.hap` on HarmonyOS NEXT

If the companion `.hap` only exposes a loopback HTTP/SOCKS5 port (without `VpnExtensionAbility`), **background survival is a major hurdle** on HarmonyOS NEXT (5.x-7.0.x):

1. **`AppServiceExtensionAbility` is unavailable**: Officially restricted to **PC/2-in-1** devices and requires an enterprise-only ACL permission.
2. **Standard background suspension**: Regular `UIAbility` apps are frozen shortly after switching to the background. `backgroundTaskManager.startBackgroundRunning` only permits specific system-approved categories (e.g., audio playback, navigation, data transfer); using dummy tasks for proxy keepalive violates AppGallery review rules and risks OS termination.
3. **`VpnExtensionAbility` (`@ohos.net.vpnExtension`) is the intended background mechanism**: HarmonyOS NEXT provides `VpnExtensionAbility` specifically for network proxy/VPN apps to run a persistent background `:vpn` process (used by `NekoBox4Harmony` and `sing-box-for-harmonyos`). Even if the host app primarily uses `ProxyController`, running the companion inside a `VpnExtensionAbility` (or keeping the companion visible in tablet split-screen/floating window) is required to prevent the OS from freezing the proxy process.

### 3.3 Alternative: External LAN / Host-Device Proxy

- Running `sing-box` on a separate device (router, PC, Mac) on the same LAN and configuring `webview.ProxyController.applyProxyOverride` in `ArkWeb` to `http://<lan-ip>:<port>` avoids all HarmonyOS signing, sandbox, background-freeze, and GPLv3 issues, at the cost of requiring an external device.

---

## 4. Practical Recommendation for MatePad Mini (HarmonyOS 7.0.x)

| Requirement Scenario | Recommended Architecture | Feasibility on MatePad Mini (7.0.x) | Host App Closed-Source? |
| :--- | :--- | :--- | :--- |
| **Must use `sing-box` + closed-source host + no VPN** | Separate GPLv3 companion `.hap` + `127.0.0.1:<port>` + `ArkWeb ProxyController` | Medium (companion risks background freeze unless split-screen or `VpnExtensionAbility`) | **Yes** |
| **Must use `sing-box` + closed-source host + reliable background** | Separate GPLv3 companion `.hap` using `VpnExtensionAbility` (system VPN or per-app VPN) | **High** (proven by `NekoBox4Harmony` on MatePad Mini) | **Yes** |
| **Must use `sing-box` + single `.hap` (no companion app)** | `OH_Ability_StartNativeChildProcessWithConfigs` with separate `libsingbox_child.so` + loopback `127.0.0.1:<port>` | High (tablet supports `NativeChildProcess`), but `.so` is inside same `.hap` | **Conditional** (needs legal review) |
| **Single `.hap` + reliable background + zero GPLv3 risk** | Replace `sing-box` with **MPL-2.0 (`Xray-core`)** or **MIT (`shadowsocks-rust` / `v2ray-core`)** as signed `c-shared` `.so` in `libs/arm64-v8a/` | **High** | **Yes** (unambiguous) |

**Bottom line**: Externalizing `sing-box` as an unsigned `.so` or binary in `filesDir` is **not viable** on HarmonyOS NEXT due to mandatory `fs-verity` code signing and sandbox `W^X`/SEPolicy enforcement. For a closed-source host app on MatePad Mini (HarmonyOS 7.0.x), either:
1. Pair the host app with a **separate GPLv3 companion `.hap`** (ideally backed by `VpnExtensionAbility` for background persistence), **or**
2. Switch the in-app proxy core from `sing-box` (GPLv3) to **MPL-2.0 (`Xray-core`)** or **MIT (`v2ray-core` / `shadowsocks-rust`)** inside `libs/arm64-v8a/`.

---

## 5. References

1. [Huawei Developer Docs — App Sandbox Directory (bundleCodeDir vs filesDir)](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/app-sandbox-directory-V5/)
2. [OpenHarmony — developtools_hapsigner (Mandatory Code Signing)](https://gitee.com/openharmony/developtools_hapsigner/blob/master/README.md?skip_mobile=true)
3. [harmony-contrib/hapsigner-rs — OpenHarmony HAP & fs-verity Code-Sign Format](https://github.com/harmony-contrib/hapsigner-rs/blob/main/docs/openharmony-format.md)
4. [Huawei Developer Docs — Native Child Process Development Guide (C/C++)](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/capi-nativechildprocess-development-guideline)
5. [Huawei Developer Docs — AppServiceExtensionAbility Constraints (PC/2in1 & ACL)](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-app-ability-appserviceextensionability)
6. [Huawei Developer Docs — VPN Application Development Guide (VpnExtensionAbility)](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/net-vpnextension-V5/)
7. [Huawei Developer Docs — Integrating bin Files in HAP (PC/2in1)](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/hap-bin/)
8. [xiaoli8571/NekoBox4Harmony — MatePad Mini HarmonyOS NEXT sing-box Client](https://github.com/xiaoli8571/NekoBox4Harmony)
