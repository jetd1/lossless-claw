# Tool-Result Persistence Collapse — RCA & Rewrite Report

Production incident #194379 (2026-08-01/02). Branch: `fix/empty-content-tool-result-parts` (PR #1054).

## Incident summary

Immediately after an OpenClaw gateway restart loaded a dist built 2026-08-01 10:04
(including transient-coverage lineage), the `agent:main` Telegram session
(`agent:main:telegram:direct:276014738`) degraded within minutes:

- tool CALLS persisted; tool RESULTS vanished from the LCM SQLite record
- every assemble inserted synthetic `[lossless-claw] missing tool result ... inserted synthetic error` markers
- the model burned context confused why its own successful commands "never worked"; the session eventually gave up with "the exec tool itself is not actually executing"

The OpenClaw session **JSONL holds the full transcript**; the SQLite record lost rows.
The failing model (Kimi K3) authors its own tool-call ids in `name:N` format —
`exec:101`, `exec:102`, reused across turns and even within minutes with
**identical** `"(no output)"` content (`exec:101` alone occurred 15× in one hour).

## Root causes (one per commit)

1. **#992 (29e879d)** — empty-content tool results (`content: []`, e.g. `update_plan`)
   produced **zero** `message_parts` rows → demoted to assistant at assembly → dropped
   by the empty-content filter → `sanitizeToolUseResultPairing` inserted synthetic
   "missing tool result" errors on every assemble.
   Fix: identity-carrying fallback part + canonical empty-result coverage signature.

2. **Stable-event-key (622f8b1)** — `stable_event_key = "tool-result:<toolCallId>"` was
   treated as conversation-globally unique (partial unique index). With recurrent
   model-authored ids, every later occurrence silently short-circuited `ingestSingle` —
   the disaster path. Each recurrent `exec:N` persisted exactly once (DB verified).
   Fix: derive stable keys only from provider-unique id formats (`toolu_…`, `call_…`);
   degrade INSERT conflicts to NULL-key persist + warn.

3. **Transcript repair (2e39381, 0a03123)** — `sanitizeToolUseResultPairing` deduped
   assistant `tool_use` blocks conversation-globally by id: after the first occurrence,
   reused ids dropped as duplicates and their results as orphans.
   Fix: occurrence-scoped pairing — pending (id, payload-fingerprint) retire to a
   consumed pool when paired; boundary-stop at next surviving same-id occurrence;
   unselected delayed results stay pairable (not marked moved).

4. **Coverage dedup (2e39381, 0a03123)** — canonical empty/tool-text coverage
   signatures keyed on `toolCallId` collapsed distinct occurrences; fork-bounded live
   suffix matched an older assembled occurrence as coverage of a newer live one.
   Fix: canonical signatures restricted to provider-unique ids; recurrent ids use the
   full lossless signature. The empty fallback rehydrates as a provider-valid
   whitespace text block; the structured `details` payload persists in part metadata.

## Key insight

The LLM context layer cannot rely on application-layer tool-call ids being reliably
unique. Model-authored ids have **no uniqueness guarantee**, and id collisions are a
legitimate, common phenomenon at context scale. Every dedup path that keys on a bare
tool-call id without provenance is one model fingerprint away from eating results.

## Rewrite principles

1. Reliability: if a layer cannot determine uniqueness, it must NOT dedup; duplicates
   are recoverable, data loss is not.
2. Prefer emitting possibly-duplicated content (repair-time dedup can recover) over
   risking data loss. Fail the fast path loudly, not the data silently.
3. Never drop a semantically meaningful message. A synthetic error inserted where a
   real result existed is the worst outcome: crash → silent misinformation.
4. Any rule at ingest/dedup must store enough to repair without replaying raw logs.
5. Uniqueness-unknowable keys are strictly for correctness-irrelevant optimization.

## Reviewer / CI protocol

The PR is live under review; additive commits only. Every actionable machine-review
finding is verified against this report before fixing. All review rounds
(machine + human) ran with `npm test` (2016 tests) and `tsc` clean.
