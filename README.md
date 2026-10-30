# Engineering Handbook

A personal reference for backend and AI engineering: concepts explained in my own words, with examples and trade-offs.

## How it's organised

| Folder | What's inside |
|---|---|
| `backend/` | APIs, databases, auth, caching, reliability |
| `ai-engineering/` | LLM APIs, embeddings, RAG, evaluation |

## Index

### Backend
| # | Topic | Key ideas | Note |
|---|-------|-----------|------|
| 01 | HTTP & API fundamentals | request journey, idempotency, status codes, cursor pagination, capacity & percentiles | [Read](backend/01-http-and-api-fundamentals.md) |
| 02 | Relational databases & SQL | keys & constraints, joins, GROUP BY, window functions, normalization vs denormalization | [Read](backend/02-relational-databases-and-sql.md) |
| 03 | Indexes & query plans | B-tree, scan types, composite & partial indexes, EXPLAIN, index trade-offs | [Read](backend/03-indexes-and-query-plans.md) |

### AI engineering
| Topic | Note |
|---|---|

## Format of every note
1. What it is, in two lines
2. Why it matters in a real system
3. A small example
4. Trade-offs and common mistakes
