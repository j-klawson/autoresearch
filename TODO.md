# LLM SOC Oracle — TODO

## rlsecd Features

*Implemented in rlsecd repo. Phase 0 of ROADMAP.*

- [ ] Add Head 5 (action_quality) to SecurityAgent
  - Extend `HEAD_NAMES` tuple in `src/rlsecd/agents/security_agent.py`
  - Update checkpoint schema and add migration from 5-head to 6-head
  - Output range [0,1], initialize to predict 0.5
- [ ] SituationReporter class in `src/rlsecd/feedback/reporter.py`
  - Composes natural language from: prediction confidence, action taken, session stats, IP history, temporal context
  - Method: `report_action(event, session_state, prediction, action) -> str`
  - Include relevant Head 0-4 predictions in the report for full context
- [ ] LLMFeedbackClient ABC in `src/rlsecd/feedback/client.py`
  - `async query(situation_report: str) -> FeedbackResponse`
  - FeedbackResponse dataclass: `action_appropriate` (bool), `confidence` (float), `reasoning` (str), `reward` (float)
  - Timeout and retry logic, rate limiting
- [ ] ClaudeAPIClient implementation (anthropic SDK, Haiku for cost)
  - Structured output via tool_use
  - System prompt with SOC analyst role
- [ ] FeedbackRewardAdapter in `src/rlsecd/feedback/adapter.py`
  - Maps `FeedbackResponse.reward` -> Head 5 target
  - Confidence gating: skip update if LLM confidence below threshold
- [ ] Wire into `daemon.py` `_handle_event`
  - After non-pass actions, compose situation report, query LLM asynchronously
  - Feed response to `agent.update()` for Head 5
- [ ] CLI flags: `--llm-feedback`, `--llm-model`, `--llm-confidence-threshold`
- [ ] Config schema additions for `[feedback]` section

## security-gym Features

*Implemented in security-gym repo. Phase 2 of ROADMAP.*

- [ ] Expose per-action consequence metrics in `info` dict
  - `service_impact_score`: impact on legitimate service from action
  - `blocked_benign_count`: false positive count for block actions
- [ ] Add ground truth action quality to EventStore
  - Was blocking correct given hindsight? (retrospective label)
  - Requires tracking outcomes after action is taken

## autoresearch Features

*Implemented in this repo. Phases 1-3 of ROADMAP.*

### Phase 1: Baseline Validation
- [ ] Design prompt template for SOC analyst oracle
  - System prompt, situation report format, structured output schema
- [ ] Shared prompt template versioning (cross-project)

### Phase 3: Fine-Tuning Pipeline
- [ ] Design fine-tuning variant of `train.py`
  - Replace GPT-from-scratch with LoRA/QLoRA on pre-trained model
  - Use HuggingFace transformers + PEFT for LoRA
- [ ] Security-domain evaluation metric
  - Replace BPB with classification accuracy on held-out ground truth
  - Per-action-type breakdown
- [ ] Dataset loader for `(situation_report, structured_feedback)` JSONL
  - Tokenize situation reports as input, structured feedback as target
- [ ] Update `program.md` for security fine-tuning workflow
  - New metrics, new evaluation criteria, domain-specific experiment ideas

## Cross-Project

- [ ] JSONL schema for training data interchange
  ```json
  {
    "situation_report": "...",
    "llm_feedback": {
      "action_appropriate": true,
      "confidence": 0.92,
      "reasoning": "...",
      "reward": 0.85
    },
    "ground_truth": {
      "was_correct": true,
      "consequence_reward": 0.9,
      "blocked_benign": 0
    },
    "metadata": {
      "dataset": "30d",
      "event_idx": 12345,
      "action": "block",
      "timestamp": "..."
    }
  }
  ```
- [ ] Shared prompt template versioning between rlsecd and autoresearch

## Dependency Order

```
rlsecd: Head 5 + SituationReporter + LLMFeedbackClient ABC
    └──> rlsecd: ClaudeAPIClient + wiring
            └──> autoresearch: prompt template design (Phase 1)
                    └──> security-gym: consequence metrics (Phase 2)
                            └──> cross-project: JSONL schema + data collection
                                    └──> autoresearch: fine-tuning pipeline (Phase 3)
```
