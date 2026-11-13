# Track B (WebRTC Voice) — Developer Under Development Configurator & P2P Lifecycle Fixes

## Branch & Verified Commit
- **Branch**: `z/agy/rc/track-b-voice-configurator` (branched from `alpha`)
- **Commit**: `a44c14dd698bcc96a8bfb0aee76347f5effc0d2f` (`signature.isValid: true`, `state: VALID`)
- **URL**: https://github.com/Antigravi-ty/RocketCompanion/commit/a44c14dd698bcc96a8bfb0aee76347f5effc0d2f

## Summary of Changes

### 1. Relocated Voice Chat from Advanced to Developer -> Under Development -> Configure
- **`AdvancedTab.tsx`**: Removed `VoiceChatSettingsSection` so Voice Chat is no longer exposed in the general Advanced settings tab.
- **`Layer3DevUnderDevelopmentRecipe.tsx`**: Added a **WebRTC Voice Chat** row (`Mic`/`MicOff` icon, default-OFF `ToggleSwitch`, and `Configure` button navigating to `dev-voice-configurator`).
- **`Layer4DevVoiceConfiguratorRecipe.tsx`**: Created the Layer 4 configurator sub-menu (`VOICE CHAT CONFIGURATOR`, panel ID `layer4-dev-voice-configurator`, parent `dev-under-development`), hosting `VoiceChatSettingsSection` with:
  - Explicit ON/OFF toggle (default OFF; microphone permission is only requested when toggled ON).
  - Mute Local Microphone + live local microphone input level meter.
  - WebRTC Input Noise Suppression (`noiseSuppression`), Echo Cancellation & Auto Gain (`echoCancellation` / `autoGainControl`).
  - Voice Output Master Gain (`0%–250%` via Web Audio `GainNode`) and Output Safety Limiter (`DynamicsCompressorNode`).
  - Per-Player Voice Mix showing connected peers (`Registered` vs `Guest` badge, player ID, live remote audio status, live remote level meter, `0%–250%` volume slider, and local mute toggle).
- **Route & Panel Registration**: Registered `dev-voice-configurator` and `layer4-dev-voice-configurator` in `uiConfig.ts`, `menuPanelSizingPresets.ts`, and `menuRouteRegistry.tsx`.

### 2. Double-Check & P2P Room Voice Fixes (Host Match / Join Room)
During code audit of `track-b-webrtc-voice`, three runtime issues were identified and fixed:
1. **Client-side peer attachment was missing**: `RuntimePostprocessingAndSessionBootstrap.ts` previously only called `attachVoiceToPeerChannel` in `onPeerConnected` (Host-only). Added `attachVoiceToPeerChannel(opts.channel, 'host', ...)` in `onJoinServer` so joining Clients also attach their `P2PWebRTCChannel` to `WebRtcNativeVoiceBridge`.
2. **Peer detachment on disconnect was not wired**: Added `onPeerDisconnected` callback to `MultiplayerSessionManager.ts` and wired `detachVoiceFromPeer` in `RuntimePostprocessingAndSessionBootstrap.ts` for peer disconnect, `onStopServer`, `onLeaveServer`, and `onUnexpectedDisconnect` (with `'*'` wildcard support in `runtimeVoiceBridge.ts`).
3. **WebRTC renegotiation offer glare**: Added `isPolite` Perfect Negotiation handling (`setLocalDescription({ type: 'rollback' })` on collision for polite peer) in `IVoiceAudioBridge.ts` and `WebRtcNativeVoiceBridge.ts` so simultaneous voice activation on Host and Client does not throw `InvalidStateError`.

## Verification
- `pnpm run format:check`: PASS
- `pnpm run check` (Radar 2.0 + `tsc --noEmit`): PASS (`1361/1361` TS files, `0` errors, `0` circular dependencies, `0` upward imports)
- `pnpm run build`: PASS

## Manual Verification & Open Items
- **2-Player Room Test**:
  1. Player A starts a room via `Play -> Host Match`; Player B joins via `Play -> Join Room`.
  2. Both players open `Settings -> Developer -> Under Development -> WebRTC Voice Chat` (toggle ON or click `Configure` to open `VOICE CHAT CONFIGURATOR`).
  3. Confirm browser microphone permission prompt appears only when toggled ON, both peers appear under `Per-Player Voice Mix` with live audio level bars, and `0%–250%` gain / mute controls work.
- **Open Items / Notes**:
  - Live 2-machine audio hardware testing was not executed inside the headless sandbox.
  - Current room topology connects Clients to Host (star topology); for >2 players in a room, Host-side audio mixing/forwarding would be needed for Clients to hear other Clients directly.
  - Built-in browser WebRTC NS/AEC/AGC is used; if stronger client-side denoising (e.g., RNNoise AudioWorklet) is needed later, it can be inserted in `WebRtcNativeVoiceBridge` without changing UI or transport layers.
