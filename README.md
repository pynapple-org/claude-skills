# claude-skills

[Claude Code](https://claude.com/claude-code) skills for [pynapple](https://github.com/pynapple-org/pynapple), maintained by the pynapple organization.

A skill gives Claude working knowledge of a library — its data structures, idioms, and the mistakes it should avoid — so that generated code looks like code a maintainer would write.

## Skills

| Skill | Description |
| --- | --- |
| `using-pynapple` | Core data structures (`Ts`, `Tsd`, `TsdFrame`, `TsdTensor`, `TsGroup`, `IntervalSet`), time series manipulation, metadata filtering, tuning curves, Bayesian and template decoding, signal processing, correlograms, and perievent analysis. |

## Installation

Add this repository as a marketplace, then install the skill:

```
/plugin marketplace add pynapple-org/claude-skills
/plugin install using-pynapple@pynapple-skills
/reload-plugins
```

Claude invokes the skill automatically when you write pynapple code — no need to mention it.

## Attribution

This repository is derived from [catalystneuro/claude-skills](https://github.com/catalystneuro/claude-skills),
copied from commit [`901b3df`](https://github.com/catalystneuro/claude-skills/commit/901b3df9b60f58bddaa1432df023b009cc95f530) (2026-07-03).
The original work is © 2025 CatalystNeuro, released under the MIT License, which is retained
verbatim in [LICENSE](LICENSE).

The upstream repository contains additional skills for the broader neurophysiology ecosystem —
NeMoS, DANDI, NWB conversion, and Spyglass. This repository carries only the pynapple skill,
which the pynapple maintainers develop independently going forward. Changes here are not
automatically reflected upstream, and vice versa.
