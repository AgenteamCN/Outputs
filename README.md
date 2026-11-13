# Track B Report: Native WebRTC Voice Channel & Developer Settings Controls

- **Branch**: `z/agy/rc/track-b-webrtc-voice`
- **Verified Commit**: [`1051b1028180250de6f45ec8c3ac1e20b6cb2e65`](https://github.com/Antigravi-ty/RocketCompanion/commit/1051b1028180250de6f45ec8c3ac1e20b6cb2e65) (`signature.isValid: true`, `state: VALID`)

## 1. Implemented Architecture (Radar 2.0 Compliant)

### A. Decoupled Voice Bridge (`src/audio/voice/`)
- `IVoiceAudioBridge.ts`: Defines the narrow anti-corruption port (`IVoicePeerConnectionPort`, `IVoiceAudioBridge`, `VoiceRenegotiationSignal`, `VoicePeerAttachmentOptions`, `VoicePeerStateSnapshot`) with `0%–250%` volume clamping (`clampVoiceVolumePct`).
- `WebRtcNativeVoiceBridge.ts`: Native WebRTC voice implementation using `RTCPeerConnection.addTrack` (Opus) and Web Audio API:
  - **Default OFF & Explicit Permission Gating**: Never requests microphone permission on startup. Only calls `navigator.mediaDevices.getUserMedia(...)` when the user explicitly toggles Voice Chat ON in Developer Settings; toggling OFF stops all local tracks and removes outbound senders.
  - **Input Constraints**: Configurable `echoCancellation`, `noiseSuppression`, and `autoGainControl` (applied at `getUserMedia` and dynamically updated via `track.applyConstraints`).
  - **0%–250% Per-Player & Master Amplification**: Uses `AudioContext.createMediaStreamSource(remoteStream) -> GainNode (0.0–2.5) -> AnalyserNode -> MasterVoiceGain (0.0–2.5) -> DynamicsCompressorNode (optional Output Safety Limiter) -> AudioContext.destination` (bypassing HTML `<audio>.volume`'s 100% cap while preventing clipping). Includes the muted `<audio>` element sink workaround for Chromium remote WebRTC stream decoding.
  - **Mid-Session Renegotiation**: Supports `voice_offer` / `voice_answer` / `voice_identity` signaling over the existing DataChannel so voice can be toggled on/off without tearing down the multiplayer session.
  - **No Spatial Attenuation & No Roof Indicator**: Kept as clean, unattenuated team-comms audio per product requirements.

### B. Transport Isolation (`src/network/transport/`)
- `dataChannelFramingPipeline.ts`, `DataChannelTransport.ts`, and `P2PWebRTCChannel.ts`: Added `voice_signal` control-frame parsing, `sendVoiceSignal`, `onVoiceSignal`, and `playerId` / `isRegisteredPlayerId` metadata fields.
- Zero imports from `src/audio/` inside `src/network/transport/` (0 Radar transport isolation violations).

### C. Registered Player ID Flag & Cloud Profile Sync (`src/ui-core/`)
- `voiceChatConfig.ts` & `voiceChatStore.ts`:
  - Distinguishes **Registered** player IDs (`cloudProfileSyncStore.loggedInUser.userId`, `isRegistered: true`) from **Guest/Ephemeral** peer IDs (`isRegistered: false`).
  - Per-player volume (`0%–250%`) and mute states for `isRegistered === true` players are persisted under `rc_voice_chat_prefs_v1` and synced via `cloudProfileSnapshotCollector.ts` (`SYNCABLE_EXTRA_STORAGE_KEYS`).
  - Ephemeral guest peer IDs are kept in session-only memory (`sessionPeerVolumes` / `sessionPeerMuted`) so random `peer_...` IDs do not pollute cloud profiles.
  - `enabled` is intentionally never persisted as `true` so microphone permission is always explicitly requested by user action.

### D. Developer Settings UI & Runtime Bridge (`src/ui-core/recipes/settings/`, `src/ui-system/core/`, `src/game/`)
- `VoiceChatSettingsSection.tsx` mounted in `AdvancedTab.tsx` (Developer Settings):
  - Master Voice Chat toggle (Default OFF, shows browser permission state)
  - Mute Local Mic toggle + live Microphone Input Level bar
  - Input Noise Suppression (WebRTC NS) & Echo Cancellation / Auto Gain (AEC / AGC) toggles
  - Voice Output Master Gain slider (`0%–250%`) + Output Safety Limiter toggle
  - Per-Player Voice Mix list with `Registered` vs `Guest` Player ID indicator, Mute button, and `0%–250%` volume slider
- `runtimeVoiceBridge.ts` + `runtimeEngineUiPort.ts` + `runtimeBridgeAdapter.ts` + `runtimeUiPort.ts` + `RuntimePostprocessingAndSessionBootstrap.ts`: Wires peer connection attach/detach and local `cloudProfileSyncStore` identity exchange on `onPeerConnected`.

## 2. Output Denoising Evaluation (Industry Options)
- **Input Denoising**: Handled natively by WebRTC (`noiseSuppression: true`, `echoCancellation: true`, `autoGainControl: true`) in Chromium/Tauri WebView with zero bundle overhead.
- **Output Denoising (Remote Peer Audio)**:
  - **Current**: Web Audio `DynamicsCompressorNode` safety limiter prevents distortion/clipping when boosting quiet teammates up to `250%`.
  - **Recommended Library if Neural Output Denoising is Needed Later**: Instead of building custom DSP/spectral subtraction, integrate **`@sapphi-red/web-noise-suppressor`** (AudioWorklet + RNNoise/Speex WASM, MIT license, ~50–100KB WASM) or **`@jitsi/rnnoise-wasm`** into the remote peer `AudioWorkletNode` slot between `MediaStreamAudioSourceNode` and `GainNode`.

## 3. Verification Results
- `pnpm run format:check` / `npx prettier`: **PASS**
- `pnpm run check` (`scripts/radar.ts` + `tsc --noEmit`): **PASS** (1359/1359 TS files, 0 Circular Deps, 0 Boundary Violations, 0 TS Errors)
- `pnpm run build` (Vite production bundle): **PASS** (`dist/` built in 28.19s)
