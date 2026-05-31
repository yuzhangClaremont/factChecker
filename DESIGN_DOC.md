# TruthfulRAG Fact Checker — System Design Document

> **Design Doc v1.0** · May 2026  
> An opinionated architecture for building a low-hallucination, web-connected fact-checking assistant.

---

## Table of Contents

1. [Motivation & Problem Statement](#1-motivation--problem-statement)
2. [Related Academic Work](#2-related-academic-work)
3. [Design Goals & Non-Goals](#3-design-goals--non-goals)
4. [High-Level Architecture](#4-high-level-architecture)
5. [Core Pipeline: Stage by Stage](#5-core-pipeline-stage-by-stage)
6. [Multi-Agent System Design](#6-multi-agent-system-design)
7. [Anti-Hallucination Mechanisms](#7-anti-hallucination-mechanisms)
8. [Tech Stack Recommendations](#8-tech-stack-recommendations)
9. [Data Flow & API Design](#9-data-flow--api-design)
10. [UI / UX Mockup Concept](#10-ui--ux-mockup-concept)
11. [Evaluation & Metrics](#11-evaluation--metrics)
12. [Open Questions & Future Work](#12-open-questions--future-work)

---

## 1. Motivation & Problem Statement

Large Language Models (LLMs) generate plausible-sounding but factually incorrect content — **hallucinations**. Standard RAG (Retrieval-Augmented Generation) reduces this by grounding responses in retrieved documents, but still suffers from:

- **Retrieval failures**: missing relevant documents.
- **Context overload**: burying correct evidence in noise.
- **Sycophancy**: overconfidence in retrieved-but-irrelevant passages.
- **Citation fabrication**: inventing source URLs and titles.

**TruthfulRAG** is an emerging paradigm that adds *truthfulness-awareness* on top of standard RAG: the system explicitly evaluates whether each retrieved piece of evidence supports, contradicts, or is irrelevant to each atomic claim, and abstains when confidence is low.

**Our goal**: Build a web app where users paste a statement and receive a rigorous, evidenced truthfulness assessment — with citations, confidence scores, and a transparent reasoning trace.

---

## 2. Related Academic Work

This design draws directly from the following papers and frameworks:

| Paper / Framework | Key Idea | How We Use It |
|---|---|---|
| **TruthfulQA** (Lin et al., 2021) | Benchmark for model truthfulness; identifies that larger models aren't always more truthful | Baseline evaluation target |
| **Self-RAG** (Asai et al., 2023) | Trains LM to self-reflect: retrieve on demand, critique own generations | Core critique loop in Verifier Agent |
| **CRAG** (Yan et al., 2024) | Corrective RAG: evaluate retrieved docs, decompose/rewrite query if quality is low | Retrieval quality gate in Research Agent |
| **Chain-of-Verification** (Dhuliawala et al., 2023) | LLM generates initial response, then systematically verifies each claim | **Central verification loop** |
| **FActScore** (Min et al., 2023) | Fine-grained atomic fact decomposition for precision evaluation | Atomic claim splitting |
| **RARR** (Gao et al., 2023) | Research-then-revise: use retrieval to fact-check and edit LM output | Revision stage after verification |
| **REACT** (Yao et al., 2022) | Synergize reasoning + acting (tool use) in interleaved traces | Agent reasoning loop pattern |
| **FreshQA** (Vu et al., 2023) | Evaluation for freshness-aware QA, highlights staleness | Temporal confidence calibration |
| **RAGAS** (Shahul et al., 2023) | Metrics: faithfulness, answer relevance, context precision | Evaluation metrics |
| **Multi-Agent Debate** (Du et al., 2023) | Multiple LM instances debate to improve factuality | Cross-agent verification layer |
| **Mixture of Agents** (Wang et al., 2024) | Cooperative multi-agent pipelines with specialized roles | **Architecture pattern** for agent roles |
| **Chain-of-Table** (Wang et al., 2024) | Structured reasoning over tabular evidence | Handling structured data retrieval |
| **Constitutional AI** (Bai et al., 2022) | Self-critique via constitutional principles | Hallucination constitution for abstention |
| **Toolformer** (Schick et al., 2023) | LM learns to call APIs (search, calculator) | Web search tool use |

### 2.1 How TruthfulRAG Differs from Standard RAG

```
Standard RAG:
  Query → Retrieve → Generate → (no verification step)

TruthfulRAG:
  Query → Decompose → Retrieve → Verify Evidence → 
    → Generate Draft → Verify Claims → Revise → Output with Confidence
```

The key additions:
- **Claim decomposition** (FActScore-style)
- **Evidence verification** (entailment/contradiction classification)
- **Self-verification loop** (Chain-of-Verification)
- **Confidence calibration with abstention**

---

## 3. Design Goals & Non-Goals

### Goals

1. **High factual precision** (>95% on benchmark factual evaluations)
2. **Low hallucination rate** (every claim attributed to retrievable source)
3. **Calibrated confidence** — user sees uncertainty when evidence is weak
4. **Transparent reasoning** — full audit trail of evidence gathered and evaluation
5. **Real-time web research** — fetches and synthesizes current information
6. **Graceful abstention** — says "I don't know" confidently when uncertain

### Non-Goals

- Replacing professional fact-checkers in sensitive domains (health, law, finance) — always disclaim
- Real-time low-latency (target <30s for thorough research is acceptable)
- Handling non-textual formats (video, audio) in v1
- Training or fine-tuning custom models (pure prompting + retrieval pipeline)

---

## 4. High-Level Architecture

```
┌────────────────────────────────────────────────────────────┐
│                     User Interface (Web App)                │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ Input    │  │ Result   │  │ Evidence │  │Confidence│   │
│  │ Statement│  │ Verdict  │  │ Explorer │  │ Timeline │   │
│  └────┬─────┘  └────▲─────┘  └────▲─────┘  └────▲─────┘   │
└───────┼──────────────┼─────────────┼──────────────┼────────┘
        │              │             │              │
┌───────▼──────────────┼─────────────┼──────────────┼────────┐
│        API Gateway (FastAPI / Next.js API Routes)         │
└───────┬──────────────────────────────────────────────┬─────┘
        │                                              │
┌───────▼──────────────────────────────────────────────▼─────┐
│                    Orchestrator (Agent Loop)                │
│                                                             │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌─────────┐ │
│  │ Research │──▶│ Verifier │──▶│Aggregator│──▶│Reviser  │ │
│  │  Agent   │   │  Agent   │   │  Agent   │   │  Agent  │ │
│  └────┬─────┘   └────┬─────┘   └────┬─────┘   └────┬────┘ │
│       │              │              │              │       │
│  ┌────▼──────────────▼──────────────▼──────────────▼────┐  │
│  │              Arbitration & Consensus Layer            │  │
│  │   (Cross-validation, confidence scoring, abstention)  │  │
│  └───────────────────────┬──────────────────────────────┘  │
│                          │                                  │
└──────────────────────────┼──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│               Retrieval & Tool Layer                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────────┐   │
│  │ Web      │  │ Vector   │  │ Knowledge│  │ Cache &   │   │
│  │ Search   │  │ Database │  │ Graph    │  │ Rate Lim. │   │
│  └──────────┘  └──────────┘  └──────────┘  └───────────┘   │
└──────────────────────────────────────────────────────────────┘
```

---

## 5. Core Pipeline: Stage by Stage

### Stage 1: Input Normalization & Decomposition

```
User Input: "Einstein won the Nobel Prize for his theory of relativity in 1921."
                          │
                          ▼
Claim Decomposition (FActScore-style)
                          │
                          ▼
Atomic Claims:
  1. [Albert Einstein] [won] [the Nobel Prize]
  2. [The Nobel Prize was awarded] [in] [1921]
  3. [The prize was for] [the theory of relativity]
```

**Implementation**: Use a dedicated LLM call with structured output (JSON schema) to decompose the input into atomic claims. Each atomic claim is a subject-predicate-object triple.

### Stage 2: Multi-Source Retrieval (Research Agent)

For each atomic claim, the Research Agent:

1. **Generates search queries** (1–3 per claim, with query rewriting)
2. **Fetches from multiple sources**:
   - Web search (Google/Bing/Search1API/SerpAPI)
   - Vector database (pre-indexed knowledge corpus)
   - Wikipedia API
   - News APIs (for temporal claims)
3. **Scores & filters** retrieved passages using a **retrieval quality gate** (CRAG-style):
   - Relevance score > threshold → keep
   - Relevance score < threshold → rewrite query, retry
   - No results → mark as "insufficient evidence"

Output: `List[Document { source, snippet, relevance_score, date }]`

### Stage 3: Evidence Verification (Verifier Agent)

For each atomic claim with its retrieved documents:

1. **Entailment classification** (per document):
   - `SUPPORTS` — evidence directly confirms
   - `CONTRADICTS` — evidence directly refutes
   - `IRRELEVANT` — evidence doesn't address the claim
   - `AMBIGUOUS` — partial or unclear
2. **Aggregate across documents**:
   - Count supporting vs. contradicting sources
   - Weight by source authority
   - Compute **evidence strength score** [0.0, 1.0]
3. **Temporal confidence** (FreshQA-inspired):
   - Check if evidence freshness is sufficient for the claim

### Stage 4: Draft Generation (Aggregator Agent)

Synthesize findings into a structured verdict:

```json
{
  "overall_verdict": "MOSTLY_TRUE",
  "overall_confidence": 0.87,
  "claims": [
    {
      "claim": "Einstein won the Nobel Prize",
      "verdict": "TRUE",
      "confidence": 0.99,
      "evidence": [
        {"source": "nobelprize.org", "snippet": "...", "supports": true}
      ]
    },
    {
      "claim": "The prize was in 1921",
      "verdict": "PARTIALLY_TRUE",
      "confidence": 0.65,
      "evidence": [
        {"source": "nobelprize.org", "snippet": "Nobel Prize 1921 awarded in 1922", "supports": false}
      ],
      "explanation": "Awarded in 1921, but ceremony was in 1922"
    },
    {
      "claim": "Prize was for relativity",
      "verdict": "FALSE",
      "confidence": 0.98,
      "evidence": [
        {"source": "nobelprize.org", "snippet": "awarded for photoelectric effect", "supports": false}
      ]
    }
  ]
}
```

Verdict categories: `TRUE`, `MOSTLY_TRUE`, `PARTIALLY_TRUE`, `MOSTLY_FALSE`, `FALSE`, `INSUFFICIENT_EVIDENCE`

### Stage 5: Self-Verification & Revision (Reviser Agent)

Chain-of-Verification loop:

1. **Generate verification questions** for each claim and its evidence
2. **Re-retrieve** if needed to answer verification questions
3. **Check for contradictions** across the draft
4. **Revise** the draft (RARR-style):
   - Fix incorrect claims
   - Add nuance to partial claims
   - Downgrade confidence where evidence is thin
5. **Iterate** up to 2 rounds or until stable

### Stage 6: Final Output Assembly

- Render verdict + evidence tree in the UI
- Include **confidence intervals** (not just point estimates)
- Show **source provenance** (screenshots/cached snippets)
- Flag **disclaimers** for low-confidence or time-sensitive claims

---

## 6. Multi-Agent System Design

### 6.1 Agent Roles & Responsibilities

| Agent | Model | Tools | Responsibility |
|---|---|---|---|
| **Router Agent** | Fast/cheap model (GPT-4o-mini / Claude Haiku) | None (classifier only) | Classify input type (factual claim, opinion, question); route to appropriate pipeline or refuse |
| **Decomposer Agent** | Strong reasoning (GPT-4o / Claude Sonnet) | Structured output | Break statement into atomic claims |
| **Research Agent** | Strong (GPT-4o) | Web search, Wikipedia, Vector DB, News API | Multi-source evidence retrieval with query rewriting |
| **Verifier Agent** | Strong (GPT-4o) | Entailment classifier, calculator | Classify evidence-support relationship per claim |
| **Aggregator Agent** | Strong (GPT-4o) | None (synthesis only) | Produce structured verdict draft |
| **Reviser Agent** | Strong (GPT-4o) | Web search (re-retrieval) | Self-critique and revision loop |
| **Critic Agent** | Second distinct model (Claude Sonnet) | None | Cross-model verification: review and challenge findings from a different LLM to reduce systematic bias |

### 6.2 Agent Communication Protocol

Agents communicate through **structured messages** (JSON schema):

```typescript
interface AgentMessage {
  type: "claim" | "evidence" | "verdict" | "critique" | "revision";
  payload: Record<string, unknown>;
  metadata: {
    agent_id: string;
    timestamp: string;
    confidence: number; // self-reported confidence
  };
}
```

### 6.3 Cross-Model Validation

To reduce single-model hallucination bias, the **Critic Agent** uses a *different* LLM provider (Claude Sonnet) to:

- Review the aggregated verdict
- Identify missing evidence or reasoning gaps
- Flag overconfident claims
- Assign a **cross-model agreement score**

If cross-model agreement < threshold (e.g., 0.7), the Arbitration Layer triggers a re-verification cycle.

### 6.4 Arbitration Layer

```
                    ┌──────────────────────┐
                    │  Multi-Agent Outputs  │
                    └──────────┬───────────┘
                               ▼
                    ┌──────────────────────┐
                    │   Arbitration Logic   │
                    │                      │
                    │  • Vote weighting     │
                    │  • Confidence pooling │
                    │  • Contradiction      │
                    │    resolution         │
                    │  • Abstention trigger │
                    └──────────┬───────────┘
                               ▼
                    ┌──────────────────────┐
                    │   Final Verdict       │
                    └──────────────────────┘
```

**Confidence pooling**: Weighted average based on:
- Source quality + freshness (40% weight)
- Number of independent sources (30% weight)
- Cross-model agreement (20% weight)
- Self-reported LLM confidence (10% weight)

**Abstention threshold**: If pooled confidence < 0.5, system outputs "Insufficient evidence to make a determination" instead of a verdict.

---

## 7. Anti-Hallucination Mechanisms

### 7.1 Architectural Safeguards

| Mechanism | Paper Origin | Implementation |
|---|---|---|
| **Atomic decomposition** | FActScore | Prevents compound errors; each claim verified independently |
| **Evidence gate** | CRAG | Retrieval quality check before any generation |
| **Self-verification loop** | Chain-of-Verification | LLM checks its own work systematically |
| **Cross-model critique** | Multi-Agent Debate | Second LLM from different provider reviews verdict |
| **Source grounding** | RARR | Every claim in output must cite a retrievable source |
| **Confidence calibration** | TruthfulQA | Explicit confidence with abstention threshold |
| **Temporal awareness** | FreshQA | Evidence freshness check; marks stale claims |
| **Constitutional constraints** | Constitutional AI | Prompt-level rules: "If uncertain, say so; cite sources; don't guess" |

### 7.2 Prompt-Level Hallucination Constitution

Every agent prompt includes these constraints:

```
CONSTITUTIONAL RULES FOR ALL AGENTS:
1. THOU SHALT NOT FABRICATE: Never invent facts, sources, or citations.
2. THOU SHALT CITE: Every factual claim must reference a specific source.
3. THOU SHALT DOUBT: If evidence is thin or contradictory, say so explicitly.
4. THOU SHALT NOT GUESS: "I don't know" is preferred over speculation.
5. THOU SHALT VERIFY DATES: Check temporal relevance of all evidence.
6. THOU SHALT DISAGREE: If evidence contradicts the user, state it clearly.
```

### 7.3 Factuality Scoring at Inference Time

```python
# Simplified scoring logic
def compute_claim_confidence(claim, evidence_set):
    n_support = sum(1 for e in evidence_set if e.label == "SUPPORTS")
    n_contradict = sum(1 for e in evidence_set if e.label == "CONTRADICTS")
    n_total = len(evidence_set)

    if n_total == 0:
        return {"confidence": 0.0, "verdict": "INSUFFICIENT_EVIDENCE"}

    # Evidence proportion
    evidence_score = (n_support - n_contradict) / n_total

    # Source diversity bonus (more independent sources = better)
    source_diversity = min(len(set(e.domain for e in evidence_set)) / 3.0, 1.0)

    # Temporal recency factor
    temporal_score = compute_temporal_recency(evidence_set, claim)

    # Combined
    confidence = 0.5 * evidence_score + 0.3 * source_diversity + 0.2 * temporal_score
    confidence = max(-1.0, min(1.0, confidence))

    # Map to verdict
    if confidence > 0.7: verdict = "TRUE"
    elif confidence > 0.3: verdict = "MOSTLY_TRUE"
    elif confidence > -0.3: verdict = "PARTIALLY_TRUE"
    elif confidence > -0.7: verdict = "MOSTLY_FALSE"
    else: verdict = "FALSE"

    return {"confidence": (confidence + 1) / 2, "verdict": verdict}
```

---

## 8. Tech Stack Recommendations

| Layer | Technology | Rationale |
|---|---|---|
| **Frontend** | Next.js 14+ (React + TypeScript) | SSR for SEO, App Router for API integration, excellent DX |
| **UI Framework** | Tailwind CSS + shadcn/ui | Fast prototyping, accessible, consistent design system |
| **Backend / API** | Next.js API Routes or FastAPI | Co-located with frontend or separate if scaling needs differ |
| **LLM Provider** | OpenAI GPT-4o (primary) + Anthropic Claude Sonnet (critic) | Cross-model diversity reduces systematic bias |
| **Agent Framework** | LangGraph or Custom orchestration | LangGraph for state machines; custom for simpler cases |
| **Web Search** | SerpAPI / Tavily / Search1API | Structured search results, faster than scraping |
| **Vector DB** | Chroma or Supabase pgvector | Lightweight, no extra infra for v1 |
| **Caching** | Redis (Upstash) | Cache search results, rate limiting |
| **Streaming** | Server-Sent Events (SSE) | Real-time agent progress in UI |
| **Deployment** | Vercel (frontend) + Modal / Railway (agents) | Serverless for frontend, compute-heavy for agents |

### 8.1 Why These LLM Choices

| Role | Model | Why |
|---|---|---|
| Decomposer | GPT-4o | Best structured output, strong reasoning |
| Research Agent | GPT-4o | Strong tool-use, instruction following |
| Verifier | GPT-4o | Consistent classification |
| Aggregator | GPT-4o | Complex synthesis |
| Reviser | GPT-4o | Self-critique capabilities |
| Critic | Claude Sonnet | Different training distribution → different failure modes |

---

## 9. Data Flow & API Design

### 9.1 Core API Endpoint

```
POST /api/check
Content-Type: application/json

{
  "statement": "Einstein won the Nobel Prize for relativity in 1921.",
  "options": {
    "thoroughness": "standard" | "deep" | "quick",
    "include_sources": true,
    "max_claims": 10
  }
}

--- Response (streamed via SSE) ---

event: progress
data: {"stage": "decomposing", "claims": [...], "progress_pct": 10}

event: progress
data: {"stage": "researching", "query": "Einstein Nobel Prize", "sources_found": 5, "progress_pct": 30}

event: progress
data: {"stage": "verifying", "claim": "Einstein won Nobel Prize", "verdict": "TRUE", "confidence": 0.99, "progress_pct": 50}

event: progress
data: {"stage": "reviewing", "issues_found": 1, "progress_pct": 80}

event: complete
data: { ... final verdict object ... }
```

### 9.2 Internal Message Format

```typescript
// FactCheckSession tracks entire lifecycle
interface FactCheckSession {
  id: string;
  input_statement: string;
  atomic_claims: AtomicClaim[];
  evidence_map: Map<string, EvidenceSet>; // claim_id → evidence
  verdicts: Map<string, ClaimVerdict>;
  overall_verdict: OverallVerdict;
  confidence: number;
  evidence_tree: EvidenceNode[];
  trace: AgentTrace[];
  created_at: string;
}

interface EvidenceNode {
  id: string;
  claim_id: string;
  source_url: string;
  source_domain: string;
  snippet: string;
  relevance_score: number;
  entailment: "SUPPORTS" | "CONTRADICTS" | "IRRELEVANT" | "AMBIGUOUS";
  retrieval_timestamp: string;
  content_hash: string; // for deduplication
}
```

### 9.3 Caching Strategy

| Cache | Key | TTL | Purpose |
|---|---|---|---|
| Web search results | `search:{query_hash}` | 24h | Avoid duplicate API calls |
| Evidence classification | `entail:{claim_hash}:{source_hash}` | 7d | Reuse classification results |
| Fact verification | `verify:{claim_hash}` | 30d (or invalidated on news event) | Full verification reuse |

---

## 10. UI / UX Mockup Concept

### Screen 1: Statement Input

```
┌─────────────────────────────────────────────────────┐
│  🔍 TruthfulRAG Fact Checker                         │
│                                                      │
│  ┌──────────────────────────────────────────────┐   │
│  │  Paste or type a statement to fact-check...   │   │
│  └──────────────────────────────────────────────┘   │
│                                                      │
│  [Quick Check]  [Thorough Research]  [Deep Dive]    │
│                                                      │
│  Recent checks:                                      │
│  • "Einstein won Nobel..." → ⚠️ Partially True      │
│  • "Vaccines cause..." → ❌ False (3 sources)        │
└─────────────────────────────────────────────────────┘
```

### Screen 2: Live Research Progress

```
┌─────────────────────────────────────────────────────┐
│  🔍 Fact-Checking: "Einstein won Nobel..."          │
│                                                      │
│  ├─ ✅ Decomposing statement (3 claims)              │
│  ├─ 🔄 Researching claim 1/3...                      │
│  │   ├─ ✅ Searched: Einstein Nobel Prize             │
│  │   ├─ 🔄 Searching: Nobel Prize 1921 winner        │
│  │   └─ ⏳ Searching: Einstein relativity Nobel       │
│  ├─ ⏳ Verifying evidence...                          │
│  ├─ ⏳ Cross-checking with Claude...                  │
│  └─ ⏳ Generating final verdict...                    │
│                                                      │
│  Estimated time remaining: 15 seconds...             │
└─────────────────────────────────────────────────────┘
```

### Screen 3: Results Dashboard

```
┌─────────────────────────────────────────────────────┐
│  Verdict: ⚠️ PARTIALLY TRUE  Confidence: 87%         │
│                                                      │
│  ┌──────────────────────────────────────────────┐   │
│  │ Overall: Einstein won the Nobel Prize,        │   │
│  │ but NOT for relativity (it was for the        │   │
│  │ photoelectric effect), and the award year     │   │
│  │ was 1921 (ceremony in 1922).                  │   │
│  └──────────────────────────────────────────────┘   │
│                                                      │
│  ┌──────────┬──────────┬────────┬────────────────┐   │
│  │ Claim    │ Verdict  │ Conf.  │ Sources        │   │
│  ├──────────┼──────────┼────────┼────────────────┤   │
│  │ Won Nobel│ ✅ TRUE  │ 99%    │ nobelprize.org │   │
│  │ Year 1921│ ⚠️ PART  │ 65%    │ nobelprize.org │   │
│  │ For rela-│ ❌ FALSE │ 98%    │ nobelprize.org │   │
│  │ tivity   │          │        │ + 2 more       │   │
│  └──────────┴──────────┴────────┴────────────────┘   │
│                                                      │
│  [View Full Evidence Tree]  [Share]  [Export PDF]   │
└─────────────────────────────────────────────────────┘
```

### Screen 4: Evidence Explorer (Drill-Down)

```
┌─────────────────────────────────────────────────────┐
│  Evidence for: "Einstein won the Nobel Prize"       │
│                                                      │
│  ✅ SUPPORTS (3 sources)                             │
│  ├── nobelprize.org — "The Nobel Prize in Physics   │
│  │   1921 was awarded to Albert Einstein..."         │
│  ├── britannica.com — "Einstein received the 1921   │
│  │   Nobel Prize in Physics..."                      │
│  └── wikipedia.org — "He was awarded the 1921       │
│       Nobel Prize in Physics..."                     │
│                                                      │
│  ❌ CONTRADICTS (0 sources)                          │
│                                                      │
│  ↪️ Irrelevant (2 sources)                           │
│  ├── someblog.com — "Einstein's theory of..."        │
│  └── twitter.com — "Einstein was a genius..."        │
└─────────────────────────────────────────────────────┘
```

---

## 11. Evaluation & Metrics

### 11.1 Offline Benchmarks

| Benchmark | What It Measures | Target |
|---|---|---|
| **TruthfulQA** | Model truthfulness on common misconceptions | >85% |
| **FActScore** | Atomic fact precision in biographies | >90% |
| **FreshQA** | Temporal factuality | >85% |
| **RGB** (RAG Benchmark) | Retrieval + generation quality | >80% F1 |
| **Custom** (fact-checking dataset) | Domain-specific accuracy | >90% |

### 11.2 Online Metrics

| Metric | How to Measure | Target |
|---|---|---|
| User-reported accuracy | Feedback buttons (✅ / ❌) | >95% positive |
| Source verification rate | % of claims with ≥1 verifiable source | >98% |
| Abstention rate | % of queries where system says "insufficient evidence" | <10% (too many = system too conservative) |
| Citation accuracy | Random audit: are cited sources real? | 100% |
| Average confidence/accuracy gap | | <0.10 |

### 11.3 A/B Testing Framework

- **Single vs. multi-model** — does cross-model critique improve accuracy >5%?
- **With vs. without self-verification loop** — does Chain-of-Verification reduce hallucination?
- **Thorough vs. quick mode** — what's the accuracy-time tradeoff?

---

## 12. Open Questions & Future Work

### Open Questions for v1

1. **What web search API should we use?** Tavily is purpose-built for AI agents; Search1API is cheaper; SerpAPI is most mature.
2. **How do we handle multi-lingual statements?** Start with English only, expand later.
3. **What's the right agent framework?** LangGraph gives state machine control; CrewAI is simpler but less flexible.
4. **Should we cache per-user or globally?** Global cache with user-specific rate limits.
5. **How do we handle real-time events?** Use SSE vs. WebSockets vs. polling?

### Future Work (Post-v1)

- **Continuous learning loop**: User feedback fine-tunes the entailment classifier
- **Knowledge graph integration**: Store verified facts in a queryable graph
- **Browser extension**: Fact-check inline as user browses
- **Batch checking**: Upload documents for full-document fact-checking
- **Image/fact verification**: Multimodal (verify claims in images, infographics)
- **Community contribution**: Users can submit new evidence sources
- **Fine-tuned verifier model**: Train a small, fast entailment classifier (e.g., DeBERTa-based) to replace LLM calls for cheaper verification
- **Custom knowledge bases**: Enterprise users can add their own document corpus

---

## Appendix A: Minimal Viable Implementation (MVP)

### A.1 Single-file MVP Architecture

For an MVP, the system can be simplified:

```
1. User submits statement
2. Single LLM call to decompose into atomic claims
3. For each claim:
   a. Search web (Tavily API)
   b. LLM call to classify evidence (supports/contradicts/irrelevant)
4. Aggregate scores using the formula in §7.3
5. Output verdict + citations
```

This can be built as a single Next.js app with API routes, one primary LLM, one search API, and no multi-agent orchestration framework. The multi-agent design in §6 is for the production-grade system.

### A.2 MVP Checklist

- [ ] Next.js app with Tailwind/shadcn UI
- [ ] Single input form + streaming results display
- [ ] GPT-4o API integration
- [ ] Tavily web search integration
- [ ] Atomic claim decomposition (structured output)
- [ ] Evidence entailment classification
- [ ] Verdict rendering with confidence score
- [ ] Source citation display

---

## Appendix B: Related Paper References

1. Lin, S., Hilton, J., & Evans, O. (2021). *TruthfulQA: Measuring How Models Mimic Human Falsehoods*. ACL 2022.
2. Asai, A., Wu, Z., Wang, Y., Sil, A., & Hajishirzi, H. (2023). *Self-RAG: Learning to Retrieve, Generate, and Critique through Self-Reflection*. ICLR 2024.
3. Yan, S., et al. (2024). *CRAG: Corrective Retrieval Augmented Generation*. NAACL 2024.
4. Dhuliawala, S., et al. (2023). *Chain-of-Verification Reduces Hallucination in Large Language Models*. ACL 2024.
5. Min, S., et al. (2023). *FActScore: Fine-grained Atomic Evaluation of Factual Precision in Long-form Text Generation*. EMNLP 2023.
6. Gao, L., et al. (2023). *RARR: Researching and Revising What Language Models Say, Using Retrieval*. EMNLP 2023.
7. Yao, S., et al. (2022). *REACT: Synergizing Reasoning and Acting in Language Models*. ICLR 2023.
8. Vu, T., et al. (2023). *FreshQA: Large Language Models Need to Know When Not to Answer*. ACL 2024.
9. Du, Y., et al. (2023). *Improving Factuality and Reasoning in Language Models through Multiagent Debate*. ICML 2024.
10. Wang, J., et al. (2024). *Mixture of Agents: An Efficient Architecture for Large Language Model Collaboration*.
11. Bai, Y., et al. (2022). *Constitutional AI: Harmlessness from AI Feedback*. NeurIPS 2022.
12. Schick, T., et al. (2023). *Toolformer: Language Models Can Teach Themselves to Use Tools*. NeurIPS 2023.
13. Shahul, E. S., et al. (2023). *RAGAS: Automated Evaluation of Retrieval Augmented Generation*. EACL 2024.

---

> **Next steps**: Review this design doc, then we can begin building the MVP — starting with the Next.js scaffold, then the core pipeline, then the multi-agent production system.
