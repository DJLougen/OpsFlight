# OpsFlight -- Agent Quick Reference

> This document is for AI coding agents (Claude, etc.) working on this codebase. It gives you the fastest path to understanding the project without reading every file.

---

## What Is This?

OpsFlight generates synthetic SRE/DevOps training data by running two LLMs against each other:

```
Task Bank (20 scenarios)
       |
       v
  Agent LLM  <--->  World Model LLM
  (thinks +          (simulates prod
   calls tools)       environment)
       |
       v
  Scored Trajectory (JSONL, ChatML + <think> tags)
```

The output is fine-tuning data for production AI agents (compatible with Unsloth, Axolotl, llama.cpp).

---

## File Map

```
OpsFlight/
├── run.py                  # CLI entry point (generate, watch, score, serve)
├── config.yaml             # All configuration (models, thresholds, features)
├── analyze_corpus.py       # Corpus analysis + visualization
├── requirements.txt        # Python deps (openai, fastapi, numpy, etc.)
│
├── src/
│   ├── __init__.py         # Data classes: ToolCall, Step, Trajectory, Task
│   ├── tools.py            # 13 agent tools + parser + diversity calc
│   ├── tasks.py            # 20 task definitions + stub generators
│   ├── episode.py          # Core agent-world loop (run_episode)
│   ├── llm.py              # Async OpenAI-compatible LLM client
│   ├── scorer.py           # 8 quality gates + weighted scoring
│   ├── prm.py              # Process Reward Model (LLM-as-judge per step)
│   ├── refinement.py       # Critic-based trajectory improvement loop
│   ├── multipath.py        # Multi-path generation + beam search
│   ├── curriculum.py       # Easy-to-hard ordering + adaptive reweighting
│   ├── replay_buffer.py    # Prioritized SQLite buffer (WAL mode)
│   ├── failure_analysis.py # Learn from bad trajectories (diagnosis, fault map, contrastive)
│   ├── watermark.py        # Reed-Solomon watermarking via phrase substitution
│   ├── pipeline.py         # Full orchestration pipeline (ties everything together)
│   └── dashboard.py        # FastAPI + WebSocket live dashboard
│
├── CHANGELOG.md            # Session-to-session feature tracking (READ THIS FIRST)
├── ROADMAP.md              # v0.1 -> v1.0 planned features
├── README.md               # Public-facing project docs
├── CONTRIBUTING.md         # Contribution guidelines
├── LICENSE                 # MIT
│
├── .opsflight_buffer.db    # SQLite replay buffer (auto-created)
├── test_corpus.jsonl       # Sample trajectories (3 examples)
└── test_run.jsonl          # Larger test output
```

---

## Architecture at a Glance

### Data Flow

```
config.yaml
    |
    v
Pipeline (src/pipeline.py)
    |
    |-- Loads agent LLM pool (round-robin, weighted)
    |-- Loads world model LLM
    |
    |-- For each task:
    |     |-- run_episode() or MultiPathGenerator.generate()
    |     |-- Scorer.score_and_update()  (8 quality gates)
    |     |-- GenerativePRM.verify()     (if prm_enabled)
    |     |-- TrajectoryRefiner.refine() (if score < threshold)
    |     |-- ReplayBuffer.insert()      (if score >= 55)
    |
    |-- FailureAnalyzer.analyze()  (if enabled, score < 50)
    |     |-- diagnosis: "what went wrong" post-mortem (SFT)
    |     |-- fault_map: per-step verdicts (process reward)
    |     |-- contrastive: bad/good pairs (DPO/ORPO)
    |
    |-- ReplayBuffer.sample()  (fill remaining with buffer)
    |-- TextProcessor.process() (watermarking)
    |-- apply_curriculum()     (ordering + reweighting)
    |
    v
output.jsonl + output_premium.jsonl
```

### Key Classes

| Class | File | One-Line Purpose |
|-------|------|-----------------|
| `Pipeline` | pipeline.py | Orchestrates the entire generation run |
| `Episode` logic | episode.py | Runs one agent-world interaction loop |
| `LLMClient` | llm.py | Async OpenAI-compatible API wrapper |
| `Scorer` | scorer.py | Evaluates trajectory quality (0-100 score) |
| `GenerativePRM` | prm.py | LLM-as-judge step-level verification |
| `TrajectoryRefiner` | refinement.py | Improves low-scoring trajectories |
| `MultiPathGenerator` | multipath.py | Multiple solution paths + beam search |
| `AdaptiveReweighter` | curriculum.py | Weights trajectories for training |
| `PrioritizedBuffer` | replay_buffer.py | SQLite store for high-quality trajectories |
| `FailureAnalyzer` | failure_analysis.py | Turns bad trajectories into training signal |
| `TextProcessor` | watermark.py | Embeds provenance watermarks |

---

## Critical Design Decisions

1. **No frameworks** -- No LangChain, no LlamaIndex. Raw OpenAI-compatible API calls only. Keep it minimal.

2. **Async throughout** -- All LLM calls and episode runs are async. The pipeline uses `asyncio.gather` for parallelism controlled by a semaphore.

3. **Agent pool with round-robin** -- Multiple agent models can be defined in `config.yaml` under `agent.models` with weights. The pipeline round-robins across them.

4. **Score is 0-100** -- Quality gates produce 0.0-1.0 scores, weighted and multiplied to a 0-100 scale. The `premium_threshold` in config is 0.75 (meaning score >= 75). The `replay.insert_threshold` is 55.0.

5. **ChatML + `<think>` tags** -- Output format matches the Harmonic reasoning series. Each trajectory becomes one user turn + one assistant turn with embedded `<think>` block.

6. **SQLite replay buffer** -- Uses WAL mode for concurrent reads. Not a persistent training store -- it's a prioritized cache that evicts stale or low-scoring entries.

---

## Common Tasks for Agents

### "Add a new task scenario"
- Edit `src/tasks.py`, add a new `Task(...)` to the list in `get_task_bank()`
- Follow the pattern: id=`task_0XX`, include `system_context` with realistic initial state, set `expected_steps` range, add `hints` and `tags`
- Categories in use: Data, API/Auth, Debugging, Pipeline, CI/CD, Migration, Monitoring, Database, ETL, Security, Resilience, Deployment, K8s, Performance, DR

### "Add a new quality gate"
- Create a new class inheriting `QualityGate` in `src/scorer.py`
- Implement `evaluate(trajectory) -> (passed: bool, score: float, reason: str)`
- Add instance to `ALL_GATES` list
- Add gate name to `verification.gates` in `config.yaml`

### "Add a new agent tool"
- Add tool definition dict to `TOOLS` list in `src/tools.py`
- The world model will automatically simulate it (it reads tool definitions from the prompt)
- No additional wiring needed -- the parser handles any tool name

### "Change scoring behavior"
- Gate weights are in the `__init__` of each gate class in `scorer.py`
- Boost multipliers are hardcoded in `Scorer.score()` (lines ~199-204)
- PRM weight split is in `prm.py` `combine_prm_with_gates()`: 60% gates, 40% PRM

### "Switch LLM models"
- Edit `config.yaml` under `world_model` and `agent` sections
- Any OpenAI-compatible endpoint works
- For local: set `base_url: http://localhost:11434/v1` (Ollama)
- For cloud: set appropriate `base_url` and `api_key`

---

## Things to Watch Out For

- **API key in config.yaml** -- The OpenRouter key is in plaintext. Don't commit changes that expose it further. Ideally migrate to env vars.
- **`task_generation` is disabled** -- The `generate_task_variants()` and `critic_filter()` functions in `tasks.py` are stubs. Don't enable without implementing them.
- **No test suite** -- There are no unit or integration tests. Be careful with refactors.
- **NumPy serialization** -- `pipeline.py` has a `_NumpyEncoder` and `__init__.py` has `_sanitize()` to handle numpy types in JSON. Don't remove these.
- **World model prompt is critical** -- The `WORLD_MODEL_SYSTEM` prompt in `episode.py` defines how realistic the simulated environment is. Changes here affect all data quality.
- **Watermarking is active** -- `text_processing.enabled: true` in config. The `TextProcessor` modifies thinking text. If debugging weird phrase substitutions in output, check this.

---

## Running the Project

```bash
# Install deps
pip install -r requirements.txt

# Generate a batch of episodes
python run.py generate --n 200 --output corpus.jsonl

# Watch one episode live (great for debugging)
python run.py watch --task-id task_003

# Score an existing corpus
python run.py score --input corpus.jsonl

# Launch the live dashboard
python run.py serve --port 8000

# Analyze corpus with visualizations
python analyze_corpus.py corpus.jsonl
```

---

## Current Models (as of 2026-04-14)

| Role | Model | Endpoint |
|------|-------|----------|
| World Model | glm-5.1:cloud | localhost:11434 (Ollama) |
| Agent (pool) | google/gemma-4-31b-it | OpenRouter |
| Agent (pool) | google/gemma-4-31b-it:free | OpenRouter |

---

## Version History

See `CHANGELOG.md` for detailed session-by-session tracking.

Current: **v0.1 Foundation** (April 2026) -- all core features implemented, no known bugs, zero TODOs in code.
Next: **v0.2 Scale & Quality** (May 2026) -- see `ROADMAP.md`.
