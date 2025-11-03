# Approach 1: Hybrid Index (Embeddings + Symbolic) [RECOMMENDED]

## Overview

This approach combines the strengths of:
- **Neural embeddings:** Capture semantic similarity, handle synonyms naturally
- **Symbolic rules:** Provide precision, explainability, and handle edge cases
- **Structured extraction:** Leverage spreadsheet structure and formulas explicitly

The hybrid approach achieves better accuracy by using embeddings for semantic understanding while grounding results with rule-based validation and scoring.

---

## Architecture Components

### 1. Indexing Pipeline (Offline/On-Change)

#### Step 1: Spreadsheet Parsing & Structure Analysis
```
Input: Spreadsheet file (XLSX, Google Sheets, etc.)
Output: Structured representation

Process:
1. Extract all sheets, cells, formulas, formats
2. Detect structure:
   - Header rows/columns (using heuristics: bold, top position, unique values)
   - Table boundaries (contiguous data regions)
   - Named ranges
   - Section breaks (empty rows, formatting changes)
3. Build cell dependency graph from formula references
```

**Key Techniques:**
- **Header detection:** Check top 3 rows for text-only cells with unique values
- **Table detection:** Find rectangular regions of similar data types
- **Section detection:** Look for bold/formatted cells followed by data blocks

#### Step 2: Concept Extraction
```
Input: Structured spreadsheet representation
Output: List of "concepts" (meaningful cells/ranges)

Process:
1. Identify meaningful cells:
   - Cells with formulas (likely calculations)
   - Cells in header positions (likely labels)
   - Cells with special formatting (bold, color, borders)
   - Named ranges
   - Cells referenced by other formulas

2. Filter out noise:
   - Empty cells
   - Pure data cells without context (unless part of a metric)
   - Temporary calculations

3. Group related cells:
   - Detect metric-value pairs (header + data cell)
   - Detect time series (header + row/column of values)
   - Detect comparative groups (Budget vs Actual)
```

**Concept Types:**
- **Metrics:** Calculated values with business meaning (KPIs)
- **Dimensions:** Categorical labels (Department, Region)
- **Data:** Raw values in context (Revenue data points)
- **Calculations:** Intermediate formula results

#### Step 3: Semantic Enrichment
```
Input: Extracted concepts
Output: Concepts with rich semantic metadata

Process for each concept:

A. Text-based semantics:
   1. Extract concept name:
      - From header if available
      - From named range
      - From nearby labels
      - Generate from context if needed

   2. Build context text:
      - Concatenate: sheet name + section + headers + cell comment
      - Example: "Q4 Financials / Profitability Metrics / Gross Profit Margin"

   3. Extract keywords from context text

B. Formula semantics:
   1. Parse formula into AST (Abstract Syntax Tree)
   2. Classify formula pattern:
      - Margin calculation: (A-B)/A
      - Growth: (New-Old)/Old
      - Ratio: A/B
      - Aggregation: SUM, AVERAGE, COUNT
      - Lookup: VLOOKUP, INDEX/MATCH

   3. Resolve input cell meanings:
      - Recursively extract concepts for referenced cells
      - Map variables: "A=Revenue, B=COGS"

   4. Generate formula description:
      - "Calculates gross margin as (Revenue - COGS) / Revenue"

C. Business domain classification:
   1. Match against business ontology:
      - Finance terms: revenue, profit, margin, ROI
      - Marketing terms: CAC, CLV, conversion, impressions
      - Sales terms: quota, pipeline, close rate
      - Operations terms: utilization, throughput, efficiency

   2. Assign domain & category:
      - Domain: finance, marketing, sales, ops
      - Category: profitability, growth, efficiency, etc.

   3. Expand with synonyms from ontology:
      - "Gross Profit Margin" → ["gross margin", "GM%", "margin"]

D. Value analysis:
   1. Detect data type: number, percentage, currency, date
   2. Analyze value pattern:
      - Is it a KPI? (formatted, prominent position)
      - Is it changing? (time series)
      - What's the typical range? (for outlier detection)
```

#### Step 4: Embedding Generation
```
Input: Enriched concepts
Output: Vector embeddings

Process:
1. Generate text for embedding:
   - Concept text: "[concept_name] [context] [formula_description]"
   - Example: "Gross Profit Margin in Q4 Financials Profitability section,
               calculates margin as (Revenue minus COGS) divided by Revenue"

2. Use pre-trained embedding model:
   - Option A: OpenAI text-embedding-3-large (3072-d, high quality)
   - Option B: sentence-transformers/all-mpnet-base-v2 (768-d, open-source)
   - Option C: Fine-tuned domain-specific model

3. Generate multiple embeddings per concept:
   - concept_embedding: From concept name + context
   - formula_embedding: From formula description
   - context_embedding: From surrounding structure

4. Store in vector database with metadata
```

**Embedding Strategy:**
- **Semantic clustering:** Similar concepts have similar embeddings
- **Query matching:** Query embedding compared to concept embeddings
- **Multi-vector:** Different embeddings capture different aspects

#### Step 5: Index Building
```
Build multiple indexes:

1. Vector Index (for semantic search):
   - Store concept embeddings in vector DB (Pinecone/Qdrant)
   - Enable ANN search with HNSW or IVF algorithm
   - Index size: ~768 floats × N concepts

2. Inverted Index (for keyword search):
   - Tokenize concept names, headers, formulas
   - Build posting lists: term → concept IDs
   - Support phrase queries and Boolean operators
   - Store in Elasticsearch/OpenSearch

3. Formula Pattern Index:
   - Extract normalized formula patterns
   - Map pattern → concept IDs
   - Enable functional queries ("show percentage calculations")

4. Relationship Graph:
   - Build directed graph of cell dependencies
   - Detect cross-sheet references
   - Find semantically similar concepts across sheets
   - Store in Neo4j/graph DB
```

---

### 2. Query Processing Pipeline (Online)

#### Step 1: Query Understanding
```
Input: Natural language query from user
Output: Structured query representation

Process:

1. Query normalization:
   - Lowercase, remove punctuation
   - Expand contractions
   - Spell correction (using edit distance or LLM)

2. Intent classification (using LLM or rule patterns):

   A. Query Type Detection:
      - Conceptual: "find profitability metrics"
      - Functional: "show percentage calculations"
      - Comparative: "budget vs actual analysis"
      - Navigational: "where is the revenue cell"

   B. Entity Extraction:
      - Business concepts: [profitability, margin, ROI]
      - Operations: [calculation, comparison, sum]
      - Filters: [Q4, 2024, finance department]

   C. Intent Structure:
      {
        "query_type": "conceptual",
        "concepts": ["profitability"],
        "operations": [],
        "filters": {},
        "comparison": null
      }

3. Query expansion:

   A. Synonym expansion (from business ontology):
      - "profitability" → ["profit", "earnings", "margin", "ROI"]
      - "sales" → ["revenue", "income", "bookings"]

   B. Related concept expansion:
      - "margin" → include "revenue" and "cost" (components)
      - "budget" → include "actual" and "variance" (related)

   C. Hypernym/hyponym expansion:
      - "metrics" → specific metrics (expand down)
      - "gross margin" → "margin" (expand up)

4. Generate query embedding:
   - Create text: "[expanded_query] [intent_type] [concepts]"
   - Pass through same embedding model as indexing
   - Result: 768-d vector for semantic similarity
```

**Example:**
```
User query: "find profit calculations"

After processing:
{
  "original": "find profit calculations",
  "normalized": "profit calculations",
  "query_type": "conceptual",
  "concepts": ["profit", "earnings", "margin", "profitability"],
  "operations": ["calculation"],
  "embedding": [0.23, 0.45, ...],
  "expanded_query": "profit earnings margin profitability calculations formulas"
}
```

#### Step 2: Multi-Index Retrieval
```
Retrieve candidate concepts from multiple indexes in parallel:

1. Semantic retrieval (Vector DB):
   - Query: Find top-K nearest neighbors to query embedding
   - Method: Approximate Nearest Neighbor (ANN) search
   - K: 50-100 candidates
   - Output: Concepts with similarity scores (0-1)

2. Keyword retrieval (Inverted Index):
   - Query: Boolean query with expanded terms
   - Example: "profit OR earnings OR margin" + "calculations OR formulas"
   - Output: Concepts with TF-IDF scores
   - Boost: Matches in concept_name > header > formula

3. Formula pattern retrieval (if functional query):
   - Query: Match formula pattern
   - Example: "percentage calculations" → patterns like "A/B", "(A-B)/A"
   - Output: Concepts with matching formulas

4. Graph retrieval (if comparative or multi-sheet):
   - Query: Graph pattern matching
   - Example: "budget vs actual" → find concepts with similar names on Budget & Actual sheets
   - Output: Concept pairs/groups with relationships

Result: Union of candidates from all indexes (typically 100-200 total)
```

#### Step 3: Re-Ranking with Hybrid Scoring

```
Input: Candidate concepts from multiple retrievers
Output: Ranked list of top-N results

Hybrid Scoring Function:
score = w1·semantic_score + w2·keyword_score + w3·formula_score +
        w4·importance_score + w5·recency_score

Where:
- w1, w2, w3, w4, w5 are learned weights (initial: 0.3, 0.25, 0.15, 0.2, 0.1)

Component Scores:

1. Semantic Score (from vector similarity):
   - Cosine similarity between query and concept embeddings
   - Range: [0, 1]
   - Normalization: Apply sigmoid if needed

2. Keyword Score (from inverted index):
   - BM25 or TF-IDF score
   - Exact match bonus: +0.2 if exact phrase match
   - Field boost: concept_name (3x), header (2x), formula (1x)
   - Range: Normalized to [0, 1]

3. Formula Score (for functional/comparative queries):
   - Pattern match: 1.0 if formula pattern matches, 0 otherwise
   - Complexity alignment: Match query complexity to formula complexity
   - Input relevance: Are formula inputs relevant to query?
   - Range: [0, 1]

4. Importance Score (concept significance):
   - Base score from signals:
     * is_kpi: +0.3
     * in_summary_section: +0.2
     * has_conditional_formatting: +0.15
     * reference_count: min(0.35, reference_count/10)
   - Range: [0, 1]

5. Recency Score (if temporal data available):
   - Decay: exp(-days_since_modified / 30)
   - Recently accessed: +0.1
   - Range: [0, 1]

6. Context Bonus:
   - Multi-sheet: +0.1 if cross-sheet match for comparative queries
   - Same domain: +0.05 if business domain matches query domain
   - Related concepts: +0.05 if part of a cluster matching the query

Final Ranking:
1. Calculate composite score for each candidate
2. Apply minimum threshold (e.g., score > 0.4)
3. Sort by score descending
4. Return top-N (default N=10)
```

#### Step 4: Result Explanation Generation

```
For each result in top-N:

1. Identify why it matched:
   - High semantic similarity → "Closely related to [query concept]"
   - Keyword match → "Contains exact term: [keyword]"
   - Formula pattern → "Uses [pattern type] calculation"
   - Cross-sheet → "Related to [concept] in [other sheet]"

2. Extract business context:
   - What: "Gross Profit Margin metric"
   - Where: "Q4 Financials sheet, Profitability section"
   - How: "Calculated as (Revenue - COGS) / Revenue"
   - Value: "Current value: 42%"

3. Generate explanation:
   Template: "[What] in [Where]. [Why matched]. [How calculated]. [Value]"

   Example: "Gross Profit Margin in Q4 Financials, Profitability Metrics
             section. Matched as a profitability metric. Calculated as
             (Revenue - COGS) / Revenue. Current value: 42%"

4. Add navigation info:
   - Cell reference: "Q4 Financials!D15"
   - Named range: "GrossProfitMargin"
   - Clickable link (in UI)

5. Compute confidence:
   - High (>0.75): "Strong match"
   - Medium (0.5-0.75): "Possible match"
   - Low (<0.5): Don't show or "Other potential matches"
```

---

### 3. Multi-Sheet Understanding (Bonus)

```
Cross-Sheet Analysis:

1. Relationship Detection:

   A. Explicit relationships (formula references):
      - Parse formula ASTs for cross-sheet references
      - Example: =Budget!D15 → links Actual and Budget sheets
      - Build bi-directional graph edges

   B. Semantic relationships (similar concepts):
      - Compare concept embeddings across sheets
      - Threshold: similarity > 0.85 → same/related concept
      - Example: "Q4 Revenue" (Actuals) ↔ "Q4 Revenue Budget" (Budget)

   C. Structural relationships (name patterns):
      - Detect sheet naming patterns: Budget, Actual, Forecast
      - Match concepts with same position/structure
      - Example: All sheets have "Revenue" in row 5

2. Comparative Query Handling:

   Query: "budget vs actual analysis"

   Process:
   a. Identify comparison dimensions: [budget, actual]
   b. Find sheets matching dimensions:
      - Sheet "Budget" matches "budget"
      - Sheet "Actuals" matches "actual"
   c. Find corresponding concepts:
      - Use semantic similarity + structural matching
      - Pair concepts: Budget!Revenue ↔ Actuals!Revenue
   d. Include variance/analysis if available:
      - Look for sheets named "Variance", "Analysis"
      - Find concepts using both budget and actual
   e. Rank by:
      - Concept similarity across sheets (higher is better)
      - Presence of comparison formula
      - Importance score

3. Multi-Sheet Result Grouping:

   Output format:
   {
     "result_group": {
       "concept_name": "Q4 Revenue",
       "sheets": [
         {
           "sheet": "Budget",
           "concept_id": "uuid-123",
           "value": "$1,000,000",
           "location": "Budget!D5"
         },
         {
           "sheet": "Actuals",
           "concept_id": "uuid-124",
           "value": "$950,000",
           "location": "Actuals!D5"
         },
         {
           "sheet": "Variance",
           "concept_id": "uuid-125",
           "value": "-$50,000 (-5%)",
           "location": "Variance!D5",
           "formula": "=Actuals!D5-Budget!D5"
         }
       ],
       "explanation": "Revenue across Budget, Actuals, and Variance sheets"
     }
   }
```

---

## Technology Stack

### Required Components

1. **Embedding Model:**
   - **Option A:** OpenAI API (text-embedding-3-large)
     - Pros: High quality, 3072 dimensions
     - Cons: API cost, latency

   - **Option B:** Self-hosted (sentence-transformers)
     - Model: all-mpnet-base-v2 or all-MiniLM-L12-v2
     - Pros: Free, fast, private
     - Cons: Lower quality than commercial

   - **Recommendation:** Start with OpenAI, migrate to self-hosted if cost is issue

2. **Vector Database:**
   - **Options:** Pinecone (managed), Qdrant (open-source), pgvector (PostgreSQL)
   - **Requirements:** ANN search, metadata filtering, <200ms query time
   - **Recommendation:** Qdrant (good balance of features and ease of use)

3. **Search Engine:**
   - **Options:** Elasticsearch, OpenSearch
   - **Requirements:** Inverted index, BM25 scoring, phrase queries
   - **Recommendation:** Elasticsearch (mature, well-documented)

4. **Graph Database:**
   - **Options:** Neo4j, Amazon Neptune
   - **Requirements:** Graph traversal, pattern matching
   - **Recommendation:** Neo4j (excellent query language, visualization)

5. **LLM (Optional but Recommended):**
   - **Use cases:** Query understanding, formula description, result explanation
   - **Options:** GPT-4, Claude, or open-source (Llama)
   - **Recommendation:** GPT-4-turbo (good balance of cost and quality)

---

## Workflow Example

### Scenario: User queries "find all profitability metrics"

#### Indexing (already done):
- Spreadsheet parsed, concepts extracted
- Embeddings generated for all concepts
- Indexes built (vector, keyword, formula, graph)

#### Query Processing:

```
1. Query Understanding:
   - Input: "find all profitability metrics"
   - Normalized: "profitability metrics"
   - Intent: conceptual query
   - Expanded: "profitability profit earnings margin metrics KPIs"
   - Embedding: [0.234, 0.456, ..., 0.789]

2. Multi-Index Retrieval:

   A. Vector DB query:
      - Top 50 by cosine similarity
      - Results: Gross Margin (0.94), Net Profit (0.91), ROI (0.88), ...

   B. Keyword query:
      - BM25 query: "profitability OR profit OR earnings OR margin"
      - Results: Profit Margin (score 2.3), EBITDA (score 1.8), ...

   C. Formula pattern:
      - Not applicable (not a functional query)

   D. Graph query:
      - Find concepts in "profitability" cluster

   Union: ~80 unique candidate concepts

3. Hybrid Re-Ranking:

   Candidate: "Gross Profit Margin" (Q4 Financials!D15)
   - Semantic score: 0.94
   - Keyword score: 0.82 (matches "profit", "margin")
   - Formula score: 0.70 (margin calculation pattern)
   - Importance score: 0.85 (is_kpi=true, in_summary=true)
   - Recency score: 0.95 (modified 2 days ago)

   Final score: 0.3×0.94 + 0.25×0.82 + 0.15×0.70 + 0.2×0.85 + 0.1×0.95
              = 0.282 + 0.205 + 0.105 + 0.170 + 0.095
              = 0.857 (high score!)

   Similar scoring for all candidates...

   Top 5 Results (after ranking):
   1. Gross Profit Margin (0.857)
   2. Net Profit Margin (0.842)
   3. EBITDA Margin (0.791)
   4. Operating Profit (0.768)
   5. ROI (0.742)

4. Explanation Generation:

   Result 1: Gross Profit Margin
   {
     "concept": "Gross Profit Margin",
     "location": "Q4 Financials!D15",
     "display_location": "Q4 Financials sheet, row 15, Profitability Metrics section",
     "value": "42%",
     "formula": "=(D13-D14)/D13",
     "formula_description": "Calculated as (Revenue - COGS) / Revenue",
     "explanation": "Gross Profit Margin is a key profitability metric. Found in Q4 Financials sheet, Profitability Metrics section. Calculated as (Revenue - COGS) / Revenue. Current value: 42%",
     "confidence": 0.857,
     "match_reason": "Strong semantic match to 'profitability metrics' + exact match on 'margin'"
   }

   Return top 5 results to user.
```

---

## Accuracy Factors

### What Makes This Approach Accurate?

1. **Semantic Understanding (Embeddings):**
   - Handles synonyms naturally ("profit" ≈ "earnings")
   - Captures concept relationships (margin relates to revenue and cost)
   - Works on novel queries not seen before

2. **Precision (Symbolic Rules):**
   - Keyword matching catches exact terms
   - Formula pattern matching ensures functional accuracy
   - Business ontology provides domain expertise

3. **Context-Aware Ranking:**
   - Multiple signals (not just similarity)
   - Importance weighting (prioritize KPIs)
   - Formula semantics (understand calculation meaning)

4. **Cross-Sheet Intelligence:**
   - Detects relationships across tabs
   - Groups related concepts logically
   - Handles comparative queries effectively

5. **Explainability:**
   - Clear match reasons build trust
   - Business context helps users understand results
   - Formula descriptions clarify calculations

### Potential Accuracy Issues:

1. **Ambiguous Queries:**
   - "Revenue" could mean total, regional, product-specific
   - Solution: Ask clarifying questions or show grouped results

2. **Poor Spreadsheet Structure:**
   - No headers, inconsistent naming, ad-hoc layout
   - Solution: Fallback to keyword search, highlight low confidence

3. **Domain Mismatch:**
   - Business ontology doesn't cover user's specific domain
   - Solution: Allow custom synonym dictionaries, learn from feedback

4. **Embedding Quality:**
   - Off-the-shelf models may not capture business nuances
   - Solution: Fine-tune on business text corpus, use larger models

5. **Weight Tuning:**
   - Default weights may not be optimal for all users
   - Solution: A/B test, learn from click-through data, personalize

---

## Advantages of Hybrid Approach

1. **Best of Both Worlds:**
   - Semantic flexibility from embeddings
   - Precision from symbolic rules

2. **Graceful Degradation:**
   - If embeddings fail, keywords still work
   - If keywords miss, semantics can catch

3. **Explainable:**
   - Can always point to specific match reasons
   - Not a black box

4. **Extensible:**
   - Easy to add new signals (user feedback, access patterns)
   - Can fine-tune individual components

5. **Production-Ready:**
   - Proven technologies (Elasticsearch + vector DB)
   - Scales to large spreadsheets
   - Acceptable latency (<2s)

---

## Disadvantages of Hybrid Approach

1. **Complexity:**
   - Multiple systems to maintain
   - More moving parts, more failure modes

2. **Cost:**
   - Embedding API costs (if using OpenAI)
   - Multiple database licenses/hosting
   - Higher compute requirements

3. **Latency:**
   - Multiple index queries in parallel
   - Embedding generation for queries
   - Target: <2s, but can be challenging

4. **Cold Start:**
   - Requires embeddings to be generated upfront
   - Initial indexing takes time (minutes for large sheets)

5. **Tuning Required:**
   - Weight balancing is non-trivial
   - May need per-domain customization

---

## Summary

**Approach 1 (Hybrid)** is recommended because it:
- Achieves high accuracy on diverse query types
- Handles novel/unseen queries well
- Provides good explanations
- Uses proven technologies
- Scales to production workloads

The combination of embeddings (for semantics) and symbolic rules (for precision) provides the best accuracy profile for the stated goal.
