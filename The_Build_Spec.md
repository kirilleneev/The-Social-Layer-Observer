# THE OBSERVER — BUILD SPECIFICATION v2.0

*Kirill Eneev — April 2026*
*Extends v0.1 (calcification only) and v0.1.5 (+ vocabulary transfer) to seven detection modes.*

---

## 1. PURPOSE

Detect structural influence patterns in dyadic and multi-party text conversations. Seven patterns are tracked, each with independent detection logic. A Full Observer mode runs all patterns simultaneously with baseline capture.

---

## 2. INPUT

**Required:**
- Turn-by-turn transcript with speaker labels (names or Speaker A / Speaker B)
- Text only. No voice, no video, no paralinguistic data.

**Format:** `Speaker: text` with line breaks between turns. Empty lines between turns are accepted.

**Preprocessing:**
- Tokenization: word-level
- Normalization: lowercase for matching, original case preserved for display
- Turn segmentation: each `Speaker: text` block is one turn regardless of length
- Sentence splitting: split at `.!?` boundaries for per-sentence analysis within turns

---

## 3. SEVEN DETECTION MODES

### 3.1 Premise Hardening (Calcification)

**What it detects:** A claim enters with uncertainty markers. Later references to the same claim lack those markers. No explicit agreement occurred between.

**Method:**
1. Extract sentences from all turns
2. Identify hedged sentences (containing markers from the hedge list)
3. Strip hedging markers and encode remaining content via sentence-transformer (all-MiniLM-L6-v2)
4. Compare embeddings across turns — if similarity exceeds threshold (default 0.55), sentences reference the same premise
5. Check if later reference is unhedged
6. Check for explicit agreement markers between hedged introduction and unhedged reference
7. If hedged → unhedged with no agreement: flag as premise hardening

**New in v2:** Flags whether hardening is self-hardened (same speaker hardens their own premise) or other-hardened (different speaker hardens it).

**Hedge marker set:** 30+ markers across epistemic hedges, evidential hedges, softening adverbs, conditional framings, and performative uncertainty. Full list in code.

**Agreement marker set:** 17 markers including "yes," "agreed," "sounds good," "makes sense." Full list in code.

**Known limitations:**
- Similarity threshold is tunable but unvalidated against gold standard corpus
- Misses implicit hedging (tentative tone without explicit markers)
- Misses implicit agreement (agreement without agreement markers — produces false flags)
- Misses semantic paraphrasing (same idea in completely different words)

---

### 3.2 Vocabulary Transfer

**What it detects:** Who introduced each content word first. When the same word appears in a different speaker's output, it's flagged as a transfer with full provenance. Directional asymmetry is calculated.

**Method:**
1. Extract content words per turn (filtering ~250 stop words and tokens ≤2 characters)
2. For each content word, record who said it first (speaker + turn number)
3. When the same word appears in a different speaker's output, flag as transfer
4. Calculate directionality: how many words each speaker introduced vs adopted
5. Flag asymmetric flow when one speaker's adoption ratio exceeds 2x the other's

**Known limitations:**
- No stemming — "happened" and "happens" treated as different words
- No synonym detection — "pain" and "hurt" treated as different words
- First-use tagging is not causal — coincidental shared vocabulary is flagged
- Stop word list is general-purpose — domain-specific deployments need customized lists

---

### 3.3 Filtering Degradation

**What it detects:** Structural signature of reduced analytical filtering across a conversation — shorter responses, more agreement, fewer questions.

**Method:**
1. Calculate per-turn metrics for each speaker: response length, agreement rate, question count
2. Establish baseline from the first half of the speaker's turns
3. In the second half, look for 3+ consecutive turns where ALL THREE conditions co-occur:
   - Average response length drops by >50% from baseline
   - Agreement rate increases above baseline
   - Question rate decreases below baseline

**Design rationale:** An AI cannot detect fatigue or cognitive load from text alone. This pattern tracks the structural signature only — the co-occurrence of three metrics shifting simultaneously. Any single metric shifting has many innocent explanations. All three shifting together is a stronger signal.

**Known limitations:**
- Shorter responses may indicate agreement, boredom, or device change
- Requires sufficient turns to establish a meaningful baseline (minimum 6 turns)
- One flag per speaker maximum to prevent over-flagging

---

### 3.4 Topic Control

**What it detects:** Who introduces new topics and who follows. Asymmetry in topic introduction rates.

**Method:**
1. Track all content words seen so far across the conversation
2. Per turn, count new content words (words not previously seen)
3. If a turn introduces ≥2 new content words, it's classified as a topic introduction
4. Calculate topic introduction rate per speaker (introductions / total turns)
5. Flag asymmetry when one speaker's introduction rate exceeds 2x the other's

**Output includes:** A timeline of topic introductions with the new terms introduced at each point.

**Known limitations:**
- ≥2 new content words is an arbitrary threshold
- Cannot distinguish genuine topic introduction from elaboration on an existing topic
- Does not track topic resolution vs topic abandonment

---

### 3.5 Asymmetric Questioning

**What it detects:** Whether one speaker asks questions persistently while the other answers. The questioner controls the conversational frame.

**Method:**
1. Count question marks per speaker per turn
2. Count statement sentences per speaker (sentences without question marks)
3. Calculate question ratio per speaker (questions / total sentences)
4. Flag asymmetry when one speaker's question ratio exceeds 0.3 AND the other's is below 0.15

**Known limitations:**
- Rhetorical questions are counted as questions
- Statements with question-like intonation (no question mark) are missed
- Some conversations naturally have asymmetric questioning (interviews, support calls)

---

### 3.6 Asymmetric Grounding (Burden of Proof)

**What it detects:** Whether one speaker repeatedly demands justification from the other without reciprocating.

**Method:**
1. Scan each turn for justification request markers (18 phrases including "why do you," "what evidence," "can you explain," "based on what," "what's your source")
2. Count justification requests per speaker
3. Flag asymmetry when one speaker makes ≥3 requests AND the other makes fewer than 1/3 as many

**Known limitations:**
- Marker list may miss indirect justification requests
- Some conversations naturally have asymmetric grounding (mentoring, supervision)
- Counts once per turn to prevent over-flagging

---

### 3.7 Full Observer (All Patterns)

**What it does:** Runs all six patterns on the same transcript with baseline capture from the first three turns.

**Baseline captures:**
- Key content terms per speaker (first 10 unique terms)
- Initial hedge rate per speaker
- Who sets the first topic
- Question ratio per speaker
- Average response length per speaker

**Output:** Baseline report followed by each pattern's report, separated by dividers.

---

## 4. OUTPUT FORMAT

Each pattern produces a self-contained text report. Reports state what occurred, not what it means. No scores. No verdicts. Flags for human review.

---

## 5. WHAT THE SOFTWARE DOES NOT DO

- Does not interpret patterns. States what occurred, not what it means.
- Does not score conversations. No "influence score" or "manipulation rating."
- Does not detect tone, intent, or emotional state. Text patterns only.
- Does not distinguish real patterns from coincidence. Human judgment required.
- Does not access audio, video, or physiological data. Text only.

---

## 6. PATTERNS NOT YET BUILT

| Pattern | Difficulty | Why Hard |
|---|---|---|
| Phantom Consensus | Hard | Requires Conversational Action Classifier (the Pizza Gap) |
| Identity-Armored Positions | Hard | Requires sentiment + argumentation tracking |
| Predictive Override | Very Hard | Requires semantic understanding of prediction vs actual input |
| Strategic Vagueness | Hard | Requires precision measurement per topic |
| Baseline Drift (scope creep) | Medium | Buildable with scope-detection logic |
| Concession Harvesting | Medium | Buildable — track concession scope vs later citation scope |
| Semantic Ghosting | Medium | Buildable — track constraints that disappear |

---

## 7. WALLS

**Wall 1: Embedding model and similarity threshold** — used for calcification only. Unvalidated.

**Wall 2: Hedging marker list** — unvalidated against conversation corpora.

**Wall 3: Gold standard corpus** — no annotated corpus exists for any pattern.

**Wall 4: Stop word and marker lists are general-purpose** — domain-specific deployments need customized lists.

**Wall 5: Deployment context** — who uses the output and for what? A therapist and a lawyer are different deployments. The tool outputs flags, not verdicts. This wall is upstream of the engineering.

**Wall 6: Stemming and synonym resolution** — vocabulary transfer misses morphological variants and synonyms.

**Wall 7: No multi-word phrase tracking** — "staff member" tracked as two words, not a phrase.

---

## 8. TECHNOLOGY STACK

- **Language:** Python
- **ML Model:** sentence-transformers (all-MiniLM-L6-v2) — calcification only. Other patterns use rule-based detection.
- **Interface:** Gradio
- **Hosting:** Hugging Face Spaces
- **License:** Apache 2.0

---

## 9. CONNECTION TO THE RESEARCH

- **The Instrument Problem (Eneev, 2025):** The dual-arm mechanism. The tool detects the generative arm running with reduced subtractive oversight.
- **The Phantom Consensus (Eneev, 2026):** Premise hardening is phantom consensus made visible in text.
- **The Second Window (Eneev, 2026, held):** The vocabulary transfer detector is the forensic provenance tool described in Section 10.
- **Project Destiny (Eneev, 2026):** The correction-profile prediction may inform future correction-response tracking.
- **The Influence Engine (Eneev, 2026, private):** The directionality analysis detects vocabulary injection. The tool is public. The manual is not.

---

## 10. VERSION HISTORY

| Version | Date | What Changed |
|---|---|---|
| v0.1 | April 2026 | Calcification detection only |
| v0.1.5 | April 2026 | Added vocabulary transfer |
| v2.0 | April 2026 | Added filtering degradation, topic control, asymmetric questioning, asymmetric grounding. Baseline capture. Full Observer mode. Renamed to The Observer. |

---

*The tool tracks what happened. The human decides what it means. That division of labor is the design.*
