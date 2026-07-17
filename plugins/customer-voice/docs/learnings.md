# customer-reply — Improvement Log

Append-only log of improvement runs. Most recent on top.

## 2026-07-16 — Cut ghostwriter, live-data research, flatter/faster flow

- **Trigger:** Zac wanted to (1) drop the ghostwriter dependency and make `customer-researcher` the only researcher, (2) route non-conceptual/logs questions through Horizon and other MCPs instead of docs, and (3) offer live verification (spun-up test env or WorkOS MCP operation) for code/SDK/API answers. Also standardizing on the internal claude-plugins copy over the stale GitHub-cached install.
- **Changes applied:**
  - SKILL.md: removed the broken `ghostwriter:voice-researcher` primary + fallback (that agent does not exist); `customer-voice:customer-researcher` is now the sole researcher.
  - SKILL.md Phase 2: added question classification (conceptual / runtime-logs / code-SDK-API) that routes research; voice-config prefetch now explicitly runs in parallel with research so drafting starts as soon as findings land.
  - SKILL.md: new Phase 3 "Live Verification" — for code/SDK/API questions only, offers via AskUserQuestion to test the recommendation against live code/SDK (verify/run skill or scratch project) or confirm feasibility via WorkOS MCP; phases renumbered (Draft 4, Review 5, Deliver 6).
  - customer-researcher.md: replaced the git stash/checkout/pull protocol with a read-only protocol (no branch switching or network pulls; note staleness in Caveats). Removes a major latency + repo-mutation risk.
  - customer-researcher.md: new Track 7 (Observability, Logs & Live Config) using Horizon MCP (Datadog logs, Snowflake `WORKOS_PUBLIC`, Sentry) and WorkOS MCP for live env config, with scoping/large-result guidance; added to Track Selection triage, source-priority (live data outranks docs for runtime questions), and output format (new Live/Runtime Findings section).
- **Rationale for approach:** chose targeted flattening + MCP tracks over a full Workflow() rewrite. The research fan-out already parallelizes via the agent, and the draft/verify loop is interactive, so a deterministic Workflow added machinery without a clear speed win. Biggest real gains were killing the double-nesting overhead, the git-pull protocol, and serialized voice reads.
- **Not done:** did not convert research to a Workflow() script; did not touch voice.md content or the bat-kol fallback chain.

## 2026-04-30 — Speed & Token Reduction

- **Trigger:** Zac flagged that recent runs use a ton of tokens and take a long time (driven by always-spawn researcher + parallel tracks)
- **Before:** Description 22/25, Structure 18/25, Instructions 14/25, Agents 10/25 (Total: 64/100)
- **After:** Description 22/25, Structure 19/25, Instructions 21/25, Agents 20/25 (Total: 82/100)
- **Changes applied:**
  - SKILL.md Phase 2 rewritten as "Triage & Research" with explicit skip conditions for follow-ups, voice/format-only, `--no-research`, and procedural requests
  - customer-researcher.md Track 1 (codebase) marker `ALWAYS` → `WHEN RELEVANT` with criteria
  - customer-researcher.md Track 5 (Slack history) marker `ALWAYS` → `WHEN RELEVANT` with criteria
  - customer-researcher.md launch instructions replaced with a `Track Selection` triage block + rule-of-thumb mapping per question type
- **Changes skipped:**
  - Cutting inline voice rules from SKILL.md (Zac chose to keep duplication for voice consistency)
  - Collapsing to a single voice source (kept bat-kol fallback chain)
