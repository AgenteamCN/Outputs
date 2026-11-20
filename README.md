# Google AI Studio LLM Workflow Audit & PocketDeck LLM Control Upgrade

- **Channel**: `ch_55ec1572`
- **Base Commit**: `78ae3820097912c0520f396fb14eff421fa7db70`
- **Branch**: `feat/llm-ax-regex-and-parallel-approvals`
- **Verified Bot Commits**:
  - `85864021fb6440f2c7e628c05fbd1494fd7c73d9` — `Add targetRole AX filter for regex clicks and prioritize mobile pending approvals`
  - `e6be5099e74d5273398f4612070332c791630db9` — `Upgrade PocketDeck LLM Control UI, manual layout mode, and Bridge order`

## 1. Google AI Studio AX & Workflow Findings (Window `47694`)

1. **API Key Unlink Flow (Double-Checked)**:
   - Linked state: `AXPopUpButton` with title `API key selected`.
   - Unlink sequence: `AXPress` (`targetRole: "AXPopUpButton"`, `targetRegex: "^API key selected$"`) -> opens `API key information` modal -> `AXPress` (`targetRole: "AXButton"`, `targetRegex: "^Unlink$"`) -> `AXPopUpButton` title updates to `No API key selected`.
2. **3+2 Model Matrix & URL Force-Init**:
   - URL format: `https://aistudio.google.com/prompts/new_chat?model=<model_id>`
   - **Gemini (3)**:
     - `gemini-4-hydrogen` (`Gemini 4 Hydrogen`): Thinking default `High`; has `URL context`; no Maps; no Temp/Top P sliders.
     - `gemini-3.8-flash` (`Gemini 3.8 Flash`): Thinking default `Medium`; has `Grounding with Google Maps` + `URL context`.
     - `gemini-3.5-flash-lite` (`Gemini 3.5 Flash Lite`): Thinking default `Minimal`; has `Grounding with Google Maps` + `URL context`.
   - **Gemma (2)**:
     - `gemma-4-26b-a4b-it` (`Gemma 4 26B A4B IT`): Thinking default `High` (`Minimal` / `High`); has `Temperature` (`1`) and `Top P` (`0.95`) sliders; no `URL context` / Maps.
     - `gemma-4-31b-it` (`Gemma 4 31B IT`): Same settings profile as `gemma-4-26b-a4b-it`.
3. **Tools & System Instructions**:
   - Tools checkboxes (`Code execution`, `Grounding with Google Search`, `Browse the url context`, `Grounding with Google Maps`, `Function calling`) report live state via `AXCheckBox` `value` (`"1"` = ON, `"0"` = OFF) and toggle cleanly with `targetRole: "AXCheckBox"`.
   - `System instructions` panel stores presets in browser `localStorage` (`+ Create new instruction`, `AXTextField`, `AXTextArea 'System instructions'`, `AXButton 'Delete system instruction'`, `AXButton 'Close panel'`). Controller-managed local presets injected directly into `AXTextArea 'System instructions'` avoid duplicate `Untitled` entries and browser storage eviction.
4. **Window Resize & AX Path Instability**:
   - Narrowing the Safari window collapses the right-hand Run settings panel (`totalElements` drops from `106` to `53`) behind `AXButton 'Toggle run settings panel'`.
   - Unlinking API key or toggling panels shifts top-level AX paths, and case-insensitive `targetRegex` without `targetRole` can hit `AXHeading` (e.g., `Thinking level`) before `AXComboBox` (`Thinking Level`). Filtering by `targetRole` + `targetRegex` resolves this deterministically.

## 2. Implemented Changes (`apps/p2p-remote-controller`)

- **AX `targetRole` Filtering**:
  - Added optional `targetRole` to `RemoteCommandRequest` (`src/types/emqxBridge.ts`), `PocketDeckRemoteAction` (`src/types/lanSharing.ts`), `parseEmqxInboundMessage` (`src/utils/emqxMessageParser.ts`), `handlePerformActionCommand` (`src/services/emqxCommandHandlers.ts`), `usePocketDeckBridge` (`src/hooks/usePocketDeckBridge.ts`), and `EmqxPendingApprovalsSection.tsx`.
- **Manual Viewport Layout Mode (`Auto` / `Mobile` / `Pad` / `Desktop`)**:
  - Added `PocketLayoutMode` persisted in `localStorage` (`agy_pocket_layout_mode`) with selector in `PocketDeckHeader.tsx` and grid/width adaptation in `PocketDeckView.tsx` and `PocketDeckLlmControlTab.tsx`.
- **Mobile Bridge Parallel Approvals**:
  - Updated `PocketDeckBridgeTab.tsx` to keep `Pending Reply (N)` below `Agent Chat` (matching bottom-up mobile thumb UX) while displaying parallel request counts and `windowId` / `targetRole` / `targetRegex` / `action` / `level` metadata.
- **Upgraded `PocketDeckLlmControlTab.tsx`**:
  - Added 2-step `Unlink API Key` + rescan, collapsed Run settings auto-reveal (`Toggle run settings panel`), 3+2 model cards with `Force URL` init, live `AXCheckBox` Tools toggles, `Stream` / `Final` mode toggle, focus-safe prompt injection, and a collapsible Advanced section (`Thinking Level` quick actions, `System Instruction` local presets, and `targetRole` + `targetRegex` custom trigger).

## 3. Verification

- `pnpm --filter p2p-remote-controller check` (`tsc --noEmit && pnpm lint`): passed.
- `pnpm --filter p2p-remote-controller test`: 10 test files, 37 tests passed.
