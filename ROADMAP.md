# LLM SOC Oracle — Roadmap

Fine-tune a local LLM to act as a SOC analyst oracle for rlsecd. The oracle evaluates rlsecd's security actions (block, throttle, alert, isolate) and provides structured feedback that trains a new prediction head (Head 5: action_quality). This is a GVF in Alberta Plan terms — the agent learns to predict the consequence of its own actions.

## Phase 0: rlsecd Foundation

**Goal:** Add the action-quality prediction head and feedback infrastructure to rlsecd. No autoresearch changes yet.

**rlsecd changes:**
- Add **Head 5 (action_quality)** to SecurityAgent's MLP learner
  - Extend `HEAD_NAMES` tuple, update checkpoint schema and migration logic
  - Output range [0,1]: 0 = bad action, 1 = perfect action
  - Starts untrained (predicts 0.5); learns from LLM feedback
- **SituationReporter** class (`src/rlsecd/feedback/reporter.py`)
  - Translates agent internal state + action + session context into natural language situation reports
  - Inputs: prediction confidence, action taken, session stats (auth attempts, duration, IPs), IP history, temporal context (time of day, day of week)
  - Method: `report_action(event, session_state, prediction, action) -> str`
  - Structured but human-readable output suitable as LLM prompt context
- **LLMFeedbackClient** ABC (`src/rlsecd/feedback/client.py`)
  - `async query(situation_report: str) -> FeedbackResponse`
  - `FeedbackResponse` dataclass: `action_appropriate` (bool), `confidence` (float), `reasoning` (str), `reward` (float [0,1])
  - Retry logic, timeout handling, rate limiting
- **FeedbackRewardAdapter** (`src/rlsecd/feedback/adapter.py`)
  - Maps `FeedbackResponse.reward` to Head 5 target value
  - Confidence gating: skip update if LLM confidence below configurable threshold
  - Exponential decay for stale feedback (if action was N events ago)
- **Action-triggered query logic** in `daemon.py` `_handle_event`
  - After non-pass actions (block, throttle, alert, isolate), compose situation report and query LLM
  - Asynchronous: don't block event processing on LLM response
  - Feed response to `agent.update()` for Head 5 when it arrives
- **CLI flags:** `--llm-feedback`, `--llm-model`, `--llm-confidence-threshold`
- **Config schema** additions for `[feedback]` section

**Cross-references:**
- Extends rlsecd Phase 3 (online continual learning)
- Feeds into rlsecd Phase 6 (learned detection threshold) — Head 5 predictions can inform threshold decisions
- SecurityAgent head schema at `src/rlsecd/agents/security_agent.py`

## Phase 1: Baseline Oracle

**Goal:** Validate the approach using Claude API as the LLM oracle before investing in fine-tuning.

**autoresearch + rlsecd changes:**
- **ClaudeAPIClient** (`LLMFeedbackClient` subclass) using anthropic SDK
  - Uses Claude Haiku for cost efficiency during data collection
  - Structured output via tool_use for reliable JSON parsing
- **Prompt template** design
  - System prompt: SOC analyst role, security domain context
  - Situation report as user message
  - Structured output schema: `{action_appropriate, confidence, reasoning, suggested_action, reward}`
- **Validation experiment:** run rlsecd + Claude oracle against security-gym 7d dataset
  - Collect `(situation_report, llm_response, ground_truth)` triples
  - Ground truth from security-gym's known labels
  - Measure: LLM judgment accuracy, false positive/negative rates, response latency
  - Target: >90% agreement with ground truth on action appropriateness
- **No fine-tuning yet** — pure evaluation of frontier model judgment

## Phase 2: Dataset Collection

**Goal:** Build a high-quality training corpus for fine-tuning.

**All three projects involved:**

**rlsecd:**
- Run rlsecd + Claude oracle against all 4 security-gym datasets (7d, 30d, 90d, 365d)
- Log every `(situation_report, llm_response, ground_truth_outcome)` triple
- Ground truth = did the action actually help? security-gym knows via `consequence_reward`

**security-gym:**
- Expose per-action consequence metrics in `info` dict
  - `service_impact_score`: how much did the action affect legitimate service
  - `blocked_benign_count`: false positive count for block actions
- Add ground truth action quality to EventStore (was blocking correct given hindsight?)

**autoresearch:**
- **JSONL schema** for training data interchange:
  ```json
  {"situation_report": "...", "llm_feedback": {...}, "ground_truth": {...}, "metadata": {...}}
  ```
- Correct LLM labels where ground truth disagrees (prefer ground truth)
- Dataset statistics: class balance, confidence distribution, action type breakdown
- Train/val/test split (80/10/10) stratified by action type and outcome

**Expected corpus size:** ~5,000-10,000 action evaluations across all datasets

## Phase 3: Fine-Tuning Pipeline

**Goal:** Adapt autoresearch for supervised fine-tuning of a small pre-trained model on security feedback data.

**autoresearch changes:**
- **Replace train.py's GPT-from-scratch** with LoRA/QLoRA fine-tuning
  - Base model: small pre-trained model (e.g., Llama 3.2 1B or similar)
  - LoRA rank 16-64, targeting attention + MLP projections
  - QLoRA (4-bit quantization) to fit on single GPU
- **Evaluation metric:** classification accuracy on held-out security-gym ground truth
  - Replace BPB with: action_appropriate accuracy, reward MAE, calibration
  - Per-action-type breakdown (block accuracy vs alert accuracy etc.)
- **Dataset loader** for `(situation_report, structured_feedback)` JSONL
  - Tokenizes situation reports as input, structured feedback as target
  - Handles the structured output format (JSON generation)
- **Autonomous experiment loop** (adapt existing autoresearch loop):
  - Try fine-tuning configs (LoRA rank, learning rate, epochs, base model)
  - Keep/discard based on validation accuracy (not BPB)
  - 5-minute time budget still applies per experiment
- **program.md** updated for security fine-tuning workflow
  - New metrics, new evaluation criteria
  - Domain-specific experiment ideas

## Phase 4: Closed Loop

**Goal:** Deploy the fine-tuned model as the oracle and establish the improvement cycle.

- Deploy fine-tuned model as **local oracle** (replaces Claude API calls)
  - Quantized inference (GGUF/AWQ) for CPU or single-GPU deployment
  - rlsecd's `LLMFeedbackClient` subclass for local model
- rlsecd trains Head 5 on fine-tuned model's feedback
- **Periodic re-collection cycle:**
  1. Run improved rlsecd against security-gym
  2. Collect new `(situation_report, feedback, ground_truth)` triples
  3. Re-fine-tune on accumulated dataset
  4. Deploy updated model
- **Model collapse monitoring:**
  - Always validate against security-gym ground truth (held-out test set)
  - Track accuracy trend across fine-tuning generations
  - Alert if accuracy drops below Phase 1 baseline
- **Confidence gating:** only use LLM labels above configurable threshold
  - Below threshold: skip Head 5 update, log for human review

## Phase 5: Production

**Goal:** Run the full system without API dependencies.

- Fine-tuned model runs **locally on CPU** (no cloud API dependency)
  - Target: <100ms inference latency per situation report
  - GGUF quantization for CPU inference via llama.cpp or similar
- rlsecd in production mode gets action feedback without ground truth
  - Head 5 predictions enable **self-evaluation** when LLM is unavailable
  - If Head 5 predicts action_quality < threshold, log for human review
- **Continuous improvement cycle** with periodic re-tuning
  - Collect production situation reports + human analyst feedback
  - Re-fine-tune on mixed dataset (security-gym ground truth + production feedback)
- **Deployment:** systemd service alongside rlsecd, shared config

## Dependencies

```
Phase 0 (rlsecd only) ──> Phase 1 (+ autoresearch) ──> Phase 2 (+ security-gym)
                                                              │
                                                              v
                          Phase 4 (closed loop) <──── Phase 3 (fine-tuning)
                                │
                                v
                          Phase 5 (production)
```

Phases 0-2 can overlap with research ROADMAP Phase 2 (factorial experiments) — they use the same rlsecd + security-gym infrastructure but independent code paths.
