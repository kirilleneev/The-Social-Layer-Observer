# THE OBSERVER — BUILD SPECIFICATION v2.0

*Kirill Eneev — April 2026*
*Supersedes: Build Spec v0.1, Build Spec v0.1.5, Project Log v0.2*

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

**What it detects:** Who introduced each content word first. When the same word appears in a different speaker's output, it's flagged as a transfer with full provenance. Directional asymmetry is calculated. Four uptake atom metrics quantify the influence dynamics.

**Method:**
1. Extract content words per turn (filtering ~250 stop words and tokens ≤2 characters)
2. For each content word, record who said it first (speaker + turn number)
3. When the same word appears in a different speaker's output, flag as transfer
4. Calculate directionality: how many words each speaker introduced vs adopted
5. Flag asymmetric flow when one speaker's adoption ratio exceeds 2x the other's
6. Calculate uptake atom metrics (see below)

**Uptake Atom Metrics (v2.0):**

The smallest unit of conversational influence — the "uptake event" — is the two-sided bond between introduction and adoption. It requires both speakers. Below this level, influence is indistinguishable from noise. Four metrics are derived from counting uptake events:

- **Asymmetry Index:** Single number from -1 to +1. Positive = first speaker's vocabulary dominates. Negative = second speaker dominates. Zero = balanced. Formula: (A→B transfers - B→A transfers) / total transfers.
- **Uptake Density:** Transfers per 1000 words. High density = speakers heavily coupled to each other's vocabulary. Low density = parallel monologues.
- **Node Survival (Decay Rate):** Average number of turns an adopted word continues to appear after first adoption. Zero survival = surface politeness. High survival = word entered the adopter's thinking. Also reports count of persistent words (survived 3+ turns) and zero-survival words (used once, dropped).
- **Coefficient of Friction:** Per speaker, proportion of introduced words that the other speaker did NOT adopt. High friction = resistance. Low friction = one speaker absorbing everything the other says.

**Known limitations:**
- No stemming — "happened" and "happens" treated as different words
- No synonym detection — "pain" and "hurt" treated as different words
- First-use tagging is not causal — coincidental shared vocabulary is flagged
- Stop word list is general-purpose — domain-specific deployments need customized lists
- Node survival tracking uses a simple consecutive-turn method — gaps in usage reset the count

---

### 3.3 Filtering Degradation

**What it detects:** Structural signature of reduced analytical filtering — shorter responses, more agreement, fewer questions co-occurring.

**Method:**
1. Calculate per-turn metrics for each speaker: response length, agreement rate, question count
2. Establish baseline from the first half of the speaker's turns
3. In the second half, look for 3+ consecutive turns where ALL THREE conditions co-occur:
   - Average response length drops by >50% from baseline
   - Agreement rate increases above baseline
   - Question rate decreases below baseline

**Known limitations:**
- Shorter responses may indicate agreement, boredom, or device change
- Requires minimum 6 turns for meaningful baseline
- One flag per speaker maximum

---

### 3.4 Topic Control

**What it detects:** Who introduces new topics and who follows. Asymmetry in topic introduction rates.

**Method:**
1. Track all content words across the conversation
2. Per turn, count new content words (not previously seen)
3. If a turn introduces ≥2 new content words, classified as topic introduction
4. Calculate introduction rate per speaker
5. Flag asymmetry when one speaker's rate exceeds 2x the other's

**Known limitations:**
- ≥2 new content words is an arbitrary threshold
- Cannot distinguish topic introduction from elaboration
- Does not track topic resolution vs abandonment

---

### 3.5 Asymmetric Questioning

**What it detects:** Whether one speaker asks questions persistently while the other answers.

**Method:**
1. Count question marks per speaker
2. Count statements per speaker
3. Calculate question ratio
4. Flag asymmetry when one speaker's ratio exceeds 0.3 AND the other's is below 0.15

**Known limitations:**
- Rhetorical questions counted as questions
- Some conversations naturally have asymmetric questioning

---

### 3.6 Asymmetric Grounding (Burden of Proof)

**What it detects:** Whether one speaker repeatedly demands justification without reciprocating.

**Method:**
1. Scan turns for 18 justification request markers
2. Count requests per speaker
3. Flag asymmetry when one speaker makes ≥3 requests AND the other makes fewer than 1/3 as many

**Known limitations:**
- May miss indirect justification requests
- Some conversations naturally have asymmetric grounding

---

### 3.7 Full Observer (All Patterns)

Runs all six patterns with baseline capture from the first three turns. Baseline captures key terms, hedge rates, first topic setter, question ratios, and average response lengths per speaker.

---

## 4. OUTPUT DESIGN

### 4.1 Format

Each pattern produces a self-contained text report stating what occurred, not what it means. No scores. No verdicts. Flags for human review.

### 4.2 Socratic Output (Clinical Deployment)

For clinical supervision deployments, every flag should be output as a Socratic question for the supervisor, not a score or verdict:

- "At turn 12, the premise hardened. What was the client's body language at this moment?"
- "Between turns 5 and 8, the direction shifted. Was this the client's choice or a response to your framing?"
- "The client's 'maybe' at turn 3 disappeared by turn 7. Was there a moment of explicit agreement you recall?"

Useless to a manager. Invaluable to a supervisor. Build the ethics into the output format, not into a policy document. If the tool outputs scores, institutions will use it as a compliance weapon. Socratic questions prevent this structurally.

---

## 5. WHAT THE SOFTWARE DOES NOT DO

- Does not interpret patterns. States what occurred, not what it means.
- Does not score conversations. No "influence score" or "manipulation rating."
- Does not detect tone, intent, or emotional state. Text patterns only.
- Does not distinguish real patterns from coincidence. Human judgment required.
- Does not access audio, video, or physiological data.

---

## 6. PATTERNS NOT YET BUILT

| Pattern | Difficulty | Why Hard |
|---|---|---|
| Phantom Consensus | Hard | Requires Conversational Action Classifier (the Pizza Gap) |
| Identity-Armored Positions | Hard | Requires sentiment + argumentation tracking |
| Predictive Override | Very Hard | Requires semantic understanding of prediction vs actual input |
| Strategic Vagueness | Hard | Requires precision measurement per topic |
| Baseline Drift (scope creep) | Medium | Track scope of key terms across turns |
| Concession Harvesting | Medium | Track concession scope vs later citation scope |
| Semantic Ghosting | Medium | Track constraints that disappear from later reasoning |
| Trajectory Breaks | Hard | Detect when a speaker changes direction after another speaker's input. Requires trajectory representation (delta-maps between consecutive outputs). |
| Convergence vs Collapse | Hard | Distinguish agreement producing new signal from agreement just converging on what existed. Requires semantic novelty measurement. |
| Vocabulary-Stance Mismatch | Medium | Speaker changes position but keeps old vocabulary. Detectable by tracking stance words vs position words separately. |
| Looping Detection | Medium | Vocabulary diversity and semantic novelty declining across turns. Conversation collapsing into repetition. Buildable with existing architecture. |
| Negative Space Mapping | Very Hard | What wasn't said that should have been, bounded by formal expectation model. Requires adjacency pair and topic graph modeling. |

---

## 7. WALLS

**Wall 1: Embedding model and similarity threshold** — used for calcification only. Unvalidated.

**Wall 2: Hedging marker list** — unvalidated against conversation corpora.

**Wall 3: Gold standard corpus** — no annotated corpus exists for any pattern.

**Wall 4: Stop word and marker lists are general-purpose** — domain-specific deployments need customized lists.

**Wall 5: Deployment context** — who uses the output and for what? A therapist and a lawyer are different deployments. The tool outputs flags, not verdicts. This wall is upstream of the engineering.

**Wall 6: Stemming and synonym resolution** — vocabulary transfer misses morphological variants and synonyms.

**Wall 7: No multi-word phrase tracking** — "staff member" tracked as two words.

---

## 8. DOCUMENTED FAILURE CASES

**Failure 1: Surface form divergence.** "Maybe if we used it only on the lower sections?" → "Since we're using the zinc panels for the lower sections." Same proposition, different words. Embedding similarity falls below threshold. Required fix: predicate-argument extraction or local LLM for premise normalization.

**Failure 2: Synonym/hypernym drift.** "A cheaper alloy" → "the zinc cladding." Same entity, different name. Requires entity resolution on top of proposition extraction.

**v0.2 engineering note:** The original v0.2 spec proposed spaCy for predicate-argument extraction. Dependency parsing is brittle on messy spontaneous speech. Alternative: use a local LLM (e.g., Llama-3-8B-Instruct) prompted to extract [Subject] [Action] [Object] [Status: Hedged/Fact]. Handles synonym drift because the LLM already knows zinc is an alloy.

---

## 9. TECHNOLOGY STACK

- **Language:** Python
- **ML Model:** sentence-transformers (all-MiniLM-L6-v2) — calcification only. Other patterns use rule-based detection.
- **Interface:** Gradio
- **Hosting:** Hugging Face Spaces
- **License:** Apache 2.0

---

## 10. CONNECTION TO THE RESEARCH

- **The Instrument Problem (Eneev, 2025):** The dual-arm mechanism. The tool detects the generative arm running with reduced subtractive oversight.
- **The Phantom Consensus (Eneev, 2026):** Premise hardening is phantom consensus made visible in text.
- **The Second Window (Eneev, 2026, held):** The vocabulary transfer detector is the forensic provenance tool described in Section 10.
- **Project Destiny (Eneev, 2026):** The correction-profile prediction may inform future correction-response tracking.
- **The Influence Engine (Eneev, 2026, private):** The directionality analysis detects vocabulary injection. The tool is public. The manual is not.
- **The Conversational Atom (Eneev, 2026, pending):** Three independent systems converged on the uptake event as the minimum unit of conversational influence. The four uptake atom metrics are derived from this finding.

---

## 11. VERSION HISTORY

| Version | Date | What Changed |
|---|---|---|
| v0.1 | April 2026 | Calcification detection only |
| v0.1.5 | April 2026 | Added vocabulary transfer |
| v2.0 | April 2026 | Added filtering degradation, topic control, asymmetric questioning, asymmetric grounding. Baseline capture. Full Observer mode. Socratic output design. Uptake atom metrics (Asymmetry Index, Uptake Density, Node Survival, Coefficient of Friction). Renamed to The Observer. |

---

*The tool tracks what happened. The human decides what it means. That division of labor is the design.*
