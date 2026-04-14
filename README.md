# System Design Interview Practice

AI-assisted system design interview preparation across multiple companies and roles.

## Purpose

Realistic mock interviews with a strict interviewer — no validation, no spoon-feeding. Sessions mirror actual interview conditions.

## Structure

```
interviews/
├── apple/
│   └── ICT3/           # Senior Engineer equivalent
├── google/
│   └── L5/
└── meta/
    └── E5/
templates/
└── session_template.md
```

## Session Format

Each session covers:
1. **High-Level Design (HLD)** — major components, data flow, API contracts
2. **Low-Level Design (LLD)** — data models, algorithms, concurrency, edge cases

## Interviewer Behavior

- Asks clarifying probes when answers are vague
- Calls out incorrect or incomplete reasoning
- Does not offer hints unless explicitly asked (and penalizes hint requests)
- Pushes on tradeoffs: consistency vs availability, latency vs throughput, cost vs reliability

## Companies & Target Roles

| Company | Role | Level |
|---------|------|-------|
| Apple   | Software Engineer | ICT3 |

## How to Use

Open a session, tell Claude to behave as a system design interviewer per `.claude/CLAUDE.md`, and start talking.
