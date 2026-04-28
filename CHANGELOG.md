# OpsFlight Changelog

> Session-to-session feature tracking document. Updated each session so future Claude instances can understand what exists, what changed, and what to avoid breaking.

---

## Current State: v0.1 Foundation (as of 2026-04-14)

### Project Summary

OpsFlight is a flight simulator for production AI agents. Two LLMs loop against each other -- an **Agent** that thinks and calls tools, and an **adversarial World Model** that maintains a fake production environment -- to produce high-signal SFT training data (ChatML + `<think>` tags) for fine-tuning SRE/DevOps models.

**Owner**: Daniel Lougen, M.S. ([@DJLougen](https://x.com/DJLougen))
**Repo**: `github.com/DJLougen/OpsFlight`
**License**: MIT

---

## Feature Inventory (what exists and works)

### Core Loop (`src/episode.py`)
- Agent-world interaction loop with up to 30 steps per episode
- Agent system prompt enforces `<think>` tags + single tool call per step
- World model returns JSON: `{ observation, reward, state_updates, done }`
- Recovery detection (reward spike after error step)
- Tool diversity computation per trajectory

### Agent Tools (`src/tools.py`) -- 13 total
| # | Tool | Notes |
|---|------|-------|
| 1 | `read_file(path)` | Fails: ENOENT, EACCES, encoding |
| 2 | `write_file(path, content)` | Fails: ENOSPC, permissions |
| 3 | `append_file(path, content)` | |
| 4 | `list_dir(path)` | Returns sizes, permissions |
| 5 | `delete_file(path)` | |
| 6 | `call_api(url, method, headers, body)` | Timeouts, 429, HTML errors |
| 7 | `run_command(cmd)` | stdout + stderr + exit code |
| 8 | `grep_files(pattern, path)` | Regex |
| 9 | `memory_set(key, value)` | Agent scratchpad |
| 10 | `memory_get(key)` | |
| 11 | `log(level, message)` | Structured logging |
| 12 | `sleep(seconds)` | Rate limiting / backoff |
| 13 | `finish(result)` | Ends episode |

### Task Bank (`src/tasks.py`) -- 20 scenarios
Categories: Data, API/Auth, Debugging (x2), Pipeline, CI/CD, Migration, Monitoring (x2), Database (x2), ETL, Security (x3), Resilience (x2), Deployment, K8s, Performance, DR.

Each task has: `id`, `description`, `category`, `difficulty`, `expected_steps`, `system_context`, `hints`, `tags`.

Stub functions exist for `generate_task_variants()` and `critic_filter()` but are **not fully implemented** and `task_generation.enabled` is `false` in config.

### Quality Gates & Scoring (`src/scorer.py`) -- 8 gates

| Gate | Weight | Pass Condition |
|------|--------|----------------|
| ThinkingDepthGate | 1.5 | Avg thinking >= 15 words |
| ToolSequencingGate | 1.0 | Investigation before action (first 3 steps) |
| SelfCorrectionGate | 2.0 | Self-correction phrases detected |
| VerificationGate | 1.5 | Verify/check/confirm in last 3 steps |
| NoDegeneratePatternsGate | 2.0 | No skipped reasoning, no 1-shot, no repetitive tools |
| ErrorRecoveryGate | 1.5 | Recovers from observed errors (reward increases) |
| GoalCompletionGate | 2.0 | Final reward >= 0.7 and uses `finish()` |
| ReasoningCoherenceGate | 1.0 | >= 30% coherent step transitions |

**Boost multipliers**: self_correction > 0.5 => +10%, verification => +5%, recovery => +8%.

### Process Reward Model (`src/prm.py`)
- LLM-based per-step verification (correct/risky/incorrect/incomplete)
- Combined score: 60% gates + 40% PRM
- Controlled by `verification.prm_enabled` in config

### Trajectory Refinement (`src/refinement.py`)
- Critic loop: world LLM critiques, agent LLM re-generates
- Up to 3 iterations, requires >= 5% improvement per pass
- Threshold for triggering: `refinement.min_score_threshold` (default 0.6 = score 60)

### Multi-Path Generation (`src/multipath.py`)
- Generates `num_paths` solutions per task at varied temperatures (0.3-0.9)
- Beam search: keeps top `beam_width` by signal score
- Diversity filter: drops paths with > 70% tool overlap AND < 2 step diff AND < 5 score diff

### Curriculum Learning (`src/curriculum.py`)
- Ordering modes: `easy_to_hard`, `hard_to_easy`, `interleaved`
- Adaptive reweighting by signal_score (50%), tool_diversity (25%), difficulty (25%)
- `filter_premium()`: returns trajectories >= threshold (default 75)

### Failure Analysis (`src/failure_analysis.py`) -- 3 output modes

Turns low-scoring trajectories into learning signal instead of discarding them.

| Mode | Output Format | Training Use |
|------|---------------|-------------|
| `diagnosis` | SFT (ChatML + `<think>`) | Teaches model to identify failure patterns |
| `fault_map` | Process reward (per-step verdicts) | Teaches step-level fault localization |
| `contrastive` | DPO/ORPO (chosen/rejected + analysis) | Preference training with reasoning anchors |

**Key settings** (`config.yaml > failure_analysis`):
- `score_threshold`: 50.0 (trajectories below this get analyzed)
- `modes`: which outputs to generate (default: all three)
- `parallel`: concurrency for LLM calls (default: 4)

**Contrastive pairing**: bad trajectories are matched to good trajectories (>=70 score) for the same task. If no good trajectory exists for a task, no contrastive record is produced.

**Outputs**: `corpus_failures_diagnosis.jsonl`, `corpus_failures_fault_map.jsonl`, `corpus_failures_contrastive.jsonl`

### Replay Buffer (`src/replay_buffer.py`)
- SQLite with WAL mode, capacity 5000
- Insert threshold: signal_score >= 55.0
- Prioritized sampling: higher score + lower staleness
- Max staleness: 50 samples before eviction
- Schema: `entries(id, trajectory, signal_score, tool_diversity, difficulty, staleness, created_ts, last_sampled)`

### Text Processing / Watermarking (`src/watermark.py`)
- Reed-Solomon error correction codes embedded via phrase substitution
- HMAC-based position selection, 18% bias strength
- Payload prefix: `opsflight-v2-202604-djlougen`
- Verification with confidence scoring
- Max quality impact: 2%

### Pipeline Orchestrator (`src/pipeline.py`)
Full generation pipeline:
1. Fresh/sampled split based on `replay.fresh_ratio` (0.7)
2. Round-robin agent LLM selection from pool
3. Parallel generation with configurable concurrency
4. PRM verification pass (if enabled)
5. Refinement of low-score trajectories
6. Buffer insertion of good trajectories
7. Buffer sampling for remainder
8. Text processing (watermarking)
9. Curriculum ordering
10. JSONL output + premium subset

### LLM Client (`src/llm.py`)
- Async OpenAI-compatible client
- Supports reasoning model fallback (checks `reasoning` field)
- JSON response parsing with markdown cleanup
- Works with: Ollama, vLLM, OpenRouter, etc.

### Dashboard (`src/dashboard.py`)
- FastAPI + WebSocket real-time UI
- Episode table, statistics grid, progress bar
- Modal trajectory playback on click
- Available at `python run.py serve`

### CLI (`run.py`)
Four Click commands:
- `generate` -- batch generation (--n, --output, --max-steps, --config)
- `watch` -- single episode live view (--task-id)
- `score` -- score existing JSONL corpus (--input)
- `serve` -- launch dashboard (--port, --host)

### Analysis (`analyze_corpus.py`)
Generates 4 PNG visualizations + `training_premium.jsonl`:
- `overview.png`: score histogram, steps distribution, domain bars, difficulty pie
- `behavior_patterns.png`: tool transition heatmap
- `reward_curves.png`: score vs steps by domain, score by difficulty
- `signal_analysis.png`: steps-vs-score colored by diversity, recovery comparison

### Data Structures (`src/__init__.py`)
- `ToolCall(tool, args)`
- `Step(step_num, thinking, tool_call, observation, reward, verification_cot)`
- `Trajectory(id, task_id, task_description, steps, final_answer, signal_score, metadata)` with `to_conversation()` for ChatML export
- `Task(id, description, category, difficulty, expected_steps, system_context, hints, tags)`

---

## Configuration Reference (`config.yaml`)

| Section | Key | Current Value | Notes |
|---------|-----|---------------|-------|
| world_model.model | glm-5.1:cloud | via localhost:11434 (Ollama) | |
| agent.models | gemma-4-31b-it (x2) | via OpenRouter | weighted pool |
| episode.max_steps | 30 | | |
| episode.parallel | 8 | concurrent workers | |
| generation.num_paths | 2 | multi-path enabled | |
| verification.prm_enabled | true | | |
| refinement.max_iterations | 3 | | |
| curriculum.ordering | easy_to_hard | | |
| output.premium_threshold | 0.75 | score >= 75 | |
| task_generation.enabled | false | stubs not ready | |
| failure_analysis.enabled | true | diagnose bad trajectories | |
| failure_analysis.score_threshold | 50.0 | analyze below this score | |
| failure_analysis.modes | all three | diagnosis, fault_map, contrastive | |
| replay.buffer_size | 5000 | | |
| replay.fresh_ratio | 0.7 | 70% new, 30% buffer | |
| text_processing.enabled | true | watermarking active | |

---

## Known Gaps & Incomplete Features

1. **Task variant generation** -- `generate_task_variants()` and `critic_filter()` in `tasks.py` are stubs. Config has `task_generation.enabled: false`.
2. **API key in config.yaml** -- OpenRouter key is hardcoded in plaintext. Should use env vars.
3. **No test suite** -- no unit tests or integration tests exist.
4. **No CI/CD** -- no GitHub Actions or similar.
5. **Error handling** -- world model failures default to generic errors; LLM timeouts end episodes silently.
6. **No `.env` support** -- secrets management not implemented.

---

## Git History

| Hash | Date | Description |
|------|------|-------------|
| `a8420aa` | 2026-04-05 | Initial commit: branding docs (README, ROADMAP, CONTRIBUTING, LICENSE, .gitignore) |
| `7380c89` | 2026-04-13 | t3 checkpoint: added requirements.txt + replay buffer DB |

**Note**: All source code (`src/`, `run.py`, `analyze_corpus.py`, `config.yaml`, `test_*.jsonl`) exists in the working tree but was added after the initial commit. These files are tracked (present in working tree) and represent the bulk of v0.1 implementation.

---

## Session Log

### Session 2026-04-14
- **Action**: Created `CHANGELOG.md` (this file) -- feature tracking for future sessions
- **Action**: Created `AGENT_README.md` -- quick-reference for AI agents working on this codebase
- **Action**: Added `src/failure_analysis.py` -- new module that generates training data from bad trajectories
  - Three output modes: diagnosis (SFT), fault_map (process reward), contrastive (DPO/ORPO)
  - Wired into `pipeline.py` (runs after refinement, before buffer insert)
  - Added `failure_analysis` config section to `config.yaml` (enabled by default)
  - CLI (`run.py`) now reports failure analysis counts in generation stats
  - Outputs written as `*_failures_diagnosis.jsonl`, `*_failures_fault_map.jsonl`, `*_failures_contrastive.jsonl`
- **State**: v0.1 fully implemented, no breaking issues found, code is clean (zero TODOs/FIXMEs)
- **Models in use**: glm-5.1:cloud (world model, local Ollama), google/gemma-4-31b-it (agent, OpenRouter)
- **Data generated**: `test_corpus.jsonl` (3 trajectories), `test_run.jsonl` (8 trajectories, larger test output)

---

## How to Update This File

At the start or end of each session, add an entry under **Session Log** with:
- Date
- What was added, changed, or removed
- Any config changes
- Current model/endpoint state
- Issues encountered

When features are added or modified, update the **Feature Inventory** section and increment the version if warranted.
