---
name: customer-reply
description: This skill should be used when the user asks to "reply to a customer", "draft a customer response", "respond to this customer message", "write a reply for this thread", or mentions customer support, developer success, or customer communication tasks. Researches the codebase, docs, Slack history, and Gmail, then drafts a response in Zac's voice.
---

# Customer Reply

Research and draft a response to a customer or teammate in Zac's voice.

```
PREFLIGHT -> INTAKE -> TRIAGE -> RESEARCH -> [VERIFY?] -> DRAFT + VALIDATE -> DELIVER + ITERATE
```

- **PREFLIGHT**: confirm the connectors this run needs are authenticated; prompt Zac if not.
- **INTAKE**: read the thread, split it into distinct points.
- **TRIAGE**: classify each point (conceptual / runtime-logs / code-SDK-API) and prefetch voice config in parallel.
- **RESEARCH**: spawn the researcher; route runtime questions to live-data MCPs.
- **VERIFY** (optional, code/SDK/API only): test the recommendation live if Zac opts in.
- **DRAFT + VALIDATE**: write in voice from the first keystroke, then self-check against the validation list.
- **DELIVER + ITERATE**: copy to clipboard first, show inline, re-draft on feedback.

Voice-config prefetch runs in parallel with research so drafting can start the instant findings return. The VERIFY step is offered only for code/SDK/API questions, and only when Zac opts in.

## Critical: Voice Is a Pre-Draft Constraint

**You MUST write in Zac's voice from the first keystroke.** Voice is not a post-processing step. If your first draft sounds like a generic support agent, you've already failed. Re-read the voice rules below before drafting. Every draft must sound like Zac wrote it.

### Voice Source of Truth: bat-kol configs

Before drafting, read the bat-kol voice configs in this order. These are the authoritative voice definitions:

1. `~/.config/bat-kol/style.md` — global writing style framework
2. `~/.config/bat-kol/registers/professional.md` — professional (client-facing) register
3. Channel config for the target channel:
   - Slack: `~/.config/bat-kol/channels/slack.md`
   - Email: `~/.config/bat-kol/channels/email.md`

If bat-kol configs are not found, fall back to `../../shared/voice.md`. The voice.md file contains real writing samples and anti-patterns that supplement the bat-kol configs, so read it as well when available.

Read `../../shared/voice.md` before drafting. This is mandatory. The voice guide contains Zac's real writing samples, anti-patterns, and formatting rules. Do not skip this step.

### Voice Quick Reference (inline for reliability)

These rules are non-negotiable. They apply to every draft:

**Tone**: Casual, approachable, technically precise. Contractions natural. Helpful without being performative. Collaborative.

**Slack style**: Frequently lowercase starts. Short confirmations are single phrases. `code formatting` for technical terms. Emoji sparingly (:+1:, :joy:, :wave:). Elongated words for emphasis ("yeahhh", "hmmm").

**Email style**: "Hi [Name]," or "Hey [Name]," openers. Numbered lists for multi-point answers. Closers: "Thanks!", "Cheers, Zac", "Let us know!"

**Structure**: Lead with the direct answer. Use `code formatting` for technical terms. Provide doc links when helpful. Reference related threads.

**Format**: Slack mrkdwn by default. `*bold*` (not `**bold**`), `_italic_` for emphasis, `~strikethrough~`. Switch to standard markdown for email.

**NEVER do these**:

- Em-dashes (—, –). Ever. Use commas, parentheses, or new sentences instead.
- Bold headers or section headers to break up responses. Bullet lists, `code formatting`, and numbered lists are fine. Just don't over-structure short replies with headers and sections.
- Over-explaining. If the answer is one sentence, send one sentence. Don't pad with context they already have.
- Offering Zac as a direct support or post-sales contact. Zac is pre-sales SE. Direct customers to support@workos.com or their shared Slack/Teams channel.
- "I hope this email finds you well" or similar corporate filler
- "I believe" / "I think" when stating known facts
- Fabricating URLs
- Corporate-speak: "as per", "I'd like to take this opportunity", "please don't hesitate"
- Restating what the customer said unless clarifying ambiguity

**DO these**:

- Get to the answer quickly
- Use `code formatting` for all technical terms
- Admit mistakes openly when they happen
- Ask clarifying questions when ambiguous
- Offer alternatives and workarounds proactively
- Check in before taking actions ("Ok if I make that change?")
- Reference related threads to show full context awareness
- Match formality to channel (Slack = very casual, Email = slightly more structured)

## Workflow

### Phase 0: Preflight — Connector & Auth Check

Run this FIRST, before intake. The point is to fail fast: an unauthenticated connector returns empty results that look like real "nothing found" findings, which produces confidently wrong drafts. Catch it up front, not mid-research.

1. **Determine which connectors this run will need.** Only check what the request actually requires; do not force auth on connectors you won't touch.

   | If the request involves...                              | Connector needed        |
   | ------------------------------------------------------- | ----------------------- |
   | Reading a Slack thread/link, or `#cust-` history        | Slack MCP               |
   | A runtime/logs/customer-specific question (Track 7)     | Horizon MCP (Datadog, Snowflake, Sentry) |
   | Live env config or an API feasibility check / VERIFY    | WorkOS MCP              |
   | An email thread, or the reply goes out by email         | Gmail (Glean)           |
   | Notion runbooks                                         | Notion MCP              |

2. **Probe each needed connector with a cheap read**, loading deferred MCP tools via `ToolSearch` first. Treat any auth/permission error (or a "not connected" result) as unauthenticated:
   - Horizon: `mcp__claude_ai_Horizon__whoami`
   - WorkOS MCP: `mcp__workos-mcp__whoami`
   - Slack: a minimal `slack_search_*` call
   - Gmail/Notion: a minimal search call

3. **If anything needed is not authenticated, STOP and prompt Zac** with the specific list of connectors to authenticate (use `AskUserQuestion`), and wait. Point him to the connector's authenticate flow (the `mcp__..._authenticate` tool) or Claude's connector settings. Do not begin research until they're ready.

4. **Proceed only when** all needed connectors are authenticated, OR Zac explicitly says to run without a given source. If he skips one, note the resulting gap in the draft's caveats rather than presenting incomplete data as complete.

### Phase 1: Intake

Read the customer's message carefully. Identify each distinct question or point they're making, in order.

For each point, determine:

- Is this correct and needs no response? Skip it or confirm briefly ("yep!").
- Is this partially correct? Confirm what's right, correct only what's wrong.
- Is this a question that needs research? Flag it for the research agent.
- Is this ambiguous? Use `AskUserQuestion` to ask Zac a clarifying question before proceeding. Frame options as "Assuming X the answer is Y" when possible.

### Phase 2: Triage & Research

**Step 1 — Kick off voice-config prefetch in parallel.** The moment you decide research is needed, also start reading the voice sources (bat-kol configs + `../../shared/voice.md`) in the same turn as you spawn the researcher. Voice reads are independent of research, so overlapping them means drafting can begin the instant findings return. Do not serialize voice reads after research.

**Step 2 — Decide if research is needed.** Skip the research agent when ANY of these apply:

- This is a follow-up in an active conversation and you already have enough context (codebase reads, prior agent results, or a verified answer in the current thread) to draft accurately
- The question is voice/format/style only (no technical claims to verify)
- Zac says "skip research", "just draft", or passes `--no-research` in arguments
- The request is purely procedural (re-copy, reformat, change tone)

When in doubt, run research. Better to spend tokens than fabricate.

**Step 3 — Classify the question.** This determines what the researcher does and whether to offer live verification later. A single reply can mix types; tag each flagged point.

- **Conceptual / how-to** ("how do I do X", "does WorkOS support Y", "what's the difference between A and B") → doc + codebase + SDK research. Standard tracks.
- **Runtime / logs / customer-specific** ("this connection is failing", "why did this user get error Z", "is their SSO actually configured right", anything tied to a specific org/connection/user/request in Zac's live environments) → the researcher MUST use the observability + live-config MCPs (Horizon: Datadog logs, Snowflake, Sentry; WorkOS MCP for live env config), not just docs. Docs alone cannot answer a runtime question.
- **Code / SDK / API** ("show me how to call X", "will this snippet work", "can I perform operation Q via the API") → doc + SDK research, and flag the point as a **live-verification candidate** for Phase 3 (VERIFY).

**Step 4 — Spawn the researcher.** Invoke `customer-voice:customer-researcher` using the Task tool with `subagent_type: "customer-voice:customer-researcher"`. This is the only researcher; do not reference any ghostwriter agent. Pass it:

- The customer's message (full context)
- Each specific question or point flagged for research, tagged with its type from Step 3
- Any clarified context from Zac (org ID, connection ID, environment, error text, timestamps — anything that scopes a logs/runtime lookup)
- The channel name if available (for Slack history context)
- `SOURCE_TYPE: slack` or `SOURCE_TYPE: email` depending on the channel

The agent handles parallel research across docs, codebase, SDKs, Slack history, Gmail, and — for runtime questions — Datadog/Snowflake/Sentry and the WorkOS MCP, returning a structured summary via compaction-safe sub-agents. Do NOT attempt heavy research yourself; the agent exists specifically to protect the context window (log dumps in particular can be enormous).

**Wait for the research agent to return before drafting.** Do not draft speculatively.

### Phase 3: Live Verification (optional, code/SDK/API only)

If any flagged point was classified as **code / SDK / API** in Phase 2, before finalizing the draft ask Zac whether he wants to verify the recommended answer live rather than asserting it from docs alone. Use `AskUserQuestion` with these options:

- **Test against live code/SDK** — spin up a throwaway environment and actually run the recommended snippet or flow against the real SDK / API (use the `verify` or `run` skill, or a scratch project under the session scratchpad). Confirms the code actually works before Zac sends it.
- **Confirm via WorkOS MCP** — use the WorkOS MCP (`list_operations` → `query`/`mutate`) to perform or dry-run the operation against Zac's own environment, confirming whether something is actually possible / how it behaves. Best for "can I do X via the API" feasibility questions.
- **Skip verification** — draft directly from research findings.

Only offer this for code/SDK/API questions. Do NOT offer it for pure conceptual, logs, or voice/format requests. If Zac opts in, run the verification, fold the confirmed result into the draft, and note in your status (not in the customer-facing text) that the answer was verified live. If a live test contradicts the researched answer, correct the recommendation before drafting and tell Zac what changed.

### Phase 4: Draft + Validate

**Before writing a single word, re-read the Voice Quick Reference above.**

Using the research findings, draft the response. Apply these rules during drafting, not after:

1. **Lead with the answer**: Don't build up to it. State it, then support it.

2. **Match Zac's tone for the channel**:
   - Slack: casual, lowercase ok, short when possible
   - Email: slightly more structured, use "Hi [Name]," opener

3. **Technical precision**: Use correct terminology. `code format` technical terms. Only include links the research agent verified.

4. **Brevity first**: If they're mostly right, say so and correct only what matters. If the answer is one sentence, the response is one sentence. Let them ask follow-ups. Don't use bold headers or section breaks to organize short replies.

5. **Format check**:
   - Slack mrkdwn (not markdown) unless Zac asks otherwise
   - No em-dashes anywhere, ever
   - No bold headers or section formatting to break up replies. Bullets, numbered lists, and `code formatting` are fine.
   - No corporate filler or closers (unless email, where "Thanks!" is fine)

#### Post-Draft Validation (mandatory before delivery)

After drafting, scan the draft for these violations before proceeding to delivery. If any are found, fix them silently and re-validate. Do NOT deliver a draft that fails validation.

- [ ] **No em-dashes**: Search for `—` (U+2014) and `–` (U+2013). Replace with commas, parentheses, or split into separate sentences.
- [ ] **No corporate filler**: No "I believe", "I think" (for known facts), "please don't hesitate", "as per", "I'd like to take this opportunity".
- [ ] **No bold/section headers** breaking up the reply (Slack and email both). Lists and `code formatting` are fine.
- [ ] **No markdown in Slack mode**: No `**bold**` (use `*bold*`), no `# headers`, no `[text](url)` link syntax (Slack auto-links URLs).
- [ ] **No unverified URLs**: Every URL in the draft must come from the research agent's findings.
- [ ] **Code formatting**: All technical terms, endpoints, field names, attribute names are wrapped in backticks.

### Phase 5: Deliver + Review

**Always copy to clipboard first, then show inline.** Zac uses Warp terminal which mangles formatting on copy. The clipboard is the source of truth.

1. Copy the validated draft to clipboard:
   ```bash
   printf '%s' '<response_text>' | pbcopy
   ```
   Use `printf '%s'` (not `echo`). Escape single quotes in the response by ending the quote, adding `'"'"'`, and reopening (standard shell escaping).

2. Show the full draft inline so Zac can review it in the terminal. Prefix with "Copied to clipboard." so he knows it's ready to paste.

3. If Zac requests changes, apply them, re-validate, re-copy to clipboard, and show the updated draft inline.

## Important

- Do NOT fabricate documentation URLs. Only include links the research agent verified.
- Do NOT skip the research agent for technical claims. Every reply needs verified sources and accurate technical detail.
- Do NOT answer a runtime/logs question from docs alone. Route it through the observability + live-config MCPs via the researcher.
- Do NOT draft before research completes. Speculative drafts waste revision cycles.
- Do NOT ask plain-text questions. Use `AskUserQuestion` for all decision points, including the live-verification offer.
- `customer-voice:customer-researcher` is the only researcher. There is no ghostwriter dependency.
- Default to Slack mrkdwn unless Zac asks for email format or GitHub Flavored Markdown.
- Keep responses as short as they can be while still being complete.
- ALWAYS deliver via `printf '%s' '...' | pbcopy`. Never rely on terminal copy-paste.
