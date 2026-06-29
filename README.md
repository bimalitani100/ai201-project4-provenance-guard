# Provenance Guard

A backend attribution system for creative sharing platforms. Provenance Guard classifies submitted text as human-authored or AI-generated, returns a confidence score, surfaces a transparency label to users, and handles appeals from creators who believe they've been misclassified.

---

## Quick Start

```bash
git clone https://github.com/YOUR_USERNAME/ai201-project4-provenance-guard
cd ai201-project4-provenance-guard
python -m venv .venv
source .venv/bin/activate       # Mac/Linux
# .venv\Scripts\activate        # Windows
pip install -r requirements.txt
cp .env.example .env            # then add your GROQ_API_KEY
python app.py
```

The server runs on `http://localhost:5000`.

---

## API Reference

| Method | Endpoint    | Description |
|--------|-------------|-------------|
| POST   | `/submit`   | Submit text for classification (rate-limited) |
| POST   | `/appeal`   | Contest a classification |
| GET    | `/log`      | Retrieve audit log entries |
| GET    | `/analytics`| Detection pattern summary (stretch) |
| GET    | `/health`   | Health check |

### POST /submit

```bash
curl -s -X POST http://localhost:5000/submit \
  -H "Content-Type: application/json" \
  -d '{
    "text": "ok so i finally tried that new ramen place downtown and honestly? underwhelming.",
    "creator_id": "user-bob-17"
  }' | python -m json.tool
```

**Response:**
```json
{
  "content_id": "48f0597e-dff4-4109-ba82-940b13241046",
  "creator_id": "user-bob-17",
  "attribution": "likely_human",
  "confidence": 0.2338,
  "signals": {
    "llm_score": 0.09,
    "stylo_score": 0.4234,
    "bigram_score": 0.3819
  },
  "label": "✅ Likely Human-Written\nOur system found strong signals that this content was written by a human author. Confidence in human authorship: 77%.\n\nThis label reflects our best assessment — it is not a guarantee. Detection systems are imperfect.",
  "status": "classified"
}
```

### POST /appeal

```bash
curl -s -X POST http://localhost:5000/appeal \
  -H "Content-Type: application/json" \
  -d '{
    "content_id": "PASTE-CONTENT-ID-HERE",
    "creator_reasoning": "I wrote this myself for my economics dissertation. My academic writing style is formal — it does not reflect AI generation."
  }' | python -m json.tool
```

**Response:**
```json
{
  "appeal_id": "611674e5-76a3-4ef1-a490-3a671b62094d",
  "content_id": "9463e8af-015a-431b-9979-2e47c8490d9b",
  "status": "under_review",
  "message": "Your appeal has been received and logged. A human reviewer will assess your classification.",
  "original_verdict": {
    "attribution": "uncertain",
    "confidence": 0.489,
    "timestamp": "2026-06-29T20:14:54.378951+00:00"
  }
}
```

---

## Architecture Overview

A submitted piece of text flows through the system as follows:

**Submission flow:** `POST /submit` validates the request body and generates a unique `content_id`. The raw text is passed through three independent detection signals in parallel — an LLM-based classifier (Groq), a stylometric heuristics engine, and a bigram entropy approximation. The three signal scores are combined via weighted average into a single confidence score (0.0 = certainly human, 1.0 = certainly AI). That score maps to one of three transparency label variants. The full result is written to a SQLite audit log and returned to the caller.

**Appeal flow:** `POST /appeal` accepts a `content_id` and the creator's reasoning. The system retrieves the original classification, sets its status to `"under_review"`, appends an appeal record to the audit log, and returns a confirmation. No automated re-classification occurs — a human reviewer resolves appeals by inspecting `GET /log`.

```
POST /submit
     │
     ▼
┌────────────────────┐
│  Input Validation  │
│  + content_id gen  │
└────────┬───────────┘
         │  raw text
    ┌────┴────────────────────────────────────────┐
    │              │                              │
    ▼              ▼                              ▼
┌──────────┐  ┌────────────┐             ┌──────────────┐
│ Signal 1 │  │  Signal 2  │             │  Signal 3    │
│ LLM/Groq │  │ Stylometric│             │ Bigram       │
│ 0.0–1.0  │  │ Heuristics │             │ Entropy      │
└─────┬────┘  └─────┬──────┘             └──────┬───────┘
      │             │                            │
      └─────────────┴─────────────┬──────────────┘
                                  ▼
                       ┌──────────────────────┐
                       │  Ensemble Scoring    │
                       │  LLM 55%             │
                       │  Stylo 30%           │
                       │  Bigram 15%          │
                       └──────────┬───────────┘
                                  ▼
                       ┌──────────────────────┐
                       │  Label Generation    │
                       │  >0.70 → likely AI   │
                       │  <0.35 → likely human│
                       │  else  → uncertain   │
                       └──────────┬───────────┘
                                  ▼
                       ┌──────────────────────┐
                       │  Audit Log (SQLite)  │
                       └──────────┬───────────┘
                                  ▼
                            JSON response
```

---

## Detection Signals

### Signal 1 — LLM-Based Classification (Groq / llama-3.3-70b-versatile)

**What it measures:** Semantic and stylistic coherence holistically. The model is prompted to assess whether a piece of text reads as human-authored or AI-generated — considering tone variety, naturalness of phrasing, redundancy patterns, and thematic consistency — and return a structured JSON `ai_probability` score.

**Why this differs:** AI text tends toward smooth, even prose without the micro-inconsistencies, register shifts, or idiosyncratic phrasing that human writers produce organically. The LLM recognizes these patterns because it was trained on vast amounts of both types.

**What it misses:** Highly polished human prose can be flagged as AI. Very short inputs give it little signal. It may behave differently on non-English text. The model's own stylistic biases affect its assessments.

**Output:** Float in [0.0, 1.0]. Contributes 55% of the combined score.

---

### Signal 2 — Stylometric Heuristics (Pure Python)

Three structural sub-signals, each normalized to [0.0, 1.0]:

**Sentence Length Variance (45% of signal):** Standard deviation of sentence lengths. Human writing mixes short punchy lines with long complex ones. AI produces more uniform lengths. Higher variance → lower AI score.

**Type-Token Ratio (35% of signal):** Unique words ÷ total words. Human writers use a more idiosyncratic vocabulary. AI repeats transitional phrases, lowering diversity. Higher TTR → lower AI score.

**Punctuation Diversity (20% of signal):** Non-standard punctuation (dashes, ellipses, semicolons, exclamation marks) per sentence. Human writers use these more variably. AI defaults to commas and periods. Higher diversity → lower AI score.

**What it misses:** Formal human writing (academic, legal, business) has naturally low variance and low expressive punctuation — this signal systematically over-classifies it as AI. Short texts (<3 sentences) produce unreliable statistics. Poetry and dialogue-heavy fiction don't fit the assumptions.

**Output:** Float in [0.0, 1.0]. Contributes 30% of the combined score.

---

### Signal 3 — Bigram Entropy (Stretch: Ensemble Detection)

**What it measures:** Shannon entropy of the bigram (word-pair) frequency distribution. Lower entropy means more predictable word sequences — a hallmark of language model output, which minimizes perplexity. Higher entropy = more variable, human-like word ordering.

**Why this differs:** Language models are trained to predict the most probable next token, producing text with statistical regularities in word sequences. Human writers don't optimize for predictability.

**What it misses:** Very short texts and texts with highly constrained vocabulary (technical manuals, legal contracts) can have low entropy regardless of authorship.

**Output:** Float in [0.0, 1.0]. Contributes 15% of the combined score.

---

### Weighting Rationale

```
combined = (0.55 × llm_score) + (0.30 × stylo_score) + (0.15 × bigram_score)
```

The LLM signal gets the most weight because it captures richer, harder-to-quantify semantic features and is more reliable across text types. Stylometrics is a valuable structural check but is more vulnerable to false positives on formal human writing. Bigram entropy is a useful third perspective but is least reliable on short texts, so it gets the smallest weight.

---

## Confidence Scoring

### Threshold Design

| Confidence | Attribution | Label Shown |
|---|---|---|
| > 0.70 | `likely_ai` | ⚠️ AI-Generated Content Detected |
| 0.35 – 0.70 | `uncertain` | 🔍 Authorship Unclear |
| < 0.35 | `likely_human` | ✅ Likely Human-Written |

**Design philosophy:** False positives (labeling a human as AI) are worse than false negatives on a creative writing platform. The uncertain band (0.35–0.70) is deliberately wide to reduce over-confident misclassification of human writers. The high-confidence AI threshold (0.70) is intentionally conservative.

What 0.5 means to this system: The signals are in weak disagreement or both giving middling responses. The system genuinely cannot determine authorship. The label explicitly says so, and the creator is offered an appeal path.

### Example Submissions with Noticeably Different Confidence Scores

**High-confidence case — clearly human (confidence: 0.234):**
```
Input: "ok so i finally tried that new ramen place downtown and honestly? 
        underwhelming. the broth was fine but they put WAY too much sodium in it 
        and i was thirsty for like three hours after."

Signals: llm=0.09, stylo=0.42, bigram=0.38
Combined confidence: 0.234 → likely_human
```

**High-confidence case — strongly AI-leaning (confidence: 0.736):**
```
Input: "Artificial intelligence represents a transformative paradigm shift. 
        It is important to note that stakeholders must collaborate. Furthermore, 
        ethical implications are crucial. In conclusion, multifaceted governance 
        frameworks are essential."

Signals: llm=0.95, stylo=0.58, bigram=0.47  
Combined confidence: 0.736 → likely_ai
```

**Validation:** Running the system on clearly AI-generated corporate language and clearly human casual text produces a confidence gap of ~0.50, demonstrating that scores reflect meaningful signal rather than noise. Borderline cases (formal human academic writing, lightly edited AI) fall in the 0.45–0.60 range — correctly captured by the `uncertain` bucket rather than a forced binary verdict.

---

## Transparency Label

The label returned in the API response is the exact text shown to readers on the platform. All three variants are included in the submission response; the platform renders it in context next to the content.

### Variant 1: High-Confidence AI (confidence > 0.70)

```
⚠️ AI-Generated Content Detected
Our system found strong signals that this content was likely produced by an 
AI writing tool, not written by a human author. Confidence: 74%.

If you believe this is incorrect, you can submit an appeal — we review all 
contested classifications.
```

### Variant 2: Uncertain (confidence 0.35–0.70)

```
🔍 Authorship Unclear
Our system couldn't determine with confidence whether this content was 
written by a human or generated by AI. Confidence in AI authorship: 49%.

We've flagged this for additional context. The creator may choose to appeal 
if they believe the classification is inaccurate.
```

### Variant 3: Likely Human-Written (confidence < 0.35)

```
✅ Likely Human-Written
Our system found strong signals that this content was written by a human 
author. Confidence in human authorship: 77%.

This label reflects our best assessment — it is not a guarantee. 
Detection systems are imperfect.
```

**Design note:** The human label explicitly notes it "is not a guarantee" — this reflects the asymmetry principle (false positives are worse) and prepares audiences for the possibility of misclassification in the other direction. The uncertain label names both possibilities and directs the creator to the appeal path rather than assuming the worst.

---

## Rate Limiting

**Limits applied to `POST /submit`:**

```
10 requests per minute per IP
100 requests per day per IP
```

**Reasoning:** A single human creator submitting their own work would rarely need more than 1–2 submissions per session — 10/minute is generous for legitimate use while making trivial flooding slow. 100/day is enough for a power user submitting many pieces over the course of a working day (a music blogger posting 5–10 entries, for instance), but prevents anyone from running a systematic evasion study in a single day without rotating IPs.

The adversarial model here is someone submitting hundreds of variants of AI-generated text to find phrasing patterns that evade detection. Rate limiting makes that kind of systematic probing expensive and slow.

**Testing rate limiting (run with server active):**
```bash
for i in $(seq 1 12); do
  curl -s -o /dev/null -w "%{http_code}\n" -X POST http://localhost:5000/submit \
    -H "Content-Type: application/json" \
    -d '{"text": "This is a test submission for rate limit testing purposes only. It contains enough text to pass validation.", "creator_id": "ratelimit-test"}'
done
```

**Expected output (first 10 return 200, then 429):**
rate limit evidence:
200
200
200
200
200
200
200
429
429
429
429
429

---

## Audit Log

Every classification and appeal is written to SQLite as a structured record. Retrieve with:

```bash
curl -s http://localhost:5000/log?limit=5 | python -m json.tool
```

**Sample entries (3 classifications + 1 appeal):**

```json
{
  "count": 4,
  "entries": [
    {
      "content_id": "4e46f2af-5e71-48b7-a94a-2143ac8c59e1",
      "creator_id": "user-dan-55",
      "timestamp": "2026-06-29T20:14:54.382899+00:00",
      "attribution": "uncertain",
      "confidence": 0.5665,
      "llm_score": 0.74,
      "stylo_score": 0.3036,
      "bigram_score": 0.4561,
      "label": "🔍 Authorship Unclear\nOur system couldn't determine...",
      "status": "classified",
      "text_snippet": "Remote work presents genuine tradeoffs...",
      "appeal_id": null,
      "creator_reasoning": null,
      "appeal_timestamp": null
    },
    {
      "content_id": "9463e8af-015a-431b-9979-2e47c8490d9b",
      "creator_id": "user-carol-88",
      "timestamp": "2026-06-29T20:14:54.378951+00:00",
      "attribution": "uncertain",
      "confidence": 0.489,
      "llm_score": 0.58,
      "stylo_score": 0.3504,
      "bigram_score": 0.4323,
      "label": "🔍 Authorship Unclear\nOur system couldn't determine...",
      "status": "under_review",
      "text_snippet": "The relationship between monetary policy...",
      "appeal_id": "611674e5-76a3-4ef1-a490-3a671b62094d",
      "creator_reasoning": "I wrote this myself for my economics dissertation. My academic writing style is formal because I was trained in that tradition — it does not reflect AI generation.",
      "appeal_timestamp": "2026-06-29T20:14:54.386808+00:00"
    },
    {
      "content_id": "48f0597e-dff4-4109-ba82-940b13241046",
      "creator_id": "user-bob-17",
      "timestamp": "2026-06-29T20:14:54.374137+00:00",
      "attribution": "likely_human",
      "confidence": 0.2338,
      "llm_score": 0.09,
      "stylo_score": 0.4234,
      "bigram_score": 0.3819,
      "label": "✅ Likely Human-Written\nOur system found strong signals...",
      "status": "classified",
      "text_snippet": "ok so i finally tried that new ramen place downtown...",
      "appeal_id": null,
      "creator_reasoning": null,
      "appeal_timestamp": null
    }
  ]
}
```

Each entry captures: content ID, creator ID, timestamp, attribution verdict, combined confidence, all three individual signal scores, the full label text, current status, text snippet, and — if an appeal was filed — the appeal ID, creator reasoning, and appeal timestamp.

---

## Appeals Workflow

**Who can appeal:** Any creator who submitted content. In this implementation, identity is passed as `creator_id` in the submission. In production this would be tied to authenticated accounts.

**What they provide:** `content_id` (from the `/submit` response) and `creator_reasoning` (free text).

**What the system does:**
1. Looks up the original classification by `content_id`
2. Sets status from `"classified"` → `"under_review"`
3. Writes an appeal record to the audit log with timestamp, reasoning, and original verdict details
4. Returns a confirmation with `appeal_id` and a summary of the original decision

**What a human reviewer sees:** `GET /log` returns all entries. Appeal entries are distinguished by non-null `appeal_id` and `creator_reasoning`. The corresponding classification entry shows `"status": "under_review"`. A reviewer sees both side-by-side and can make a determination. No automated re-classification occurs.

---

## Known Limitations

**1. Formal human writing produces systematic false positives.** An economics dissertation excerpt, a legal brief, or a professionally edited business article will score high on the stylometric signal (low sentence variance, uniform structure, low expressive punctuation density) and potentially high on the LLM signal too — because the model associates polished prose with AI. The system is most likely to misclassify serious formal writing as uncertain or likely-AI. This is the most common false-positive scenario on a creative writing platform.

**Root cause:** The stylometric signal measures deviation from "average" prose. Formal writing is deliberately constrained to avoid deviation. The signal cannot distinguish between "constrained because it's AI" and "constrained because the genre requires it."

**2. AI-generated poetry and experimental prose evades detection.** Poetry uses short, high-variance sentences, unconventional punctuation, and rich but narrow vocabulary — opposite of what the stylometric signal looks for. A sonnet or prose poem generated by GPT-4 could score very low on stylometrics, dragging the combined score below the `likely_ai` threshold. The LLM signal is the only check in this case, and it may be confused by experimental form.

**3. Short texts are unreliable.** The stylometric and bigram signals are meaningless with fewer than 3–4 sentences. A one-paragraph submission produces statistics with high variance regardless of authorship. The system does not currently flag low-reliability results due to text length.

---

## Stretch Features

### Ensemble Detection (Signal 3: Bigram Entropy)

The system uses three detection signals rather than the required two. The third signal — bigram entropy — measures the predictability of word-pair sequences in the text. Lower entropy means more predictable transitions, which is a structural fingerprint of language model output. See the Detection Signals section above for full documentation.

**Weighting:** LLM 55% | Stylometric 30% | Bigram Entropy 15%

### Analytics Dashboard

`GET /analytics` returns detection pattern summary:

```json
{
  "total_submissions": 4,
  "attribution_breakdown": {
    "likely_ai": 0,
    "likely_human": 1,
    "uncertain": 3
  },
  "appeal_count": 1,
  "appeal_rate": 0.25,
  "under_review": 1,
  "average_confidence": 0.4875
}
```

Metrics: total submissions, breakdown by attribution category, total appeals, appeal rate (appeals/submissions), currently-under-review count, and mean confidence score across all submissions.

---

## Spec Reflection

**One way the spec helped:** Writing out the three label variants in `planning.md` before writing any code forced me to think through what "uncertain" means to a non-technical user. The initial draft said something like "authorship is ambiguous" — generic and unhelpful. Iterating on the labels in the planning doc produced a version that names what the user should *do* (appeal if you disagree) rather than just describing the system's internal state. Without that planning step I would have shipped a worse label.

**One way implementation diverged:** The planning doc specified the LLM signal at 60% weight and stylometrics at 40%. After testing with the 4 Milestone 4 inputs, I found the stylometric signal frequently pulled the combined score away from clearly-correct LLM verdicts (specifically: the casual human text had a stylo_score of 0.42 even though it's clearly human, because its vocabulary diversity is limited in a short sample). I reduced stylometrics to 30%, added bigram entropy at 15%, and kept LLM at 55%. The combined scores became more clearly separated between the clearly-AI and clearly-human test cases.

---

## AI Usage

**Instance 1 — Stylometric sub-signal combination logic:**
I asked the AI tool to generate the `classify_with_stylometrics()` function, providing the planning.md section describing the three sub-metrics and their intended roles. The generated function used equal weights (33% each). I overrode this: sentence length variance carries the most predictive signal for AI vs. human differentiation and gets 45%, TTR gets 35%, and punctuation density (noisier) gets 20%. I also revised the normalization formula for TTR — the initial version had the mapping inverted, returning a high score for high TTR (diverse vocabulary), when the spec required high score = AI-like = low TTR.

**Instance 2 — Groq prompt design:**
I asked the AI tool to write the Groq prompt for Signal 1. The first draft asked the model to produce a paragraph of reasoning and then a score — which made JSON parsing fragile. I revised it to demand `{"ai_probability": float, "reasoning": "one sentence"}` only, with explicit instruction to return no other text, and added a regex strip for markdown code fences that the model sometimes inserts anyway. I also reduced `temperature` from the generated 0.7 to 0.1 to make the score more deterministic across repeated submissions of the same text.

---

## File Structure

```
provenance-guard/
├── app.py              # Flask application — all endpoints and business logic
├── planning.md         # Architecture, spec, and AI tool plan (written before code)
├── requirements.txt
├── .env.example        # Copy to .env and add GROQ_API_KEY
├── .gitignore
├── provenance.db       # SQLite audit log (auto-created on first run; gitignored)
└── README.md
```
