# Ranking & Relevance Scoring Strategy

## Overview

Ranking is critical for accuracy in semantic search. Even if we retrieve all relevant concepts, poor ranking means users won't see the best results first. This document details the multi-signal ranking strategy that optimizes for accuracy.

---

## Ranking Philosophy

**Core Principle:** Combine multiple relevance signals to capture different aspects of "what matters" to the user.

**Signals We Use:**
1. **Semantic relevance** - How closely does the concept match the query meaning?
2. **Lexical relevance** - Does it contain the exact keywords?
3. **Formula relevance** - For functional queries, does the formula match the pattern?
4. **Importance** - Is this a key metric or just a data cell?
5. **Recency** - Has this been updated recently?
6. **Context** - Does it fit the query context (domain, sheet, type)?

---

## Hybrid Scoring Function

### Formula

```
final_score = w1 × semantic_score +
              w2 × keyword_score +
              w3 × formula_score +
              w4 × importance_score +
              w5 × recency_score +
              context_bonus +
              field_bonus
```

### Default Weights (Learned/Tuned)

```python
w1 = 0.30  # Semantic similarity (embeddings)
w2 = 0.25  # Keyword matching (BM25)
w3 = 0.15  # Formula pattern matching
w4 = 0.20  # Importance signals
w5 = 0.10  # Recency/temporal signals
```

**Rationale:**
- Semantic (30%): Primary signal for meaning
- Keyword (25%): Strong for exact matches
- Formula (15%): Important for functional queries, less so for conceptual
- Importance (20%): Ensures key metrics surface
- Recency (10%): Tie-breaker, not primary

---

## Component 1: Semantic Score

**Input:** Query embedding vs Concept embedding

**Method:** Cosine similarity

```python
semantic_score = cosine_similarity(query_embedding, concept_embedding)

# Range: [-1, 1], typically [0, 1] for related concepts
# Normalize: clip to [0, 1]
```

**Refinements:**

1. **Multi-vector matching:** Compare query to multiple concept embeddings
   ```python
   semantic_score = max(
       cosine_similarity(query_emb, concept_emb),
       cosine_similarity(query_emb, formula_emb),
       cosine_similarity(query_emb, context_emb)
   )
   ```

2. **Query expansion boost:** If query term is in synonym list
   ```python
   if concept.canonical_term in query.expanded_terms:
       semantic_score *= 1.1  # 10% boost
   ```

3. **Domain alignment:** Same business domain
   ```python
   if query.domain == concept.domain:
       semantic_score *= 1.05  # 5% boost
   ```

**Confidence Thresholds:**
- >0.85: Very strong semantic match
- 0.70-0.85: Strong match
- 0.50-0.70: Moderate match
- <0.50: Weak match (may filter out)

---

## Component 2: Keyword Score

**Input:** Query keywords vs Concept text fields

**Method:** BM25 (Best Match 25) scoring

```python
# BM25 formula
score(Q, C) = Σ IDF(qi) × (f(qi, C) × (k1 + 1)) / (f(qi, C) + k1 × (1 - b + b × |C| / avgdl))

where:
- Q = query keywords
- C = concept document
- f(qi, C) = frequency of query term qi in concept
- |C| = length of concept text
- avgdl = average document length
- k1 = 1.5 (saturation parameter)
- b = 0.75 (length normalization)
```

**Field-Specific Boosting:**

Different fields have different importance:

```python
keyword_score = (
    3.0 × BM25(query, concept.name) +
    2.0 × BM25(query, concept.header) +
    1.5 × BM25(query, concept.section) +
    1.0 × BM25(query, concept.formula_text) +
    0.5 × BM25(query, concept.context)
) / 8.0  # Normalize by sum of weights
```

**Exact Match Bonus:**

```python
if query.phrase in concept.name:
    keyword_score += 0.3  # Significant boost for exact phrase match

if query.phrase in concept.header:
    keyword_score += 0.2
```

**Normalization:** Scale keyword_score to [0, 1]

---

## Component 3: Formula Score

**Input:** Query intent vs Formula pattern

**Applicable When:** Query is functional (e.g., "show percentage calculations")

**Method:** Pattern matching + semantic formula understanding

```python
def formula_score(query, concept):
    if not query.is_functional:
        return 0.0  # Not applicable

    score = 0.0

    # Pattern match
    if query.target_pattern in concept.formula_patterns:
        score += 0.6  # Strong match

    # Partial pattern match
    if query.target_pattern.category == concept.formula_pattern.category:
        score += 0.3  # Category match (e.g., both are aggregations)

    # Complexity alignment
    complexity_diff = abs(query.expected_complexity - concept.formula_complexity)
    complexity_score = max(0, 1 - complexity_diff / 10)
    score += 0.2 × complexity_score

    # Function usage
    if query.target_functions ∩ concept.functions_used:
        score += 0.2  # Uses expected functions

    return min(score, 1.0)
```

**Examples:**

Query: "show percentage calculations"
- Concept with formula `=A/B` (percentage pattern) → score = 0.8
- Concept with formula `=SUM(A:A)` (aggregation) → score = 0.0

Query: "lookup functions"
- Concept with `VLOOKUP` → score = 1.0
- Concept with `INDEX/MATCH` → score = 1.0
- Concept with `SUM` → score = 0.0

---

## Component 4: Importance Score

**Input:** Multiple importance signals

**Method:** Weighted combination of binary and continuous signals

```python
def importance_score(concept):
    score = 0.0

    # Binary signals
    if concept.is_kpi:
        score += 0.30
    if concept.is_named_range:
        score += 0.20
    if concept.in_summary_section:
        score += 0.20
    if concept.has_conditional_formatting:
        score += 0.10
    if concept.has_comment:
        score += 0.05

    # Continuous signals
    reference_score = min(0.35, concept.reference_count / 20)
    score += reference_score

    # Sheet importance
    if "dashboard" in concept.sheet_name.lower():
        score += 0.15
    if "kpi" in concept.sheet_name.lower():
        score += 0.15
    if concept.is_first_sheet:
        score += 0.10

    # Position importance (top-left is more important)
    position_score = (1.0 - concept.row / 100) * 0.05  # Top rows favored
    position_score += (1.0 - concept.col / 26) * 0.05  # Left columns favored
    score += min(position_score, 0.10)

    # Normalize to [0, 1]
    return min(score, 1.0)
```

**Signal Hierarchy:**
1. KPI/Named Range (explicit importance)
2. Reference count (implicit importance)
3. Sheet naming (structural importance)
4. Formatting (visual importance)
5. Position (heuristic importance)

---

## Component 5: Recency Score

**Input:** Last modified timestamp

**Method:** Exponential decay

```python
def recency_score(concept, current_time):
    days_since_modified = (current_time - concept.last_modified).days

    # Exponential decay with 30-day half-life
    decay_rate = -0.023  # ln(0.5) / 30
    base_score = exp(decay_rate × days_since_modified)

    # Boost for recently accessed (if tracking enabled)
    if concept.last_accessed and (current_time - concept.last_accessed).days < 7:
        base_score += 0.2

    # Boost for very recent edits
    if days_since_modified < 1:
        base_score += 0.1

    return min(base_score, 1.0)
```

**Decay Curve:**
- 0 days: score = 1.0
- 7 days: score ≈ 0.85
- 30 days: score = 0.5
- 90 days: score ≈ 0.12
- 365 days: score ≈ 0.0006

**Note:** Recency is a weak signal (10% weight) to avoid over-prioritizing unimportant recent cells.

---

## Context Bonuses

Additional boosts based on query-concept context alignment.

### Multi-Sheet Query Bonus

```python
def multisheet_bonus(query, concept):
    if not query.is_comparative:
        return 0.0

    # If concept is part of a cross-sheet relationship matching query
    if concept.has_cross_sheet_match(query.dimensions):
        return 0.3

    return 0.0
```

Example: Query "budget vs actual" → concepts that exist in both Budget and Actual sheets get +0.3

### Domain Match Bonus

```python
def domain_bonus(query, concept):
    if query.domain == concept.domain:
        return 0.1  # Same business domain
    return 0.0
```

### Time Period Bonus

```python
def time_period_bonus(query, concept):
    if query.time_period and concept.time_period:
        if query.time_period == concept.time_period:
            return 0.15  # Exact period match
        if are_adjacent_periods(query.time_period, concept.time_period):
            return 0.05  # Adjacent period
    return 0.0
```

Example: Query "Q4 revenue" → concepts labeled Q4 get +0.15

---

## Field Boost (Multiplicative)

Based on which field matched:

```python
def get_field_multiplier(matched_field):
    multipliers = {
        "concept_name": 1.3,      # Best match location
        "canonical_term": 1.25,   # Dictionary match
        "header": 1.2,            # Column/row header
        "named_range": 1.15,      # Named range
        "section": 1.1,           # Section title
        "formula": 1.05,          # In formula
        "context": 1.0            # General context
    }
    return multipliers.get(matched_field, 1.0)
```

---

## Query Type Adjustments

Different query types prioritize different signals.

### Conceptual Queries
Example: "find profitability metrics"

```python
# Emphasize semantic and domain understanding
w1 = 0.40  # Semantic ↑
w2 = 0.20  # Keyword
w3 = 0.05  # Formula ↓
w4 = 0.25  # Importance
w5 = 0.10  # Recency
```

### Functional Queries
Example: "show percentage calculations"

```python
# Emphasize formula patterns
w1 = 0.20  # Semantic ↓
w2 = 0.15  # Keyword
w3 = 0.45  # Formula ↑↑
w4 = 0.15  # Importance ↓
w5 = 0.05  # Recency
```

### Navigational Queries
Example: "where is revenue"

```python
# Emphasize exact matching and importance
w1 = 0.25  # Semantic
w2 = 0.40  # Keyword ↑
w3 = 0.05  # Formula
w4 = 0.25  # Importance
w5 = 0.05  # Recency
```

### Comparative Queries
Example: "budget vs actual"

```python
# Emphasize relationships and context
w1 = 0.30  # Semantic
w2 = 0.25  # Keyword
w3 = 0.10  # Formula
w4 = 0.20  # Importance
w5 = 0.15  # Recency ↑
# + multisheet_bonus
```

---

## Diversity and De-duplication

### Avoiding Redundancy

If multiple results are very similar, promote diversity:

```python
def diversify_results(ranked_results, top_k=10):
    diverse_results = []
    seen_concepts = set()

    for result in ranked_results:
        # Check similarity to already selected results
        is_duplicate = False
        for seen in diverse_results:
            similarity = concept_similarity(result, seen)
            if similarity > 0.95:  # Very similar
                is_duplicate = True
                break

        if not is_duplicate:
            diverse_results.append(result)

        if len(diverse_results) >= top_k:
            break

    return diverse_results
```

### Grouping Related Results

For multi-sheet queries, group related concepts:

```python
def group_results(results, query):
    if query.is_comparative:
        # Group by concept across sheets
        groups = {}
        for result in results:
            key = result.canonical_name
            if key not in groups:
                groups[key] = []
            groups[key].append(result)
        return groups
    else:
        return results  # No grouping
```

---

## Learning to Rank (Optional Enhancement)

### Using User Feedback

```python
# Collect features for each result
features = [
    semantic_score,
    keyword_score,
    formula_score,
    importance_score,
    recency_score,
    position_in_retrieval,
    concept_type,
    sheet_position,
    ...
]

# Label: user clicked or not
label = 1 if user_clicked else 0

# Train gradient boosted trees (LightGBM, XGBoost)
model = LGBMRanker()
model.fit(features, labels, group=query_ids)

# At query time
learned_score = model.predict(features)
```

**Advantages:**
- Learns optimal weight combination from data
- Captures non-linear interactions
- Adapts to user preferences

**Disadvantages:**
- Requires labeled data (clicks, ratings)
- Less explainable
- More complex to maintain

---

## Confidence Scoring

Each result gets a confidence level based on scores:

```python
def confidence_level(final_score, semantic_score, keyword_score):
    # High confidence: strong on multiple signals
    if final_score > 0.75 and semantic_score > 0.7:
        return "high"

    # Medium confidence: decent overall score
    elif final_score > 0.50:
        return "medium"

    # Low confidence: weak match
    else:
        return "low"
```

**Display:**
- High: Show prominently
- Medium: Show with "Possibly relevant" qualifier
- Low: Show in "Other potential matches" section or filter out

---

## Result Filtering

### Minimum Score Threshold

```python
MIN_SCORE_THRESHOLD = 0.40

filtered_results = [r for r in results if r.score >= MIN_SCORE_THRESHOLD]
```

### Maximum Results

```python
MAX_RESULTS = 50  # Don't overwhelm user

top_results = filtered_results[:MAX_RESULTS]
```

### Dynamic Thresholding

```python
def dynamic_threshold(results):
    if not results:
        return []

    # If top result has high score, use high threshold
    top_score = results[0].score

    if top_score > 0.85:
        threshold = 0.60  # Only show high-quality results
    elif top_score > 0.70:
        threshold = 0.50
    else:
        threshold = 0.40  # More lenient if no great matches

    return [r for r in results if r.score >= threshold]
```

---

## Ranking Example Walkthrough

### Query: "find profitability metrics"

**Candidate:** Gross Profit Margin (Q4 Financials!D15)

#### Step 1: Component Scores

```python
# Semantic Score
query_emb = [0.23, 0.45, ..., 0.78]  # "profitability metrics" embedding
concept_emb = [0.25, 0.43, ..., 0.76]  # "Gross Profit Margin" embedding
semantic_score = cosine_similarity(query_emb, concept_emb) = 0.94

# Keyword Score
# BM25 matching "profitability" and "metrics" against concept text
keyword_score = BM25("profitability metrics", concept.text) = 0.72
# Exact match bonus: "margin" in query expansion
keyword_score += 0.1 = 0.82

# Formula Score (not functional query)
formula_score = 0.0

# Importance Score
is_kpi = True → +0.30
is_named_range = True → +0.20
in_summary_section = True → +0.20
has_conditional_formatting = True → +0.10
reference_count = 8 → +0.35 (capped)
sheet_name = "Q4 Financials" → +0.0 (not dashboard)
importance_score = 1.15 (capped at 1.0) = 1.0

# Recency Score
days_since_modified = 3
recency_score = exp(-0.023 × 3) + 0.1 (recent) = 1.03 (capped at 1.0) = 1.0
```

#### Step 2: Weighted Combination

```python
final_score = 0.30 × 0.94 +   # Semantic
              0.25 × 0.82 +   # Keyword
              0.15 × 0.0 +    # Formula
              0.20 × 1.0 +    # Importance
              0.10 × 1.0      # Recency

final_score = 0.282 + 0.205 + 0.0 + 0.20 + 0.10 = 0.787
```

#### Step 3: Context Bonuses

```python
# Domain match (finance)
final_score += 0.1 = 0.887

# Field boost (concept_name match)
final_score × 1.2 = 1.064 (capped at 1.0) = 1.0
```

#### Step 4: Confidence

```python
confidence = "high"  # final_score > 0.75, semantic_score > 0.7
```

#### Result

```json
{
  "concept": "Gross Profit Margin",
  "location": "Q4 Financials!D15",
  "score": 0.89,
  "confidence": "high",
  "components": {
    "semantic": 0.94,
    "keyword": 0.82,
    "importance": 1.0,
    "recency": 1.0
  },
  "explanation": "Strong match for profitability metric. Found in Q4 Financials, key performance indicator.",
  "rank": 1
}
```

---

## Ranking Evaluation Metrics

### Precision@K

```python
# Of top K results, how many are relevant?
precision_at_k = relevant_in_top_k / k

# Target: P@5 > 0.80 (4 out of 5 relevant)
```

### Recall@K

```python
# Of all relevant results, how many in top K?
recall_at_k = relevant_in_top_k / total_relevant

# Target: R@10 > 0.70 (find 70% of relevant in top 10)
```

### Mean Reciprocal Rank (MRR)

```python
# Average of reciprocal ranks of first relevant result
mrr = mean([1/rank_of_first_relevant for query in queries])

# Target: MRR > 0.75 (first relevant typically in top 2-3)
```

### Normalized Discounted Cumulative Gain (nDCG)

```python
# Accounts for graded relevance and position
dcg_at_k = Σ (2^relevance[i] - 1) / log2(i + 1) for i in 1..k

idcg_at_k = DCG of perfect ranking

ndcg_at_k = dcg_at_k / idcg_at_k

# Target: nDCG@10 > 0.80
```

---

## A/B Testing Framework

To optimize weights and strategies:

### Experiment Setup

```python
# Control: Current weights
control_weights = [0.30, 0.25, 0.15, 0.20, 0.10]

# Variant A: Emphasize semantic more
variant_a_weights = [0.40, 0.20, 0.10, 0.20, 0.10]

# Variant B: Emphasize keyword more
variant_b_weights = [0.25, 0.35, 0.10, 0.20, 0.10]
```

### Metrics to Track

1. Click-through rate (CTR) on top result
2. Time to first click
3. Query refinement rate
4. User satisfaction rating (if collected)
5. Task completion rate

### Statistical Significance

```python
# Run for N queries, test statistical significance
# Use t-test or Mann-Whitney U test
# Adopt winning variant if p < 0.05 and improvement > 5%
```

---

## Summary

The ranking strategy combines:

1. **Multiple signals** (semantic, keyword, formula, importance, recency)
2. **Weighted scoring** (tunable weights per query type)
3. **Context bonuses** (multi-sheet, domain, time)
4. **Diversity** (avoid redundancy)
5. **Confidence levels** (high/medium/low)
6. **Learning** (optional ML ranking)

This multi-faceted approach ensures that the most relevant results surface first, directly impacting the **accuracy** of the semantic search system—the primary optimization goal.

---

## Next Steps

1. Implement baseline ranking with default weights
2. Collect query logs and user feedback
3. Measure precision@K, recall@K, MRR, nDCG
4. A/B test weight variations
5. Implement learning-to-rank if sufficient data
6. Continuously iterate based on metrics
