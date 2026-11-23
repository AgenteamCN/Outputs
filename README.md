# Architecture Proposal: Sub-Agent Token Proxy, Cascading GitHub Revocation & Irreversible Sub-Channel Isolation

- **Target Repo**: `Antigravi-ty/agyWorker` (`workers/agentlivechatchannel`) + `getgithubapptoken` Worker
- **Status**: Design Proposal & Open Questions (for review)

---

## 1. Problem Statement

1. **Over-privileged GitHub Token Issuance**:
   - Currently, sub-agents receive Chief's admin `x-agy-request-id` in their Initial Input solely to call `https://getgithubapptoken.maboco-mark.workers.dev/`.
   - If a sub-agent is compromised, misbehaves, or is revoked via `POST /agents/revoke`, its 1-hour `ghs_...` GitHub App installation token remains valid and untraceable.
2. **Sub-Channel Collision & Predictability**:
   - Reusing static channel names like `ch_code_a2_worker1` across sessions/days risks collisions with stale/zombie agents waking up late.
   - Using predictable prefixes derived directly from the Chief <-> Human channel without a secret/salt could risk leaking or guessing the parent channel.

---

## 2. Proposed Architecture

### A. Worker Proxy for GitHub App Tokens + DO Tracking + Cascading Revocation
1. **New Endpoint on `agentlivechatchannel`**: `POST /github/token`
   - **Auth**: Accepts any valid, non-revoked `x-agy-request-id` (`agy_sub_...` or admin).
   - **Flow**:
     1. Worker validates `identity` via DO `/auth` (rejects `401` if revoked).
     2. Worker calls `getgithubapptoken.maboco-mark.workers.dev/` using a Worker-only secret (`env.GITHUB_TOKEN_GATEWAY_SECRET` or `env.ADMIN_REQUEST_ID`) — **sub-agents never see Chief's admin ID**.
     3. Worker records `(id, agent_id, sub_token, ghs_token, expires_at, revoked)` into a new DO SQLite table `github_tokens` (expires_at = `now + 3600s` or parsed from GitHub response).
     4. Worker returns `{ ok: true, token: ghs_token, expiresAt }` (or raw text if `?raw=1`).
2. **Cascading Revocation on `POST /agents/revoke`**:
   - When an admin revokes a sub-agent (`token` or `agentId`):
     1. Mark the agent `revoked = 1` in `agents`.
     2. Query all non-revoked, non-expired `ghs_token` rows for that `token` / `agent_id` in `github_tokens`.
     3. For each active `ghs_token`, call:
        ```http
        DELETE https://api.github.com/installation/token
        Authorization: Bearer <ghs_token>
        User-Agent: agyWorker
        ```
     4. Mark those rows `revoked = 1` and return `{ ok: true, revokedGithubTokens: count }`.

### B. Irreversible Hash-Derived Sub-Channel IDs
- **Formula**:
  ```
  channel = "ch_sub_" + SHA256(chiefChannel + ":" + agentId + ":" + randomSalt128bit).slice(0, 10)
  ```
- **Properties**:
  1. **One-way (Irreversible)**: Even if a sub-agent knows `ch_sub_<10hex>` and its own `agentId`, it cannot reverse-engineer `chiefChannel` (`ch_dc28ee43`) without `chiefChannel` and `randomSalt128bit`.
  2. **Zero Cross-Session Collision**: Every session / agent spawn uses a fresh `randomSalt128bit`, guaranteeing zero overlap with yesterday's or zombie agents' channels.
  3. **DO Enforcement**: `agents.allowedChannels` strictly binds the sub-agent to `[ch_sub_<10hex>]`, so it can never subscribe or publish to `ch_dc28ee43`.

---

## 3. Open Questions & Edge Cases to Confirm (?)

1. **[?] `getgithubapptoken` Worker Auth**:
   - Currently `getgithubapptoken` authenticates via `x-agy-request-id`. Should `agentlivechatchannel` hold Chief's `x-agy-request-id` as a Wrangler Secret, OR should `getgithubapptoken` accept a dedicated Worker-to-Worker shared secret (`X-Worker-Proxy-Secret`) so Chief's ID isn't overloaded?
2. **[?] GitHub Token Repository & Permission Scoping**:
   - Does `getgithubapptoken` currently support passing `?repositories=agyDevKit` or `permissions` in the request body? If yes, DO `agents` table could also store `allowedRepos` so a sub-agent working on `agyDevKit` cannot even touch `agyWorker` or `Outputs`!
3. **[?] Token Caching vs. Fresh Issue per Call**:
   - Should `POST /github/token` issue a fresh `ghs_...` every time (simpler, but more rows to `DELETE` on revoke), or reuse the latest cached `ghs_...` for that `agent_id` if remaining TTL > 10 minutes (fewer GitHub API calls + faster revocation)?
4. **[?] `DELETE /installation/token` Timeout & Error Handling**:
   - During `POST /agents/revoke`, if GitHub API is slow or a token already expired (`401`), the Worker should treat `401`/`404` as already-revoked and use `Promise.allSettled` with a short timeout (e.g. 3s) so `/agents/revoke` never blocks.
5. **[?] Storing `ghs_...` in DO SQLite**:
   - Should `github_tokens` store the `ghs_...` plaintext (needed to call `DELETE /installation/token`) encrypted with a Worker AES-GCM key (`env.TOKEN_ENC_KEY`) or is DO's isolated per-account storage sufficient since it's never returned to list endpoints?
6. **[?] Should Sub-Agent Branch Restrictions Be Enforced at GitHub Proxy Level?**:
   - Currently `allowedBranches` (`["a2-*"]`) is declarative/advisory in `agyWorker`. Note that GitHub App installation tokens cannot restrict by branch name natively, so branch isolation still relies on Chief review + GitHub branch protection rules on `a2`/`main`.
