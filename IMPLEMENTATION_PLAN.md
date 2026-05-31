# TruthfulRAG Fact Checker — Implementation Plan

> Using **MiniMax API** (`gc_B-AJ8fsVwXzDSDQIe1MOOGgbDxLLmLVK`) for all AI agents  
> Based on the design doc at `DESIGN_DOC.md`

---

## 📦 MiniMax API Setup

**API Base**: `https://api.minimaxi.com/v1`  
**Model**: `MiniMax-Text-01` (strong reasoning, supports tool use, 4M context)  
**Auth**: Bearer token in `Authorization` header

### Environment Setup

```bash
# .env.local
MINIMAX_API_KEY=gc_B-AJ8fsVwXzDSDQIe1MOOGgbDxLLmLVK
MINIMAX_BASE_URL=https://api.minimaxi.com/v1
MINIMAX_MODEL=MiniMax-Text-01
SEARCH_API_KEY=  # Tavily or Search1API (to be added)
```

### Quick Test

```python
# test_minimax.py
from openai import OpenAI
client = OpenAI(
    api_key="gc_B-AJ8fsVwXzDSDQIe1MOOGgbDxLLmLVK",
    base_url="https://api.minimaxi.com/v1"
)
response = client.chat.completions.create(
    model="MiniMax-Text-01",
    messages=[{"role": "user", "content": "Hello"}]
)
print(response.choices[0].message.content)
```

---

## 🗺️ Implementation Roadmap

### Phase 1: MVP Core (Week 1)

| Step | Task | Files | Done |
|------|------|-------|------|
| 1.1 | Initialize Next.js + Tailwind + shadcn/ui scaffold | `app/`, `components/`, `tailwind.config.ts` | ☐ |
| 1.2 | MiniMax API wrapper with OpenAI-compatible client | `lib/minimax.ts` | ☐ |
| 1.3 | Single input form + streaming results UI | `app/page.tsx`, `components/FactCheckForm.tsx` | ☐ |
| 1.4 | Atomic claim decomposition endpoint | `app/api/decompose/route.ts` | ☐ |
| 1.5 | Mock web search (hardcoded or simulated) | `lib/search.ts` | ☐ |
| 1.6 | Evidence verification + verdict generation | `app/api/check/route.ts` | ☐ |
| 1.7 | Results display with confidence bars + sources | `components/ResultCard.tsx` | ☐ |

### Phase 2: Real Web Research (Week 2)

| Step | Task | Files | Done |
|------|------|-------|------|
| 2.1 | Integrate Tavily/Search1API web search | `lib/search.ts` | ☐ |
| 2.2 | Query rewriting & multi-query per claim | `lib/research.ts` | ☐ |
| 2.3 | CRAG-style retrieval quality gate | `lib/quality_gate.ts` | ☐ |
| 2.4 | Real-time progress streaming via SSE | `lib/streaming.ts` | ☐ |
| 2.5 | Caching layer (Redis or in-memory) | `lib/cache.ts` | ☐ |

### Phase 3: Multi-Agent System (Week 3)

| Step | Task | Files | Done |
|------|------|-------|------|
| 3.1 | Agent orchestration framework (LangGraph or custom) | `lib/orchestrator.ts` | ☐ |
| 3.2 | Router Agent + Decomposer Agent | `agents/router.ts`, `agents/decomposer.ts` | ☐ |
| 3.3 | Research Agent with tool use | `agents/researcher.ts` | ☐ |
| 3.4 | Verifier Agent (entailment classification) | `agents/verifier.ts` | ☐ |
| 3.5 | Aggregator + Reviser Agent (Chain-of-Verification) | `agents/aggregator.ts`, `agents/reviser.ts` | ☐ |

### Phase 4: Anti-Hallucination & Polish (Week 4)

| Step | Task | Files | Done |
|------|------|-------|------|
| 4.1 | Confidence calibration + abstention logic | `lib/confidence.ts` | ☐ |
| 4.2 | Evidence source deduplication & authority scoring | `lib/scoring.ts` | ☐ |
| 4.3 | Evidence Explorer drill-down UI | `components/EvidenceTree.tsx` | ☐ |
| 4.4 | History & recent checks | `lib/history.ts` | ☐ |
| 4.5 | Error handling, rate limits, fallbacks | `lib/errors.ts` | ☐ |

---

## 🏗️ Project Structure

```
factChecker/
├── DESIGN_DOC.md              # Full architecture document
├── IMPLEMENTATION_PLAN.md     # This file
├── .env.local                 # API keys (gitignored)
├── next.config.ts
├── package.json
├── tsconfig.json
├── tailwind.config.ts
├── app/
│   ├── layout.tsx
│   ├── page.tsx               # Main input page
│   ├── globals.css
│   ├── api/
│   │   ├── check/route.ts     # Main fact-check endpoint (SSE)
│   │   └── history/route.ts   # Recent checks
│   └── check/
│       └── [id]/page.tsx      # Individual result view
├── components/
│   ├── FactCheckForm.tsx       # Statement input
│   ├── ProgressTracker.tsx     # Live progress display
│   ├── ResultCard.tsx          # Verdict summary
│   ├── ClaimRow.tsx            # Per-claim result
│   ├── EvidenceTree.tsx        # Drill-down evidence browser
│   ├── ConfidenceBadge.tsx     % Visual confidence indicator
│   └── SourceCard.tsx          # Source citation card
├── lib/
│   ├── minimax.ts              # MiniMax API client wrapper
│   ├── search.ts               # Web search abstraction
│   ├── orchestrator.ts         # Agent orchestration
│   ├── confidence.ts           # Scoring & calibration
│   ├── cache.ts                # Caching layer
│   ├── errors.ts               # Error handling
│   └── types.ts                # Shared TypeScript types
├── agents/
│   ├── router.ts               # Input classifier
│   ├── decomposer.ts           # Claim decomposition
│   ├── researcher.ts           # Multi-source retrieval
│   ├── verifier.ts             # Entailment classification
│   ├── aggregator.ts           # Verdict synthesis
│   └── reviser.ts              # Self-verification & revision
└── public/
    └── assets/
```

---

## 🚀 MiniMax Agent Configuration

Since MiniMax is the sole LLM provider, we use **prompt specialization** and **structured output** to simulate different agent roles:

### Agent System Prompts

Each agent gets a distinct system prompt that defines its persona, tools, and output format:

```typescript
// lib/minimax.ts
const AGENT_CONFIGS = {
  decomposer: {
    systemPrompt: `You are a claim decomposition specialist.
Break the user's statement into atomic, verifiable claims.
Output a JSON array of { claim, subject, predicate, object } objects.
Rules:
- Each claim must be independently verifiable
- No compound claims (split "and" statements)
- Preserve all temporal and quantitative information`,
    response_format: { type: "json_object" }
  },
  verifier: {
    systemPrompt: `You are an evidence verification expert.
Given a claim and a set of retrieved documents, classify each document as:
- SUPPORTS: directly confirms the claim
- CONTRADICTS: directly refutes the claim
- IRRELEVANT: does not address the claim
- AMBIGUOUS: partially supports but with important caveats
Output JSON: { claim, documents: [{ source, label, confidence, relevant_snippet }] }`,
    response_format: { type: "json_object" }
  },
  // ... other agents
};
```

### Structured Output via JSON Mode

MiniMax's OpenAI-compatible API supports `response_format: { type: "json_object" }` — we use this extensively for all agent outputs to ensure parseable, machine-readable responses.

---

## 📝 MVP Implementation Steps (Detailed)

### Step 1: Scaffold the Next.js Project

```bash
npx create-next-app@latest . --typescript --tailwind --eslint --app --src-dir=false --import-alias="@/*"
npm install @radix-ui/react-progress @radix-ui/react-accordion @radix-ui/react-tabs
npm install lucide-react clsx tailwind-merge class-variance-authority
```

### Step 2: MiniMax Client

```typescript
// lib/minimax.ts
import OpenAI from "openai";

export const minimax = new OpenAI({
  apiKey: process.env.MINIMAX_API_KEY!,
  baseURL: process.env.MINIMAX_BASE_URL || "https://api.minimaxi.com/v1",
});

export const MODEL = process.env.MINIMAX_MODEL || "MiniMax-Text-01";

export async function agentCall(
  systemPrompt: string,
  userMessage: string,
  options?: { json?: boolean; temperature?: number }
) {
  const response = await minimax.chat.completions.create({
    model: MODEL,
    messages: [
      { role: "system", content: systemPrompt },
      { role: "user", content: userMessage },
    ],
    response_format: options?.json ? { type: "json_object" } : undefined,
    temperature: options?.temperature ?? 0.3, // Low temp for accuracy
  });
  return response.choices[0].message.content!;
}
```

### Step 3: Fact-Check Endpoint (SSE Stream)

```typescript
// app/api/check/route.ts
export async function POST(req: Request) {
  const { statement } = await req.json();

  const stream = new ReadableStream({
    async start(controller) {
      // 1. Decompose
      controller.enqueue(encodeSSE("progress", { stage: "decomposing", pct: 10 }));
      const claims = await decomposeClaims(statement);

      // 2. Research each claim
      controller.enqueue(encodeSSE("progress", { stage: "researching", pct: 30 }));
      const evidenceMap = await researchClaims(claims);

      // 3. Verify
      controller.enqueue(encodeSSE("progress", { stage: "verifying", pct: 50 }));
      const verdicts = await verifyClaims(claims, evidenceMap);

      // 4. Aggregate & score
      controller.enqueue(encodeSSE("progress", { stage: "aggregating", pct: 70 }));
      const draft = aggregateVerdict(verdicts);

      // 5. Self-verify & revise
      controller.enqueue(encodeSSE("progress", { stage: "reviewing", pct: 85 }));
      const final = await reviseVerdict(draft, evidenceMap);

      // 6. Complete
      controller.enqueue(encodeSSE("complete", final));
      controller.close();
    },
  });

  return new Response(stream, { headers: { "Content-Type": "text/event-stream" } });
}
```

### Step 4: Verdict Display Component

```tsx
// components/ResultCard.tsx
export function ResultCard({ verdict }: { verdict: FactCheckResult }) {
  return (
    <div className="space-y-4">
      <div className="flex items-center gap-3">
        <VerdictBadge verdict={verdict.overall} />
        <ConfidenceBar value={verdict.confidence} />
      </div>
      <div className="text-sm text-muted-foreground">{verdict.summary}</div>
      <div className="divide-y">
        {verdict.claims.map((claim) => (
          <ClaimRow key={claim.id} claim={claim} />
        ))}
      </div>
    </div>
  );
}
```

---

## 🔑 Key Decisions for MiniMax-Only Architecture

| Decision | Rationale |
|---|---|
| **Single model for all agents** | MiniMax-Text-01 supports 4M context + function calling; no need for multiple providers at MVP stage |
| **Prompt specialization** over model specialization | Each agent gets a different system prompt + JSON schema instead of a different model |
| **Low temperature (0.1–0.3)** | Minimizes creative hallucination; we want deterministic, factual outputs |
| **JSON mode for all structured outputs** | Ensures parseable, machine-readable agent communication |
| **No agent framework in MVP** | Custom orchestration with async/await is simpler than LangGraph for the single-model case |

---

## 📊 Cost Estimate (MiniMax)

MiniMax API pricing (approximate, may vary):

- **MiniMax-Text-01**: ~¥0.025/1K input tokens, ~¥0.1/1K output tokens

Per fact-check (avg 500 input + 3 claims × 3 searches × 2 LLM calls × 200 output tokens):
- ~3,000 input tokens + ~1,500 output tokens ≈ ¥0.23 (~$0.03 USD) per check
- **1,000 checks/month ≈ $30 USD**

---

## 🔜 Next Steps (After MVP)

1. Add **Tavily web search** for real internet research
2. Add **research depth levels** (quick / thorough / deep dive)
3. Add **cross-model critique** if a second provider becomes available
4. Add **user history & shareable result links**
5. Add **feedback loop** (thumbs up/down to improve scoring)

---

> **Ready to start building.** Phase 1, Step 1: scaffold the Next.js project and set up the MiniMax client.
