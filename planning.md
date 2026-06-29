# Provenance Guard — Planning Document

## System Architecture Narrative

A piece of text enters the system via `POST /submit`. The endpoint validates the request, assigns a unique `content_id`, and passes the raw text through two independent detection signals in parallel. Signal 1 is an LLM-based classifier (Groq / llama-3.3-70b-versatile) that assesses semantic and stylistic coherence holistically. Signal 2 is a stylometric heuristics engine that measures structural, statistical properties of the text — sentence length variance, type-token ratio (vocabulary diversity), and punctuation density. The two scores are combined via a weighted average into a single confidence score (0.0 = certainly human, 1.0 = certainly AI). That score maps to one of three transparency label variants. The result, both signal scores, the combined confidence, and the label are written to a structured audit log (SQLite). The full response is returned to the caller. 

An appeal enters via `POST /appeal` with a `content_id` and the creator's reasoning. The system retrieves the original classification, sets its status to `"under_review"`, and appends an appeal record to the audit log.

---

## Architecture

```
POST /submit
     │
     ▼
┌────────────────────┐
│  Input Validation  │
│  + content_id gen  │
└────────┬───────────┘
         │  raw text
    ┌────┴──────────────────────────────────┐
    │                                       │
    ▼                                       ▼
┌──────────────────┐             ┌─────────────────────┐
│  Signal 1: LLM   │             │  Signal 2:           │
│  (Groq)          │             │  Stylometric         │
│  Output: 0.0–1.0 │             │  Heuristics          │
│  (AI probability)│             │  Output: 0.0–1.0     │
└────────┬─────────┘             └──────────┬──────────┘
         │ llm_score                         │ stylo_score
         └──────────────┬────────────────────┘
                        ▼
              ┌──────────────────────┐
              │  Confidence Scoring  │
              │  weighted average    │
              │  (LLM 60%, Stylo 40%)│
              └──────────┬───────────┘
                         │ confidence: 0.0–1.0
                         ▼
              ┌──────────────────────┐
              │  Label Generation    │
              │  >0.70 → likely AI   │
              │  <0.35 → likely human│
              │  else  → uncertain   │
              └──────────┬───────────┘
                         │ label text
                         ▼
              ┌──────────────────────┐
              │  Audit Log (SQLite)  │
              │  writes entry        │
              └──────────┬───────────┘
                         │
                         ▼
                   JSON response


POST /appeal
     │
     ▼
┌────────────────────────┐
│  Look up content_id    │
│  in audit log          │
└────────┬───────────────┘
         │
         ▼
┌────────────────────────┐
│  Update status →       │
│  "under_review"        │
└────────┬───────────────┘
         │
         ▼
┌────────────────────────┐
│  Append appeal record  │
│  to audit log          │
└────────┬───────────────┘
         │
         ▼
   JSON confirmation
```

---

## Detection Signals

### Signal 1: LLM-Based Classification (Groq)

**What it measures:** Semantic and stylistic coherence holistically. The LLM is prompted to assess whether a piece of text reads as human-authored or AI-generated, considering factors like tone variety, naturalness of phrasing, redundancy patterns, and thematic consistency.

**Why this differs between human and AI writing:** AI text tends toward smooth, even prose without the micro-inconsistencies, register shifts, or idiosyncratic phrasing that human writers produce organically. The LLM is well-positioned to recognize these patterns because it was trained on vast amounts of both types.

**Output format:** A float from 0.0 to 1.0, where 1.0 = high probability of AI generation. Extracted from a structured JSON response from the model.

**Blind spots:** The LLM may flag highly polished human prose (e.g., professionally edited magazine writing) as AI-generated because it's smooth and well-structured. It can also be confused by AI text that has been lightly edited by a human to introduce irregularities. It reflects training biases — it may perform differently on non-English or code-switched text.

---

### Signal 2: Stylometric Heuristics (Pure Python)

**What it measures:** Statistical structural properties of text that differ between human and AI writing:

1. **Sentence Length Variance (SLV):** Standard deviation of sentence lengths in words. Human writing has higher variance (mix of short punchy sentences and long complex ones). AI tends to produce more uniform sentence lengths.

2. **Type-Token Ratio (TTR):** Unique words / total words. Human writers use a more diverse, idiosyncratic vocabulary. AI text often repeats the same transition words and phrases, lowering TTR.

3. **Punctuation Density (PD):** Non-standard punctuation marks (dashes, ellipses, semicolons, exclamation points) per sentence. Human writers use these more liberally and variably; AI defaults to commas and periods.

**Why this differs between human and AI writing:** AI language models are trained to minimize perplexity, which naturally produces statistically "average" text — average sentence lengths, average vocabulary richness. Human writing reflects individual habits, errors, and stylistic choices that deviate from the statistical mean.

**Output format:** A float from 0.0 to 1.0, where 1.0 = high probability of AI generation, derived from combining the three sub-metrics.

**Blind spots:** Formal human writing (academic papers, legal documents, business reports) naturally has lower variance, lower TTR, and less expressive punctuation — this signal will systematically over-classify such text as AI-generated. Short texts (< 3 sentences) produce unreliable statistics. Non-prose formats (poetry, dialogue-heavy fiction) don't fit the assumptions.

---

## Combining Signals into a Confidence Score

Both signals produce a score in [0.0, 1.0] where higher = more AI-like.

Combined score = (0.6 × llm_score) + (0.4 × stylo_score)

**Rationale for weighting:** The LLM signal captures richer, harder-to-quantify semantic features and is more reliable across text types. Stylometrics is a useful independent check but is more vulnerable to false positives on formal human writing, so it gets a lower weight.

---

## Uncertainty Representation

| Confidence Score | Label Category | Meaning |
|---|---|---|
| > 0.70 | Likely AI-generated | Strong signal from both detectors that this text exhibits AI patterns |
| 0.35 – 0.70 | Uncertain | Signals disagree or are weak; human review recommended |
| < 0.35 | Likely human-written | Strong signal that this text exhibits human authorship patterns |

**What 0.6 means:** The system has a weak-to-moderate lean toward AI but is not confident. Both detectors may be giving middling scores, or they may disagree. The label in this range acknowledges uncertainty explicitly.

**Design philosophy:** False positives (labeling a human as AI) are worse than false negatives for a creative writing platform. The uncertain band is deliberately wide (0.35–0.70) to reduce over-confident misclassification of human writers. The high-confidence AI threshold (0.70) is intentionally conservative.

---

## Transparency Label Variants

### High-Confidence AI (confidence > 0.70)
```
⚠️ AI-Generated Content Detected
Our system found strong signals that this content was likely produced by an 
AI writing tool, not written by a human author. Confidence: [X]%.

If you believe this is incorrect, you can submit an appeal — we review all 
contested classifications.
```

### Uncertain (confidence 0.35–0.70)
```
🔍 Authorship Unclear
Our system couldn't determine with confidence whether this content was 
written by a human or generated by AI. Confidence in AI authorship: [X]%.

We've flagged this for additional context. The creator may choose to appeal 
if they believe the classification is inaccurate.
```

### Likely Human-Written (confidence < 0.35)
```
✅ Likely Human-Written
Our system found strong signals that this content was written by a human 
author. Confidence in human authorship: [X]%.

This label reflects our best assessment — it is not a guarantee. 
Detection systems are imperfect.
```

---

## Appeals Workflow

**Who can appeal:** Any creator who submitted content — identified by `creator_id` in the original submission. (In a production system this would be authenticated; here we accept it as a field.)

**What they provide:** `content_id` (from the submission response) and `creator_reasoning` (free text explaining why they believe the classification is incorrect).

**What the system does on appeal:**
1. Looks up the original classification record by `content_id`
2. Updates the record's status from `"classified"` to `"under_review"`
3. Appends an appeal entry to the audit log with timestamp, `content_id`, `creator_id`, `creator_reasoning`, and the original classification details
4. Returns a confirmation JSON with appeal ID and status

**What a human reviewer sees:** When a reviewer queries `GET /log`, they see all entries. Appeal entries are distinguished by `event_type: "appeal"` and include `creator_reasoning`. The corresponding classification entry has `status: "under_review"`. A reviewer can see both side-by-side to make a determination.

**No automated re-classification:** The system does not automatically change the classification on appeal. A human reviewer makes that call.

---

## Anticipated Edge Cases

1. **Formal human academic or business writing:** A graduate student submitting their thesis excerpt will likely score high on the stylometric signal (low variance, formal vocabulary, uniform sentence structure) and potentially high on the LLM signal too, since the model associates polished prose with AI. This is the most likely false-positive scenario. The wide uncertain band mitigates but doesn't eliminate this.

2. **AI-generated poetry or experimental prose:** Poetry uses short, high-variance sentences, unconventional punctuation, and rich but narrow vocabulary — the exact opposite of what the stylometric signal looks for. A sonnet generated by GPT-4 might score very low on the stylometric signal, dragging the combined score toward "uncertain" or even "human." The system would under-classify AI-generated creative/experimental content.

3. **Very short texts (< 50 words):** Both signals become unreliable with small samples. Stylometric statistics are meaningless with 2–3 sentences. The LLM has less to work with. The system should note in its response when text is short and lower confidence accordingly.

4. **Non-native English speakers with formal writing styles:** A human writer who learned English formally may write in an unusually regular, polished style that pattern-matches to AI characteristics.

---

## Rate Limiting Rationale

- **10 requests per minute per IP:** A single human creator submitting their own work would rarely need to submit more than 1–2 pieces per session. 10/minute allows for rapid iteration and testing without enabling trivial flooding.
- **100 requests per day per IP:** Generous enough for a power user submitting many pieces; restrictive enough that running a large-scale evasion study would require rotating IPs.
- **Reasoning:** The adversarial model here is someone trying to systematically probe the classifier to find evasion techniques. Rate limiting makes brute-force probing slow and expensive.

---

## AI Tool Plan

### Milestone 3: Submission Endpoint + Signal 1
- **Spec sections provided to AI:** Detection Signals (Signal 1), Architecture diagram, API surface sketch
- **What I'll ask for:** Flask app skeleton with `POST /submit` route stub + `classify_with_llm()` function that calls Groq and returns a float score
- **Verification:** Call `classify_with_llm()` directly on 3 test inputs; confirm it returns a float in [0,1] before wiring into the route. Check Flask route returns valid JSON with `content_id`.

### Milestone 4: Signal 2 + Confidence Scoring
- **Spec sections provided to AI:** Detection Signals (Signal 2), Uncertainty Representation, Architecture diagram
- **What I'll ask for:** `classify_with_stylometrics()` function + `combine_scores()` weighted average function
- **Verification:** Run both signals on the 4 test inputs specified in the milestone. Print individual scores. Confirm that clearly AI text scores > 0.7 and clearly human text scores < 0.35 on the combined score.

### Milestone 5: Production Layer
- **Spec sections provided to AI:** Transparency Label Variants, Appeals Workflow, Architecture diagram
- **What I'll ask for:** `generate_label()` function + `POST /appeal` endpoint + Flask-Limiter setup
- **Verification:** Submit text that hits each of the three label bands and confirm label text matches spec exactly. Test appeal endpoint with a real `content_id` from a prior submission and check `GET /log` shows `"under_review"` status. Test rate limiting with 12 rapid requests and confirm 429s appear after request 10.

---

## Stretch Features (Planned)

- **Ensemble detection (3+ signals):** Add a third signal — perplexity approximation via bigram entropy — to the stylometric pipeline. Document the updated weighting.
- **Analytics dashboard:** `GET /analytics` endpoint returning detection pattern summary, appeal rate, and score distribution histogram data.
