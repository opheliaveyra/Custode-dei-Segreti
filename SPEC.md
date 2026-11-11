# SPEC — Reference design for an agent credential vault

For platform/tool builders who want the real thing, not just agent behavior
rules. A markdown policy cannot give a model secure storage; this is the
minimum architecture that can.

## Components

1. **Vault service** — encrypted-at-rest credential store, outside the model's
   context, transcript, memory, and workspace. The model process has no API
   that returns a secret value.
2. **Secure entry UI** — a form rendered by the platform (not by the model)
   where the user types the secret. Fields: origin, label, value, and a
   "save" checkbox (checked = persist in Vault; unchecked = one-time use).
   The submission returns only a receipt: `saved | used_once | failed`.
3. **Filler** — a browser/tool component that receives the value from the
   Vault at fill time, after the approval gate, and types it into the target
   origin. The value transits Vault → Filler only; it never passes through
   model tokens.
4. **Approval gate** — every release is scoped to (origin, task, credential)
   and requires explicit user approval, except standing grants the user has
   knowingly configured per origin. Grants come in grains: **once**,
   **time-boxed** (e.g. 1 hour), or **standing for a named scope** — and
   high-risk targets (production systems, payment, account/security
   settings) are excluded from standing grants: they always need a live
   approval. The request shown to the user must include the agent's stated
   reason, labelled as the agent's claim, not as a verified fact.
   Approvals are auditable and revocable.
4a. **Executor (preferred over release)** — where feasible, the Vault does
   not hand a value to any filler at all: it performs the operation itself
   (fill this form, sign this request, run this authenticated call) and
   returns only the outcome. The API surface contains **no reveal/export
   operation** — that absence is structural, not a policy promise
   (Sallyport pattern).
5. **Reference issuer** — for one-time codes and transient secrets, issues
   opaque references (`[credential:<uuid>]`) backed by an in-memory entry
   with a short TTL (reference: ~10 minutes) and single-purpose binding to
   one origin/step. The model relays the reference, never the value.

## Data model

See `credential-reference.schema.json`. Metadata only: `id`, `origin`,
`label`, `type` (`password | api_key | otp | oauth | passkey`), `created_at`,
`last_used_at`. The schema deliberately has no field that can hold a value;
`additionalProperties: false` so one can't be smuggled in.

## Flows

### Save
User → Secure entry UI → Vault. Model receives receipt + reference metadata.

### Use
Model requests sign-in for origin X → platform checks Vault → approval gate
(user approves) → Filler fetches value from Vault → Filler types it →
model receives only the outcome (`signed_in | rejected | needs_code`).

### One-time code
Code arrives via a connected channel → reference issuer stores it in memory
(TTL ~10 min) → model passes reference + expected origin to the waiting step
→ user approves sharing → Filler receives the value and types it → entry is
destroyed.

### Delete
User-only, in settings UI. The model can link to settings; it cannot call delete.

## Threat model / non-goals

- Protects against: secrets leaking via transcripts, memory files, prompt
  logs, context compaction, exported chats, and the model itself being
  prompt-injected into reciting a password (it never has it).
- Does not protect against: a compromised Vault service or filler, a user
  approving a release to a phishing origin (mitigate: bind approvals to the
  exact HTTPS origin and show it on the approval UI), or secrets the user
  pastes into chat anyway (mitigate: agent rules in `AGENTS.md`).
- Audit logs record metadata (when, which origin, which credential id, what
  outcome) — never values, never code contents.

## Lessons folded in from prior art (see PRIOR-ART.md)

- **Second injection point — placeholder substitution.** For API calls, the
  agent holds only a dummy placeholder (e.g. `__github_pat__`); an egress proxy
  swaps in the real value after the request leaves the agent's process
  (Infisical Agent Vault pattern).
- **Origin comes from the page/request, not from the agent.** The filler reads
  the actual target origin itself, so a manipulated agent cannot redirect a
  fill to a look-alike site (agent-vault-cli pattern).
- **Exact host match; no wildcards.** Bindings that would rely on wildcard
  hosts are rejected at creation time, not stored as false permissiveness
  (1Claw pattern). Requests matching no rule are denied by default.
- **Write-only values.** After a credential is saved, no API — including the
  owner's own tooling surface exposed to agents — returns its value.
- **Never inject into a URL query string.** Header/body injection only; URLs
  land in logs (Managed Agents vaults pattern).
- **Sessions are time-bound and revocable**, with a stated revocation lag;
  approval for one session grants nothing to the next (1Password for Agents).
- **Delegation only narrows.** An agent handing work to another agent may pass
  a narrower scope, never a wider one.
- **Ship a plaintext-residue audit.** A `secrets audit --check`-style scan of
  config/memory/generated files is part of conformance, because a reference
  system is bypassed by any plaintext copy left where the agent can read it
  (OpenClaw SecretRef caveat).
- **Run the vault on a different trust domain from the agent** (separate host
  or sealed sidecar). Same-host deployment voids the guarantee and must be
  documented as such.
- **Approval grains and labelled reasons** (keypaste pattern): deny / once /
  time-boxed / standing-for-scope; the agent's reason is displayed as its
  claim; prod-class targets always require a live answer.
- **Kill switch with revocation propagation** (Caracal, agent-secrets
  patterns): revoking an agent or credential invalidates everything derived
  from it — active sessions, leases, and delegations — in one action, with a
  stated propagation bound. Rotation hooks run on kill where configured.
- **Tamper-evident audit**: append-only log, hash-chained (and optionally
  signed by a platform key/enclave), so later edits are detectable.
- **Config files reject secrets structurally** (sigcli pattern): validators
  refuse secret-looking fields in plain config; secrets enter only via the
  dedicated intake command/UI, never by editing a file.
- **Credential-type preference**: OAuth / passkey / reusable browser session
  first, stored password last (sigcli pattern) — the best password is the
  one the Vault never has to hold.
- **Explicit non-pattern — leases that return raw values.** Some tools lease
  a secret's value to the agent for a TTL (agent-secrets pattern). That may
  be acceptable for short-lived API tokens between machines; it does **not**
  conform here for passwords, TOTP seeds, or recovery codes, which must
  never take value form in the agent at any TTL.

## Conformance checklist

- [ ] Model cannot retrieve a secret value through any tool
- [ ] Secure entry is rendered by the platform, not the model
- [ ] Save is opt-in per credential; one-time use leaves nothing stored
- [ ] Release requires approval scoped to origin + task
- [ ] OTP references expire in minutes and are single-origin
- [ ] Deletion is user-only via settings
- [ ] Nothing secret in transcripts, memory, logs, code, or URLs
- [ ] Filler verifies origin from the page/request itself; exact host match, no wildcards
- [ ] Saved values are write-only; no tool returns a secret value to any agent
- [ ] Sessions expire, revocation takes effect within a stated bound, unmatched hosts are denied
- [ ] Delegation to other agents can only narrow scope
- [ ] A plaintext-residue audit over config/memory/generated files passes
- [ ] Approval grains exist (once / time-boxed / standing-for-scope); high-risk targets always need live approval; requests show the agent's reason labelled as a claim
- [ ] No reveal/export operation exists in the agent-facing API
- [ ] Kill switch revokes an agent/credential and everything derived from it within a stated bound
- [ ] Audit log is append-only and tamper-evident (hash chain or signatures)
- [ ] Config validators structurally reject secret-valued fields
- [ ] Password/TOTP-seed/recovery credentials are never leased back as values, at any TTL
