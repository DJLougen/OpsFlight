# OpsFlight

**The Flight Simulator for Production AI Agents**

Train reliable software agents that can debug production issues, deploy configs, run data pipelines, perform security audits, and handle real SRE/DevOps work -- **without ever touching real infrastructure**.

Instead of risky, slow, or expensive real environments, OpsFlight uses two LLMs in a loop:
- **Agent LLM** -- thinks step-by-step and calls tools
- **Adversarial World Model LLM** -- maintains a realistic fake production environment and returns believable (sometimes tricky) observations

The result: thousands of clean, high-signal training trajectories with deep reasoning, self-correction, verification, and proper tool sequencing.

---

## Why OpsFlight?

Real production agent training is hard:
- **Dangerous** -- one wrong `rm` or config change can break things
- **Expensive and slow** -- spinning up clusters, databases, APIs
- **Doesn't scale** -- you can't run 1000 parallel incidents

OpsFlight solves this with a safe, scalable "flight simulator" that produces production-grade data overnight for the cost of LLM API calls.

This data is fully compatible with your existing pipelines (Unsloth, Axolotl, llama.cpp, etc.) and uses the same `<think>` format as the Harmonic series.

---

## Key Features

- **13 realistic tools** (read/write files, run commands, call APIs, grep, memory scratchpad, etc.)
- **Adversarial but fair World Model** -- rate limits, partial failures, encoding issues, mixed stdout/stderr, etc.
- **20+ hard ops-grade scenarios** (502 debugging, K8s CrashLoop, secret rotation, circuit breaker, disaster recovery, etc.)
- **Built-in Scorer** -- per-step and final rewards (0.0-1.0)
- **Automatic filtering** -- thinking depth, self-correction (10x boost), verification (3.6x), alternative exploration (14x)
- **Zero degenerate patterns** -- no skipped reasoning, no immediate tool jumps
- **Ready-to-train output** -- SFT format with `<think>` tags + metadata (signal_score, difficulty, domain)

---

## Architecture

```
        +----------------------------------+
        |           TASK BANK              |
        |  "The prod service returns 502.  |
        |   Investigate and fix it."       |
        +----------------+-----------------+
                         | sample task
                         v
   +-------------------------------------------+
   |            EPISODE LOOP                    |
   |                                            |
   |   +---------+  action   +--------------+   |
   |   |  AGENT  |---------->| WORLD MODEL  |   |
   |   |  (LLM)  |           |    (LLM)     |   |
   |   |         |<----------|              |   |
   |   | thinks  | obs +     | Maintains:   |   |
   |   | + calls | state     | - filesystem |   |
   |   | 1 tool  | + reward  | - APIs       |   |
   |   |         |           | - processes  |   |
   |   +---------+           | - network    |   |
   |        |                +--------------+   |
   |        | finish()                          |
   |        v                                   |
   |   +----------+                             |
   |   |  SCORER  | -> 0.0 to 1.0              |
   |   +----------+                             |
   +-------------------+-----------------------+
                       |
                       v
              +-----------------+
              |  TRAJECTORY     |
              |  (JSONL)        |
              |                 |
              | <think> tags    |
              | + metadata      |
              | + signal_score  |
              +-----------------+
```

## Quick Start

```bash
git clone https://github.com/DJLougen/OpsFlight.git
cd OpsFlight
pip install -r requirements.txt
```

### Generate episodes

```bash
# Run a full batch
python run.py generate --n 200 --output corpus.jsonl --max-steps 30

# Watch a single episode live
python run.py watch --task-id task_003

# Score an existing corpus
python run.py score --input corpus.jsonl

# Launch the live dashboard
python run.py serve --port 8000
```

### Analyze the corpus

```bash
python analyze_corpus.py corpus.jsonl
```

This produces:
- `analysis_output/overview.png` -- score, steps, signal distributions
- `analysis_output/behavior_patterns.png` -- tool transition heatmap
- `analysis_output/reward_curves.png` -- reward trajectories by category
- `analysis_output/signal_analysis.png` -- signal score vs steps/tags
- `analysis_output/training_premium.jsonl` -- quality-gated SFT data

### Configuration

```yaml
# config.yaml
world_model:
  base_url: http://localhost:11434/v1
  model: kimi-k2.5:cloud
  temperature: 0.7

agent:
  base_url: http://localhost:11434/v1
  model: kimi-k2.5:cloud
  temperature: 0.3

episode:
  max_steps: 30
  parallel: 4
  timeout_seconds: 120
```

Any OpenAI-compatible endpoint works (Ollama, vLLM, OpenRouter, etc.).

---

## The 13 Agent Tools

| Tool | Description |
|------|-------------|
| `read_file(path)` | Read file contents (may fail: ENOENT, EACCES, encoding) |
| `write_file(path, content)` | Write/overwrite file (may fail: ENOSPC, permissions) |
| `append_file(path, content)` | Append to existing file |
| `list_dir(path)` | List directory contents with sizes and permissions |
| `delete_file(path)` | Remove a file |
| `call_api(url, method, headers, body)` | HTTP request (may: timeout, 429, return HTML errors) |
| `run_command(cmd)` | Shell command with stdout + stderr + exit code |
| `grep_files(pattern, path)` | Regex search across files |
| `memory_set(key, value)` | Store in agent working memory |
| `memory_get(key)` | Retrieve from working memory |
| `log(level, message)` | Emit structured log (debug/info/warn/error) |
| `sleep(seconds)` | Wait (for rate limits / backoff) |
| `finish(result)` | Submit final answer and end the episode |

---

## Task Bank (20 Production Scenarios)

| # | Scenario | Steps | Category |
|---|----------|-------|----------|
| 1 | CSV cleanup with mixed delimiters + encoding | 8-12 | Data |
| 2 | OAuth2 config deploy with rate limits | 10-15 | API/Auth |
| 3 | 502 Bad Gateway debugging (nginx port mismatch) | 8-10 | Debugging |
| 4 | Data pipeline: paginated API + aggregation | 15-20 | Pipeline |
| 5 | CI/CD failure: security bug in auth code | 10-14 | CI/CD |
| 6 | API migration: XML to JSON with validation | 10-15 | Migration |
| 7 | Multi-service health monitoring + restart | 12-18 | Monitoring |
| 8 | Database migration with backup + verification | 10-14 | Database |
| 9 | ETL with malformed data + dedup + reporting | 10-15 | ETL |
| 10 | Secret rotation via Vault + config update | 12-16 | Security |
| 11 | Memory leak investigation (Node.js) | 8-12 | Debugging |
| 12 | Retry-with-backoff against flaky API | 10-15 | Resilience |
| 13 | Cross-source security audit (CSV + logs + IAM) | 10-14 | Security |
| 14 | Blue-green deployment validation | 10-14 | Deployment |
| 15 | Log aggregation + threshold alerting | 10-14 | Monitoring |
| 16 | K8s CrashLoopBackOff (OOMKilled) | 10-14 | K8s |
| 17 | Circuit breaker state machine | 10-15 | Resilience |
| 18 | Full security scan (secrets + TLS + deps) | 12-18 | Security |
| 19 | Database slow query investigation + indexing | 10-14 | Performance |
| 20 | Disaster recovery drill (DB failover) | 10-14 | DR |

---

## Example Trajectory

```
Task: "A production service is returning 502 errors. Investigate and fix."

Step 0: read_file("/var/log/nginx/error.log")
  → "connect() failed (111: Connection refused) upstream: http://127.0.0.1:8081/"
  reward: 0.15

Step 1: call_api("https://api.internal/health", "GET")
  → "HTTP 502 Bad Gateway <html>...</html>"
  reward: 0.25

Step 2: run_command("df -h")
  → Filesystem output (all healthy — rules out disk)
  reward: 0.35

Step 3: run_command("ps aux | grep app")
  → "app-server --port=8080" (running on 8080, not 8081!)
  reward: 0.50

Step 4: read_file("/etc/nginx/conf.d/upstream.conf")
  → "server 127.0.0.1:8081" — wrong port confirmed
  reward: 0.60

Step 5: write_file("/etc/nginx/conf.d/upstream.conf", fixed config)
  → Updated 8081 → 8080
  reward: 0.70

Step 6: run_command("nginx -s reload")
  → Reload successful
  reward: 0.80

Step 7: call_api("https://api.internal/health", "GET")
  → HTTP 200 OK {"status": "healthy"}
  reward: 0.90

Step 8: finish("Root cause: nginx upstream port 8081 vs app on 8080. Fixed config, reloaded, verified.")
  reward: 1.00

Final score: 1.0 | 9 steps | Tags: debugging, nginx, ops
```

---

## Training Data Format

Output is ChatML-compatible with `<think>` tags, identical to the Harmonic reasoning format:

```json
{
  "id": "ccd2db66-7365",
  "conversations": [
    {"role": "user", "content": "Debug the 502 errors on the production service..."},
    {"role": "assistant", "content": "<think>\nStep 0: I'll call read_file(...)\nResult: Connection refused...\n...\n</think>\n\nRoot cause: nginx upstream port mismatch..."}
  ],
  "metadata": {
    "task_id": "task_003",
    "domain": "debugging_nginx",
    "difficulty": "hard",
    "signal_score": 72.5,
    "steps": 9,
    "tool_diversity": 0.46,
    "has_recovery": true,
    "chain_length": 9
  }
}
```

---

## Live Dashboard

```bash
python run.py serve --port 8000
```

Real-time web UI showing:
- Episode generation progress with WebSocket updates
- Score/steps/signal distributions
- Tool usage heatmaps and transition matrices
- Quality gate pass rates
- Click any episode to replay its full trajectory
- Run individual episodes or batches from the browser

---

## Datasets (Hugging Face)

- [dj-lougen/opsflight-traces-v1](https://huggingface.co/datasets/dj-lougen/opsflight-traces-v1) -- Clean filtered trajectories

## Models (Hugging Face)

- [DJLougen/Harmonic-OpsFlight-9B](https://huggingface.co/DJLougen/Harmonic-OpsFlight-9B) -- Fine-tuned on OpsFlight data

---

## Related Work

- [Harmonic series](https://huggingface.co/DJLougen) -- Reasoning-focused base models
- [hermes-agent-traces-filtered](https://huggingface.co/datasets/DJLougen/hermes-agent-traces-filtered) -- Previous agent data filtering work

---

## Contributing

Contributions welcome! Ideas for new tasks, improvements to the World Model prompt, or better scoring logic are especially appreciated.

---

## License

MIT License -- free to use for research and commercial fine-tuning.

---

## Contact

- Creator: **Daniel Lougen, M.S.** ([@DJLougen](https://x.com/DJLougen) on X)
- Hugging Face: [DJLougen](https://huggingface.co/DJLougen)

Built as a side project while pursuing a PhD in visual neuroscience at the University of Toronto. Applying rigorous data filtering and cognitive science principles to agent training.

**Star the repo if you're building better agents!**
