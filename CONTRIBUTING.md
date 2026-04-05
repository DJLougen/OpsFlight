# Contributing to OpsFlight

Thanks for your interest in contributing to OpsFlight!

## How to Contribute

### New Tasks
The biggest impact you can have is adding new production-grade scenarios to the task bank. Good tasks:
- Require 8-15 steps to solve
- Have realistic initial state (config files, log files, API endpoints)
- Include at least one error the agent must recover from
- Have clear, verifiable success criteria

### World Model Improvements
The adversarial world model is the core of OpsFlight. Improvements that make it more realistic are always welcome:
- More realistic command output formats
- Better error simulation (race conditions, partial writes, network partitions)
- Improved state consistency tracking

### Scoring & Filtering
Better quality metrics for trajectories:
- New signal score components
- Degenerate pattern detection
- Tool sequence quality heuristics

## Getting Started

1. Fork the repo
2. Create a feature branch
3. Make your changes
4. Submit a PR with a clear description

## Code Style

- Python, async throughout
- No frameworks (no LangChain, no LlamaIndex) -- raw API calls only
- Keep it minimal -- single file if possible

## Questions?

Open an issue or reach out on X [@DJLougen](https://x.com/DJLougen).
