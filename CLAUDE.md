# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is a **design documentation repository** for a semantic search system for spreadsheets, created for the Superjoin hiring assignment. It contains no executable code—only comprehensive solution design documents in Markdown format.

**Primary Goal:** Optimize for **accuracy** in semantic search results.

## Document Architecture

The solution is organized as interconnected design documents:

### Core Flow
```
SOLUTION.md (entry point)
    ↓
├─→ approach1-hybrid.md [RECOMMENDED]
│   └─→ Embeddings + Symbolic rules approach
│
├─→ approach2-rules.md [ALTERNATIVE]
│   └─→ Rules-based + heuristics approach
│
├─→ data-structures.md
│   └─→ 6 primary structures: Cell Concept Index, Inverted Index,
│       Formula Pattern Index, Business Ontology, Relationship Graph, Query Cache
│
├─→ architecture.md
│   └─→ ASCII workflow diagrams for indexing, query processing,
│       multi-sheet understanding, and incremental updates
│
├─→ ranking.md
│   └─→ Multi-signal scoring: semantic + keyword + formula + importance + recency
│       Default weights: [0.30, 0.25, 0.15, 0.20, 0.10]
│
├─→ example-queries.md
│   └─→ 7 categories: Conceptual, Functional, Comparative, Navigational,
│       Complex/Ambiguous, Synonym handling, Multi-constraint
│       50+ test queries with expected results
│
└─→ edge-cases.md
    └─→ 9 categories covering 30+ edge cases with mitigation strategies
```

### Key Design Decisions

**Approach 1 (Hybrid - Recommended):**
- Combines vector embeddings (semantic understanding) with symbolic rules (precision)
- Multi-index architecture: Vector DB + Elasticsearch + Neo4j + PostgreSQL
- Achieves 85-90% accuracy on known patterns, 75-85% on novel queries
- Technology stack: Qdrant/Pinecone, Elasticsearch, Neo4j, PostgreSQL, OpenAI/sentence-transformers

**Approach 2 (Rules-based - Alternative):**
- Pattern matching + business dictionary + formula cataloging
- Simpler stack: Elasticsearch or in-memory index + PostgreSQL
- Achieves 80-85% accuracy on known patterns, 40-60% on novel queries
- Use when: budget-constrained, privacy-first, or standard business formats

**Ranking Strategy (Critical for Accuracy):**
- Hybrid scoring function with 5 weighted components
- Query-type adaptive weighting (conceptual vs functional vs navigational)
- Confidence levels: high (>0.75), medium (0.5-0.75), low (<0.5)
- Evaluation metrics: Precision@5 >80%, Recall@10 >70%, MRR >0.75

## Working with Documents

### Reading Order for Understanding
1. Start: `SOLUTION.md` (overview, approach comparison)
2. Deep dive: `approach1-hybrid.md` (recommended implementation)
3. Data layer: `data-structures.md` (6 primary indexes)
4. Visual flow: `architecture.md` (ASCII diagrams)
5. Validation: `example-queries.md` + `edge-cases.md`

### Modifying Documents

**When updating any approach:**
- Maintain consistency across `SOLUTION.md`, `approach1-hybrid.md`, `approach2-rules.md`
- Update trade-off comparison table in `SOLUTION.md`
- Verify accuracy claims remain realistic

**When adding/modifying data structures:**
- Update `data-structures.md` with rationale for each structure
- Reflect storage technology choices in approach documents
- Update memory/performance estimates

**When changing ranking logic:**
- Update weight values in `ranking.md`
- Propagate to `approach1-hybrid.md` workflow examples
- Update accuracy predictions if significantly changed

**When adding test cases:**
- Categorize properly in `example-queries.md` (7 categories)
- Include: query, intent, expected results, accuracy test criteria
- Add corresponding edge cases to `edge-cases.md` if complex

**When documenting edge cases:**
- Format: Scenario → Impact → Mitigation → Expected Behavior
- Categorize by impact: High (must handle), Medium (should), Low (nice-to-have)
- Include real spreadsheet examples

### Cross-References to Maintain
- `README.md` summarizes all other documents—update when major changes occur
- `SOLUTION.md` references all detailed documents—keep links working
- Architecture diagrams in `architecture.md` should match data structures in `data-structures.md`
- Example queries should be testable against the ranking strategy in `ranking.md`

## Key Concepts & Terminology

**Concept:** A semantically meaningful cell or cell range (not just raw cells). Has name, context, formula semantics, embeddings, relationships, and importance signals.

**Multi-Signal Ranking:** Combining semantic similarity, keyword matching, formula patterns, importance indicators, and recency into a weighted score. Query-type adaptive.

**Formula Semantics:** Understanding calculation patterns (margin calculation, growth rate, lookup, etc.) beyond raw formula text. Mapped to business concepts.

**Cross-Sheet Relationship:** Linking related concepts across multiple sheets (Budget/Actual/Variance, Q3/Q4, etc.) through explicit formulas, semantic similarity, or structural matching.

**Confidence Scoring:** Each result gets high/medium/low confidence based on multiple signal strength. Filters or flags results below thresholds.

## Accuracy Targets

All design decisions should preserve these targets:
- **Precision@5:** >80% (4 of 5 top results relevant)
- **Recall@10:** >70% (capture 70% of relevant results in top 10)
- **MRR:** >0.75 (first relevant result typically in top 2-3)
- **Response Time:** <2 seconds for 90% of queries

## Common Modifications

### Adding a New Query Type
1. Add to `example-queries.md` with category and expected results
2. Update query type classification in `approach1-hybrid.md` → Query Understanding section
3. Adjust ranking weights in `ranking.md` → Query Type Adjustments
4. Add edge cases if needed in `edge-cases.md`

### Proposing Technology Alternative
1. Update technology stack section in relevant approach document
2. Update trade-off comparison in `SOLUTION.md`
3. Update storage estimates in `data-structures.md`
4. Revise implementation phases in `SOLUTION.md` if timeline changes

### Adding Formula Pattern
1. Document in `data-structures.md` → Formula Pattern Index
2. Add detection logic to `approach1-hybrid.md` → Semantic Enrichment
3. Add test queries to `example-queries.md` → Functional Queries
4. Update accuracy factors if impactful

### Documenting New Edge Case
1. Add to appropriate category in `edge-cases.md`
2. Include: Scenario, Impact, Mitigation, Expected Behavior
3. Rate impact: High/Medium/Low
4. Add test query to `example-queries.md` if needed

## Solution Philosophy

This design prioritizes:
1. **Accuracy over speed** (but maintains <2s target)
2. **Explainability** (users understand why results matched)
3. **Robustness** (handles messy real-world spreadsheets)
4. **Practical implementation** (uses proven technologies)

Trade-offs accepted:
- Storage overhead (3-5x due to multiple indexes) for query speed
- Setup complexity (embeddings, multiple DBs) for accuracy
- Eventual consistency (2-3s update lag) for performance

## Reference Documents

**Problem Statement:** `Superjoin Hiring Problem Statement Round — Build Semantic Search for Spreadsheets.pdf`

**Main Entry Point:** `README.md` or `SOLUTION.md`

**For Interview Prep:** Follow reading order in README.md → "For the Interview Discussion"
