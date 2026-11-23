# Sub-Agent (`atlas`) Startup Friction Report & Initial Input Optimizations

- **Agent**: `atlas` (`specialist`, channel `ch_sub_3458d88135`)
- **Timestamp**: 2026-11-23T14:37Z – 14:40Z

---

## 1. Two Startup Hiccups Observed

1. **Unauthenticated `git clone` on Private Repo**:
   - `atlas` initially ran `git clone https://github.com/Antigravi-ty/agyDevKit.git` without fetching the GitHub installation token first, hitting:
     `fatal: could not read Username for 'https://github.com': terminal prompts disabled`
2. **Parsing Plain-Text `ghs_...` Token with `jq`**:
   - `atlas` piped the output of `https://getgithubapptoken.maboco-mark.workers.dev/` into `jq`, hitting:
     `jq: parse error: Invalid numeric literal at EOF`
   - Reason: the Initial Input did not explicitly state that the token gateway returns a **raw plain-text `ghs_...` string** (not JSON).
3. **Context Overhead from `GET /channel/state`**:
   - `GET /channel/state` returned a full list of historical `done` tasks, adding unnecessary input tokens at startup even though tasks are assigned directly via chat (`ch_sub_*`).

---

## 2. Recommended Initial Input Fixes

1. Explicitly include the ready-to-run authenticated clone snippet:
   ```bash
   TOKEN=$(curl -s -H "x-agy-request-id: ..." "https://getgithubapptoken.maboco-mark.workers.dev/")
   # Outputs raw plain-text ghs_... directly (NOT JSON, do not pipe to jq)
   git clone --branch a2 "https://x-access-token:${TOKEN}@github.com/Antigravi-ty/agyDevKit.git" /workspace/agyPreviewInstant
   ```
2. Make `GET /channel/state` optional (or omit it when tasks are dispatched directly via chat) to keep startup context lean.
