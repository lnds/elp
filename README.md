# ELP — Empirical Lane Parallelism

A method for human-orchestrated parallel multi-agent software engineering.

> *"Welcome back, my friends, to the show that never ends."*
> — Emerson, Lake & Palmer, *Karn Evil 9: First Impression Part 2*, 1973.

## What is ELP?

**ELP** (*Empirical Lane Parallelism*) is a software engineering method that combines three ideas:

1. **Parallel lanes** — each unit of work runs as an isolated session (a `git worktree`, its own branch, its own `tmux` session). Multiple lanes run in parallel.
2. **Brief as contract** — each lane receives a structured document defining its scope, decisions already made, what to do and what not to do, and what to deliver.
3. **Empirical capture of the process** — pre-code audits, mandatory per-lane retrospectives, and per-build measurements are kept as a dataset that feeds the next lanes.

ELP does not replace classical development methods, it complements them with an explicit pattern for coordinating multiple agents without the system degrading within weeks.

## Documents

| Language | Markdown | PDF |
|---|---|---|
| English | [elp-method-en.md](elp-method-en.md) | [elp-method-en.pdf](elp-method-en.pdf) |
| Español | [elp-method.md](elp-method.md) | [elp-method.pdf](elp-method.pdf) |

## Status

Version 0.1 — first public articulation of the method.

The method was formalized from accumulated practice in long-duration projects with multiple AI agents working in parallel. Section 13 of the document presents an empirical case study: a self-hosted functional language with algebraic effects, built in 14 days with 868 commits and 144 merged PRs.

## Author

Eduardo Díaz Cortés — [lnds.net](https://lnds.net)

## License

The text of this method is released under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
You are free to share and adapt it, with attribution.
