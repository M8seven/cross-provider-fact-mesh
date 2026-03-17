# Cross-Provider Fact Mesh: Zero-Cost Claim Verification for Multi-LLM Fusion Systems

**Defensive Publication — Establishing Prior Art**

**Authors:** Valentino Paulon
**Date:** March 17, 2026
**Affiliation:** OneAnswerAI (independent)
**Implementation:** Production since March 2026 in OneAnswerAI v2.0

---

## Abstract

We describe **Cross-Provider Fact Mesh (CPFM)**, a verification layer for multi-provider Large Language Model (LLM) fusion systems that detects and annotates factual claims without requiring additional API calls, external databases, or supplementary LLM invocations. CPFM operates entirely through deterministic pattern matching (~1ms latency, zero marginal cost) on existing provider responses and web search context that are already present in the fusion pipeline.

Unlike prior approaches to LLM hallucination mitigation — which rely on extra LLM calls (self-consistency, debate protocols, retrieval-augmented verification) — CPFM exploits the structural redundancy inherent in multi-provider architectures: when N≥2 independent models answer the same question, their agreement and disagreement patterns constitute a free verification signal that is currently discarded by naive fusion strategies.

---

## 1. Problem Statement

Multi-provider LLM systems query multiple AI models (e.g., GPT-4, Gemini, Claude) in parallel and fuse their responses into a single answer. This architecture exists in production systems including OneAnswerAI, You.com, and various enterprise deployments.

A critical failure mode occurs during fusion: **information loss under uncertainty**. When providers disagree on a specific detail (e.g., an episode number), naive fusion models tend to either:

1. **Omit the detail entirely** — "the episode number is not confirmed" (even when 2/3 providers and web search agree)
2. **Hallucinate a compromise** — selecting an incorrect middle ground
3. **Arbitrarily pick one** — without signaling confidence

The fusion model has no structured way to know which claims are well-supported and which are contested.

## 2. Prior Art

Existing approaches to LLM factual verification:

| Method | Mechanism | Cost | Latency |
|--------|-----------|------|---------|
| **Self-Consistency** (Wang et al., 2023) | Sample N completions from one model, majority vote | N× base cost | N× base latency |
| **LLM-as-Judge** | Separate LLM call to evaluate factuality | 1× additional call | +2-10s |
| **DelphiAgent** (2024) | Multi-agent debate with iterative LLM rounds | 3-5× base cost | 3-5× base latency |
| **ReConcile** (2024) | Multi-model roundtable discussion | 3-5× base cost | 3-5× base latency |
| **FIRE** (2024) | Retrieval + LLM re-evaluation | 1× retrieval + 1× LLM | +3-8s |
| **RAG Verification** | Retrieve, then verify claims against retrieved docs via LLM | 1× retrieval + 1× LLM | +2-5s |

**All existing methods require additional LLM inference calls.** None exploit the cross-provider redundancy signal that is already present (and paid for) in multi-provider architectures.

## 3. Method: Cross-Provider Fact Mesh

### 3.1 Architecture

CPFM is a pure-computation layer inserted between the provider response collection phase and the fusion prompt construction phase of a multi-provider LLM system:

```
Provider A response ──┐
Provider B response ──┼──→ CPFM ──→ Annotated Fusion Prompt ──→ Fusion Model
Provider C response ──┤              (claims + verdicts)
Web search context ───┘
```

No additional API calls are made. CPFM operates on data already present in the pipeline.

### 3.2 Pipeline Stages

#### Stage 1: Claim Extraction

Deterministic pattern matching extracts structured claims from each provider's response text. The current implementation targets **numeric claims** in entertainment/trivia domains:

**Claim types:**
- Seasons (e.g., "Season 8", "la terza stagione", "5 seasons", "5 stagioni")
- Episodes (e.g., "Episode 14", "62 episodes", "prima puntata")
- Years (e.g., "2019", "released in 2008")
- Season×Episode notation (e.g., "S05E14", "S6E1")

**Pattern categories:**
- Keyword-before-number: `season 5`, `stagione 3`, `episode 14`
- Number-before-keyword (plural only): `5 seasons`, `62 episodi`, `8 puntate`
- Ordinal forms: `first season`, `terza stagione`, `prima puntata`
- Compound notation: `S05E14` → populates both season and episode claims
- Bilingual: English + Italian patterns (extensible to other languages)

**Design choice — plural-only for N-before-keyword:** The pattern `(\d+)\s+keyword` only matches plural forms to avoid misparse. "Season 8 episode 15" contains "8 episode" which would incorrectly match as "8 episodes" with a singular-inclusive pattern. By requiring plural ("8 episodes", "5 seasons"), we eliminate this false positive while capturing the intended "how many" pattern.

The same extraction runs on web search context (with `[N]` reference markers stripped).

#### Stage 2: Cross-Reference Scoring

Each extracted claim value is scored by comparing across providers and web search:

| Status | Condition | Confidence |
|--------|-----------|------------|
| **VERIFIED** | Web search confirms the value | Highest |
| **LIKELY** | ≥2 providers agree, no web data for this claim type | High |
| **UNCERTAIN** | 2/3 providers agree, 1 disagrees, no web to arbitrate | Medium |
| **UNVERIFIED** | Only 1 provider states this value, no web data | Low |
| **CONTRADICTED** | Providers disagree, no web to break tie | Flag for omission |

**Scoring precedence:** Web confirmation (VERIFIED) takes absolute priority. When web data exists for a claim type, all provider claims of that type are scored against web, ignoring inter-provider agreement. When no web data exists, inter-provider agreement determines the verdict.

**Contradiction deduplication:** Multiple CONTRADICTED entries for the same claim type are collapsed into a single annotation showing all competing values.

#### Stage 3: Annotation Block Construction

Scored claims are formatted as a structured text block:

```
FACT VERIFICATION RESULTS:
- [VERIFIED by web] Season 5 — claimed by Model A, Model B, Model C
- [VERIFIED by web] Episode 62 — claimed by Model A, Model B
- [UNVERIFIED] Episode 64 — only Model C → HEDGE with "reportedly" or "approximately"
- [CONTRADICTED] Year: Model A says 2019; Model B says 2020 → OMIT specific number
```

#### Stage 4: Fusion Prompt Injection

The annotation block is injected into the fusion model's prompt, between the web search context and the "produce fused answer" instruction. The fusion model reads the verdicts as structured guidance, not as a constraint system — it may still exercise judgment but with explicit signals about claim reliability.

### 3.3 Guard Conditions

CPFM activates only when:
1. The query is classified as **factual** (detected via question-word + topic-indicator heuristic)
2. **≥2 provider responses** are available (single-provider mode has no cross-reference signal)
3. **≥1 numeric claim** is extracted from any provider response

When guard conditions are not met, CPFM returns an empty string and the fusion prompt is unchanged (zero behavioral change for non-applicable queries).

### 3.4 Computational Cost

| Operation | Time | API Calls | Memory |
|-----------|------|-----------|--------|
| Claim extraction (3 providers + web) | <1ms | 0 | O(n) where n = total text length |
| Scoring | <0.1ms | 0 | O(k) where k = unique claim values |
| Annotation formatting | <0.1ms | 0 | O(k) |
| **Total CPFM overhead** | **<1ms** | **0** | **Negligible** |

## 4. Key Innovation

The core insight is that **multi-provider LLM architectures already contain a verification signal in the form of inter-provider agreement patterns**, and this signal can be extracted and formalized through deterministic computation without any additional inference calls.

This is distinct from:
- **Self-consistency** — which generates multiple samples from a single model (CPFM uses responses that were already generated from different models for the primary answer)
- **LLM-as-Judge** — which requires an additional LLM call to evaluate (CPFM is purely deterministic)
- **Debate/discussion protocols** — which require iterative multi-round LLM exchanges (CPFM is single-pass, non-iterative)
- **RAG verification** — which requires retrieval + LLM re-evaluation (CPFM uses web search results already present in the pipeline)

## 5. Empirical Observations

In production testing (March 2026, OneAnswerAI v2.0):

**Without CPFM:**
> "Lo zio Junior spara a Tony nella sesta stagione. L'episodio esatto non è confermato dalle fonti disponibili."

The fusion model discarded a correct detail (Episode 1, "Members Only") that one provider stated, because it couldn't assess inter-provider agreement.

**With CPFM:**
> "L'attentato avviene al primo episodio della stagione: zio Junior, in preda a demenza, spara a Tony."

The CPFM annotation `[VERIFIED by web] Season 6` + `[UNVERIFIED] Episode 1` gave the fusion model enough signal to include the episode number (correctly) while understanding it had lower certainty.

## 6. Extensibility

The CPFM architecture is domain-agnostic. While the current implementation targets numeric entertainment claims, the extraction stage can be extended to:

- **Scientific claims**: quantities, measurements, chemical formulas
- **Geographic claims**: coordinates, distances, population figures
- **Financial claims**: prices, percentages, market figures
- **Historical claims**: dates, durations, sequences of events
- **Medical claims**: dosages, statistics, diagnostic criteria

Each domain requires only new regex patterns in the extraction stage. The scoring and annotation stages are domain-independent.

The bilingual approach (English + Italian) demonstrates the pattern for multilingual extension — each language adds extraction patterns without changing the scoring logic.

## 7. Limitations

1. **Claim type coverage**: Currently limited to numeric claims. Qualitative claims ("the character is a doctor" vs "a lawyer") are not extracted.
2. **Semantic equivalence**: "Season 6 Episode 1" and "the premiere of Season 6" express the same claim but only the former is extracted. Natural language variation in how claims are expressed limits recall.
3. **Web search dependency for VERIFIED status**: VERIFIED is the highest-confidence verdict but requires web search to be enabled and to contain relevant numeric data.
4. **Domain specificity of extraction patterns**: Each new domain requires hand-crafted regex patterns, though the scoring framework is reusable.

## 8. Conclusion

Cross-Provider Fact Mesh demonstrates that multi-provider LLM systems contain an untapped verification signal that can be exploited at zero marginal cost. By formalizing inter-provider agreement patterns through deterministic extraction and scoring, CPFM provides structured factual guidance to fusion models without the latency and cost penalties of existing verification approaches.

This defensive publication establishes prior art for the method described herein, effective March 17, 2026, as implemented in OneAnswerAI v2.0 production systems.

---

## Citation

```bibtex
@misc{paulon2026cpfm,
  title={Cross-Provider Fact Mesh: Zero-Cost Claim Verification for Multi-LLM Fusion Systems},
  author={Paulon, Valentino},
  year={2026},
  month={March},
  note={Defensive Publication. Implementation: OneAnswerAI v2.0},
  url={https://github.com/M8seven/cross-provider-fact-mesh}
}
```

## License

This document is published as a **Defensive Publication** under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). The method described herein is placed into the public record to establish prior art and prevent third-party patent claims. Anyone may implement this method freely.
