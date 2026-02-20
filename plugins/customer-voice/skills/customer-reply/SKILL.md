---
name: customer-reply
description: This skill should be used when the user asks to "reply to a customer", "draft a customer response", "respond to this customer message", "write a reply for this thread", or mentions customer support, developer success, or customer communication tasks. Researches the codebase, docs, Slack history, and Gmail, then drafts a response in Zac's voice.
---

# Customer Reply

Research and draft a response to a customer or teammate in Zac's voice.

```
INTAKE -> RESEARCH -> DRAFT + VALIDATE -> DELIVER (clipboard) -> ITERATE
  |          |              |                    |                   |
 Read     Spawn         Apply voice,         pbcopy first,      Re-draft
 thread   researcher    then self-check      show inline         on feedback
```

## Critical: Voice Is a Pre-Draft Constraint

**You MUST write in Zac's voice from the first keystroke.** Voice is not a post-processing step. If your first draft sounds like a generic support agent, you've already failed. Re-read the voice rules below before drafting. Every draft must sound like Zac wrote it.

Read `../../shared/voice.md` before drafting. This is mandatory. The voice guide contains Zac's real writing samples, anti-patterns, and formatting rules. Do not skip this step.

### Voice Quick Reference (inline for reliability)

These rules are non-negotiable. They apply to every draft:

**Tone**: Casual, approachable, technically precise. Contractions natural. Helpful without being performative. Collaborative.

**Slack style**: Frequently lowercase starts. Short confirmations are single phrases. `code formatting` for technical terms. Emoji sparingly (:+1:, :joy:, :wave:). Elongated words for emphasis ("yeahhh", "hmmm").

**Email style**: "Hi [Name]," or "Hey [Name]," openers. Numbered lists for multi-point answers. Closers: "Thanks!", "Cheers, Zac", "Let us know!"

**Structure**: Lead with the direct answer. Use `code formatting` for technical terms. Provide doc links when helpful. Reference related threads.

**Format**: Slack mrkdwn by default. `*bold*` (not `**bold**`), `_italic_` for emphasis, `~strikethrough~`. Switch to standard markdown for email.

**NEVER do these**:

- Em-dashes (—, –). Use commas, parentheses, or new sentences instead.
- "I hope this email finds you well" or similar corporate filler
- "I believe" / "I think" when stating known facts
- Over-explaining things the customer clearly already understands
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

### Phase 1: Intake

Read the customer's message carefully. Identify each distinct question or point they're making, in order.

For each point, determine:

- Is this correct and needs no response? Skip it or confirm briefly ("yep!").
- Is this partially correct? Confirm what's right, correct only what's wrong.
- Is this a question that needs research? Flag it for the research agent.
- Is this ambiguous? Use `AskUserQuestion` to ask Zac a clarifying question before proceeding. Frame options as "Assuming X the answer is Y" when possible.

### Phase 2: Research

**Always invoke the `customer-researcher` agent** using the Task tool with `subagent_type: "customer-voice:customer-researcher"`. Pass it:

- The customer's message (full context)
- Each specific question or point flagged for research
- Any clarified context from Zac
- The channel name if available (for Slack history context)

The agent handles parallel research across codebase, docs, SDKs, Slack history, and Gmail with compaction-safe sub-agents. Do NOT attempt research yourself; the agent exists specifically to protect context window during heavy research.

**Wait for the research agent to return before drafting.** Do not draft speculatively.

### Phase 3: Draft + Validate

**Before writing a single word, re-read the Voice Quick Reference above.**

Using the research findings, draft the response. Apply these rules during drafting, not after:

1. **Lead with the answer**: Don't build up to it. State it, then support it.

2. **Match Zac's tone for the channel**:
   - Slack: casual, lowercase ok, short when possible
   - Email: slightly more structured, use "Hi [Name]," opener

3. **Technical precision**: Use correct terminology. `code format` technical terms. Only include links the research agent verified.

4. **Brevity first**: If they're mostly right, say so and correct only what matters. If the answer is one sentence, the response is one sentence. Let them ask follow-ups.

5. **Format check**:
   - Slack mrkdwn (not markdown) unless Zac asks otherwise
   - No em-dashes anywhere
   - No corporate filler or closers (unless email, where "Thanks!" is fine)

#### Post-Draft Validation (mandatory before delivery)

After drafting, scan the draft for these violations before proceeding to delivery. If any are found, fix them silently and re-validate. Do NOT deliver a draft that fails validation.

- [ ] **No em-dashes**: Search for `—` (U+2014) and `–` (U+2013). Replace with commas, parentheses, or split into separate sentences.
- [ ] **No corporate filler**: No "I believe", "I think" (for known facts), "please don't hesitate", "as per", "I'd like to take this opportunity".
- [ ] **No markdown in Slack mode**: No `**bold**` (use `*bold*`), no `# headers`, no `[text](url)` link syntax (Slack auto-links URLs).
- [ ] **No unverified URLs**: Every URL in the draft must come from the research agent's findings.
- [ ] **Code formatting**: All technical terms, endpoints, field names, attribute names are wrapped in backticks.

### Phase 4: Deliver + Review

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
- Do NOT skip the research agent. Every reply needs verified sources and accurate technical detail.
- Do NOT draft before research completes. Speculative drafts waste revision cycles.
- Do NOT ask plain-text questions. Use `AskUserQuestion` for all decision points.
- Default to Slack mrkdwn unless Zac asks for email format or GitHub Flavored Markdown.
- Keep responses as short as they can be while still being complete.
- ALWAYS deliver via `printf '%s' '...' | pbcopy`. Never rely on terminal copy-paste.
