# OpsFlight Roadmap

## v0.1 — Foundation (April 2026)
- [x] Core episode runner (agent + world model loop)
- [x] 13 agent tools
- [x] 20 hard ops-grade tasks
- [x] Adversarial world model with realistic output
- [x] Per-step and final scoring
- [x] JSONL trajectory recording
- [x] SFT conversion with `<think>` tags
- [x] Live web dashboard
- [x] Quality analysis pipeline (signal scores, quality gates)
- [ ] First public dataset on Hugging Face (200+ episodes)

## v0.2 — Scale & Quality (May 2026)
- [ ] 50+ task scenarios (add: DNS failures, cert expiry, deployment rollbacks, IAM issues, log rotation, cron debugging)
- [ ] Failure injection modes (force the world model to break things mid-episode)
- [ ] Multi-agent episodes (two agents collaborating on an incident)
- [ ] Weaker agent mode (use small models to generate failure trajectories for DPO)
- [ ] 1,000+ episode corpus
- [ ] Harmonic-OpsFlight-9B model release

## v0.3 — Advanced Scenarios (June 2026)
- [ ] Multi-service incidents (cascading failures across 3+ services)
- [ ] Time-pressure scenarios (SLA countdown, escalation paths)
- [ ] Observability integration (fake Grafana dashboards, PagerDuty alerts)
- [ ] Terraform/IaC debugging scenarios
- [ ] Cloud provider API simulation (AWS, GCP)
- [ ] 5,000+ episode corpus

## v1.0 — Production Release (Q3 2026)
- [ ] Full benchmark suite for agent evaluation
- [ ] Leaderboard for model comparison
- [ ] Plugin system for custom tools and environments
- [ ] 10,000+ episode corpus
- [ ] Multiple model releases (7B, 14B, 32B)
- [ ] Paper submission

## Long-term Vision
- Standard benchmark for production agent capability (like HumanEval but for ops)
- Community-contributed task packs (security, databases, networking, cloud)
- Integration with real observability stacks for hybrid training
