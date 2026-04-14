# System Design Interview & Learning

AI-assisted system design preparation — two modes: structured learning and mock interviews.

## Modes

### `/learn` — Teacher Mode
Structured lessons across a 3-layer curriculum. Tracks progress, uses web research for current content, writes key terms to chat before speaking.

### `/interview` — Interviewer Mode
Strict mock interviews. No validation, no hints (unless asked, penalized). Sessions logged with score and feedback.

## Curriculum: 3-Layer Hierarchy

```
Layer 3: Question Archetypes (12 question types)
         ↓ built from
Layer 2: Patterns (10 design patterns)
         ↓ built from
Layer 1: Foundations (12 base concepts)
```

See [curriculum/00_overview.md](curriculum/00_overview.md) for the full topic map and status.

## Repository Structure

```
CLAUDE.md                        # Interviewer + teacher behavior rules
curriculum/
├── 00_overview.md               # Full 3-layer curriculum map
├── progress.md                  # Current position and gaps
├── layer1-foundations/          # L1-01 through L1-12 lesson notes
├── layer2-patterns/             # L2-01 through L2-10 lesson notes
└── layer3-archetypes/           # L3-01 through L3-12 question breakdowns
interviews/
├── apple/ICT3/                  # Session logs
├── google/L5/
└── meta/E5/
templates/
└── session_template.md
```

## Companies & Target Roles

| Company | Role | Level | Sessions |
|---------|------|-------|---------|
| Apple   | Software Engineer | ICT3 | 1 |

## Voice UX

Key terms and data points are written to chat before every voice explanation — read these while listening to follow along without needing to scroll back.

## Learning Path (Recommended Order)

1. L1-01 Networking → L1-02 Push Delivery (APNs/FCM)
2. L1-08 Message Queues (Kafka/SQS)
3. L1-05 + L1-06 Databases (Relational + NoSQL)
4. L1-07 Caching
5. L1-10 Distributed Systems Fundamentals
6. Layer 2 Patterns
7. Layer 3 Mock Interviews
