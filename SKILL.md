---
name: fact-checker
description: >
  Guides an LLM agent to evaluate whether a user's statements are factually accurate,
  internally consistent, and appropriately confident. Use this skill whenever a user makes
  a specific factual claim, cites statistics or research, describes technical processes,
  recounts historical events, or asserts causal relationships — even casually embedded in
  a longer message. Also triggers when the agent needs to decide how much to trust user-
  provided context before acting on it (e.g. in RAG pipelines, customer support, travel
  advisory, or medical triage). Always apply this skill when the downstream action depends
  on the truth of what the user said, not just their intent.
---

# Fact-Checker Skill

Guides an LLM to assess the factual reliability of user statements and produce a
structured `FactCheckResult` that downstream agents can act on — deciding whether to
accept, question, correct, or escalate a claim.

This skill does **not** replace web search. It provides a reasoning framework for
evaluating claims using (a) the model's parametric knowledge, (b) internal consistency
checks, and (c) signals from the conversation itself. For time-sensitive or highly
specific claims, combine with a web search tool.

---

## When to apply this skill

- User makes a specific factual assertion ("X costs Y", "Z happened in year N")
- User cites statistics, research findings, or named sources
- User describes a causal chain or technical mechanism
- User's claim is the premise for a downstream action (booking, recommendation, diagnosis)
- A claim seems plausible but subtly off — partial truths are the hardest case
- Conversational context contains internal contradictions

Do **not** apply to:
- Pure opinion statements ("I prefer X to Y")
- Hypotheticals and thought experiments
- Claims the user explicitly frames as uncertain ("I think maybe…")

---

## Core output schema

Produce a `FactCheckResult` JSON object. Omit fields that genuinely don't apply.
Never fabricate a source citation — use `null` if unknown.

```json
{
  "claim": "verbatim or minimally paraphrased user statement being evaluated",
  "verdict": "true | mostly_true | uncertain | mostly_false | false | unverifiable",
  "confidence": "high | medium | low",
  "reasoning": "2-4 sentences explaining the verdict",
  "error_type": null,
  "corrections": [],
  "sources_to_check": [],
  "follow_up_questions": [],
  "recommended_action": "accept | clarify | correct | escalate | search"
}
```

### Field definitions

**verdict**
- `true` — claim is accurate to the best of available knowledge
- `mostly_true` — core claim is right but contains minor inaccuracies or misleading framing
- `uncertain` — insufficient information to verify; plausible but not confirmable
- `mostly_false` — claim contains a significant factual error but is not entirely wrong
- `false` — claim is clearly incorrect
- `unverifiable` — claim is about private, future, or inherently unprovable matters

**error_type** (populate when verdict is mostly_false or false)
- `factual_error` — wrong date, number, name, or attribution
- `conflation` — two separate things merged into one
- `causal_error` — correlation treated as causation, or wrong causal direction
- `anachronism` — correct fact applied to wrong time period
- `magnitude_error` — order-of-magnitude off (e.g. millions vs billions)
- `partial_truth` — selectively true; misleading by omission
- `hallucinated_source` — cited source does not exist or did not say this

**recommended_action**
- `accept` — proceed treating claim as true
- `clarify` — ask the user for more detail before deciding
- `correct` — politely provide the accurate information
- `escalate` — claim has safety/legal/medical implications; defer to expert
- `search` — claim requires real-time or highly specific verification

---

## Reasoning process

Follow these steps in order. Do not skip steps for "obvious" cases — the most dangerous
errors are the ones that feel obvious.

### Step 1 — Parse the claim

Extract the atomic assertions embedded in the user's statement. A single sentence
often contains multiple claims.

> "Wechat Pay requires a Chinese bank card and foreigners can't use it"

Breaks into:
1. WeChat Pay requires a Chinese bank card (to link)
2. Foreigners cannot use WeChat Pay

Evaluate each atomic claim separately before combining into a verdict.

### Step 2 — Classify claim type

| Type | Examples | Key risk |
|------|----------|----------|
| **Empirical fact** | dates, quantities, names | Outdated info, magnitude errors |
| **Technical process** | "how X works" | Oversimplification, version mismatch |
| **Statistical** | percentages, studies | Cherry-picking, base rate confusion |
| **Causal** | "X causes Y" | Correlation/causation, confounders |
| **Policy/regulatory** | laws, rules, pricing | Jurisdiction mismatch, recency |
| **Geographic/cultural** | local practices, norms | Regional variation, stereotyping |

### Step 3 — Check internal consistency

Before reaching for external knowledge, check whether the claim is consistent with:
- Other things the user has said in this conversation
- Basic logical constraints (quantities that don't add up, timelines that don't fit)
- Domain priors (a claim that violates a well-established principle is suspect)

Flag any internal contradiction as a `consistency_warning` in your reasoning.

### Step 4 — Apply parametric knowledge

Evaluate the claim against what the model knows, with explicit uncertainty tracking:

- State what you know confidently
- State where your knowledge might be outdated (training cutoff risk)
- State where you genuinely don't know

**High-risk zones for outdated knowledge:**
- Pricing, fees, exchange rates
- App features and policies (especially Chinese super-apps)
- Regulatory rules (visa policies, data laws)
- Company structures, leadership
- Medical guidelines updated post-2024

For any claim in these zones, set `recommended_action: search` regardless of apparent confidence.

### Step 5 — Assess source quality (if user cited one)

If the user cited a source, evaluate:

| Signal | Weight |
|--------|--------|
| Named primary source (government site, peer-reviewed paper) | Strong positive |
| Named reputable secondary source (major newspaper, known org) | Moderate positive |
| Unnamed "studies show" / "experts say" | Weak — request specifics |
| Social media, forum, anecdote | Negative — verify independently |
| Source cannot be found or attributed | Flag as possible `hallucinated_source` |

Do not validate a claim just because a source was cited. Check whether the source
plausibly exists and would actually say what the user claims.

### Step 6 — Produce verdict and recommended action

Map your findings to the verdict scale. When in doubt, choose the more cautious verdict.
A `mostly_true` with a clear correction is more useful than a `true` that lets an error pass.

Choose `recommended_action` based on:
- Stakes of the downstream action
- How fixable the error is
- Whether a correction would help or would derail the conversation

---

## Output modes

Choose the output mode based on context. Do not always produce full JSON —
that would be disruptive in casual conversation.

### Mode 1: Silent (default in conversation)

For most conversational use, do not surface the FactCheckResult to the user.
Use it internally to decide how to respond:
- `accept` → continue naturally
- `clarify` → ask a focused follow-up question
- `correct` → weave the correction into your response naturally
- `search` → trigger web search before responding
- `escalate` → add a disclaimer and recommend professional advice

### Mode 2: Transparent (when user asks, or stakes are high)

Surface a human-readable summary:
```
I want to flag something before we proceed: [claim] may not be entirely accurate.
[1-2 sentence correction or uncertainty note.]
[Optional: here's what I'd suggest checking.]
```

### Mode 3: Structured (for agent pipelines)

Return the raw `FactCheckResult` JSON for downstream consumption.
Use this mode when this skill is called by an orchestrating agent, not directly by a human.

---

## Handling partial truths (hardest case)

Partial truths are the most dangerous error type because they feel correct and the
user may have genuine confidence in them. Signs of a partial truth:

- Claim is true in one jurisdiction but not another
- Claim was true historically but policy has changed
- Claim is true for one product tier but not another
- Claim omits a critical caveat that reverses its practical meaning

**Example (SinoEZ context):**
> User: "You can now use foreign credit cards in China"

Partially true: Visa/Mastercard are now accepted at some international hotels and
tourist spots, and WeChat Pay added a foreign card binding option in 2023. But the
coverage is still very limited and unreliable outside major tourist zones.

Verdict: `mostly_true` | error_type: `partial_truth`
Correction: Clarify scope and reliability before the user relies on this for trip planning.

---

## Special handling for SinoEZ agent context

When operating as part of the SinoEZ travel advisory agent, apply heightened scrutiny to:

- Claims about app availability or payment methods in China (change frequently)
- Visa and entry policy claims (update without notice)
- VPN legality claims (sensitive; do not provide legal advice)
- Claims about which apps "work" without VPN (version and region dependent)
- Health and safety claims about food, water, air quality

For any of these, default to `recommended_action: search` and add a note that
official sources or on-the-ground verification is recommended.

---

## Multi-claim conversations

In a longer conversation, maintain a running `ClaimLog`:

```json
{
  "session_claim_log": [
    {
      "turn": 3,
      "claim": "...",
      "verdict": "mostly_false",
      "corrected": true
    }
  ],
  "overall_reliability_signal": "high | medium | low | mixed"
}
```

Use `overall_reliability_signal` to calibrate how much to trust subsequent unverified
claims from the same user. A user who has made multiple false claims warrants higher
scrutiny on new assertions — especially if the downstream action has real-world consequences.

---

## References

For domain-specific claim patterns, read the relevant reference file:

- `references/china-digital-claims.md` — common misconceptions about Chinese apps,
  payments, connectivity, and regulations (high-priority for SinoEZ)
- `references/statistical-fallacies.md` — how to spot and correct common statistical
  errors in user-cited data

Load a reference file only when the claim type matches. Do not load both by default.
