# Approach 2: Rules-Based + Heuristics

## Overview

This approach relies on explicit rules, keyword dictionaries, and formula pattern matching to achieve semantic search without neural embeddings. It's a more traditional information retrieval approach enhanced with spreadsheet-specific heuristics.

**Philosophy:** Use expert-crafted rules and domain knowledge to map queries to concepts, prioritizing precision and explainability over coverage of novel queries.

---

## Architecture Components

### 1. Indexing Pipeline (Offline/On-Change)

#### Step 1: Spreadsheet Parsing (Same as Approach 1)
- Extract sheets, cells, formulas, formats
- Detect structure (headers, tables, sections)
- Build dependency graph

#### Step 2: Concept Extraction with Rule-Based Classification

```
Process for each cell:

1. Apply Classification Rules:

   Rule 1: Is it a Metric?
   - Has formula? → Score +2
   - In bold? → Score +1
   - Has percentage/currency format? → Score +1
   - Referenced by other cells? → Score +ref_count/10
   - Near "Total" or "Summary" keywords? → Score +1
   - Threshold: Score ≥ 3 → Metric

   Rule 2: Is it a Header?
   - In top 3 rows? → Likely header
   - Text type + next cell is number? → Likely header
   - Bold or formatted? → Likely header
   - Contains capital letters or colons? → Likely header

   Rule 3: Is it Important?
   - Named range? → Important
   - Referenced by 3+ cells? → Important
   - In first sheet? → More important
   - Near top-left corner? → More important

2. Extract Concept Name:
   - Use header if available (column + row headers)
   - Use named range name
   - Use nearby text label
   - Fallback: Generate from formula (e.g., "Sum of Revenue")

3. Extract Keywords:
   - Tokenize concept name and context
   - Remove stopwords (the, is, in, of)
   - Stem words (running → run, calculated → calculate)
   - Store in inverted index: keyword → concept IDs
```

#### Step 3: Formula Pattern Cataloging

```
Build a library of formula patterns using regex and AST matching:

Pattern Library:

1. Percentage Calculations:
   - Pattern: A/B where A and B are numbers
   - Variations: (A/B)*100, A/B formatted as %
   - Keywords: ["percentage", "percent", "%", "ratio"]
   - Examples: =D5/D10, =(SUM(A1:A10)/B1)*100

2. Margin Calculations:
   - Pattern: (A-B)/A or (A-B)/B
   - Keywords: ["margin", "markup", "spread"]
   - Examples: =(Revenue-COGS)/Revenue

3. Growth/Change:
   - Pattern: (New-Old)/Old
   - Keywords: ["growth", "change", "increase", "YoY", "MoM"]
   - Examples: =(Q4_2024-Q4_2023)/Q4_2023

4. Aggregations:
   - Pattern: SUM(range), AVERAGE(range), COUNT(range)
   - Keywords: ["sum", "total", "average", "mean", "count"]
   - Examples: =SUM(A1:A100), =AVERAGE(Revenue)

5. Conditionals:
   - Pattern: IF, SUMIF, COUNTIF, AVERAGEIF
   - Keywords: ["if", "conditional", "filtered", "criteria"]
   - Examples: =SUMIF(Category,"Sales",Amount)

6. Lookups:
   - Pattern: VLOOKUP, HLOOKUP, INDEX/MATCH, XLOOKUP
   - Keywords: ["lookup", "find", "match", "search"]
   - Examples: =VLOOKUP(ID,Table,2,FALSE)

7. Comparisons:
   - Pattern: A-B (difference), A/B (ratio)
   - Keywords: ["vs", "versus", "difference", "variance", "comparison"]
   - Examples: =Actual-Budget, =Actual/Budget

For each cell with a formula:
1. Parse formula into AST
2. Match against pattern library
3. Assign pattern tags (can have multiple)
4. Extract pattern parameters (what's A, B, etc.)
5. Generate human-readable description
```

#### Step 4: Business Dictionary Construction

```
Manually curated dictionary of business terms and relationships:

Dictionary Structure:

{
  "domains": {
    "finance": {
      "profitability": {
        "terms": [
          {
            "canonical": "Gross Profit Margin",
            "synonyms": ["gross margin", "GM", "gross profit %", "margin"],
            "related": ["revenue", "COGS", "cost of sales"],
            "formula_patterns": ["margin_calculation"],
            "typical_format": "percentage",
            "definition": "Gross profit divided by revenue"
          },
          {
            "canonical": "Net Profit Margin",
            "synonyms": ["net margin", "NM", "net profit %", "bottom line"],
            "related": ["net income", "revenue", "expenses"],
            "formula_patterns": ["margin_calculation"],
            "typical_format": "percentage",
            "definition": "Net income divided by revenue"
          },
          {
            "canonical": "EBITDA",
            "synonyms": ["earnings before interest", "operating profit"],
            "related": ["revenue", "operating expenses", "depreciation"],
            "formula_patterns": ["aggregation"],
            "typical_format": "currency",
            "definition": "Earnings before interest, taxes, depreciation, amortization"
          }
        ]
      },
      "growth": {
        "terms": [
          {
            "canonical": "Year over Year Growth",
            "synonyms": ["YoY growth", "annual growth", "yearly change"],
            "related": ["revenue", "current year", "prior year"],
            "formula_patterns": ["growth_calculation"],
            "typical_format": "percentage"
          }
        ]
      }
    },
    "marketing": {
      "efficiency": {
        "terms": [
          {
            "canonical": "Customer Acquisition Cost",
            "synonyms": ["CAC", "acquisition cost", "cost per customer"],
            "related": ["marketing spend", "new customers", "sales"],
            "formula_patterns": ["ratio"],
            "typical_format": "currency"
          },
          {
            "canonical": "Return on Ad Spend",
            "synonyms": ["ROAS", "ad ROI", "advertising efficiency"],
            "related": ["revenue", "ad spend", "marketing"],
            "formula_patterns": ["ratio"],
            "typical_format": "ratio"
          }
        ]
      }
    }
  }
}

Build inverted indexes:
1. Term → dictionary entry
2. Synonym → canonical term
3. Domain → terms in domain
4. Formula pattern → applicable terms

Size: ~500-1000 curated terms across common business domains
```

#### Step 5: Keyword Indexing

```
Build multiple inverted indexes:

1. Exact keyword index:
   - Tokenize concept names, headers, cell text
   - Index: keyword → [concept_ids]
   - Example: "profit" → [concept_123, concept_456, ...]

2. Stemmed keyword index:
   - Stem keywords (profitability → profit)
   - Index: stem → [concept_ids]
   - Handles variations: profit, profits, profitability

3. N-gram index (for partial matching):
   - Generate 2-grams and 3-grams
   - Index: ngram → [concept_ids]
   - Example: "gross profit" → ["gr", "ro", "os", "gro", "ros", "oss", ...]
   - Helps with typos and partial matches

4. Formula text index:
   - Index formula strings (as text)
   - Enables searching within formulas
   - Example: "VLOOKUP" → [all concepts using VLOOKUP]

Storage: Elasticsearch or simple in-memory hash map for small sheets
```

---

### 2. Query Processing Pipeline (Online)

#### Step 1: Query Analysis with Rule-Based NLP

```
Input: Natural language query
Output: Structured query representation

Process:

1. Tokenization & Normalization:
   - Lowercase
   - Remove punctuation
   - Split into tokens: ["find", "profit", "calculations"]

2. Intent Classification (Rule-Based):

   Rule-based patterns:

   Conceptual queries (match against these patterns):
   - "find [concept]" → look for concept by name
   - "show [concept]" → same as find
   - "where is [concept]" → navigational query
   - "[concept] metrics" → find metrics in category
   - "all [concept]" → broad search for concept

   Functional queries:
   - "percentage calculations" → pattern = percentage
   - "show formulas" → any cell with formula
   - "lookup functions" → pattern = lookup
   - "conditional sums" → pattern = SUMIF/COUNTIF

   Comparative queries:
   - "[A] vs [B]" → find A and B, compare
   - "[A] versus [B]" → same
   - "[A] compared to [B]" → same
   - "difference between [A] and [B]" → same
   - "variance analysis" → find variance calculations

3. Entity Extraction (Dictionary Lookup):

   For each token/phrase:
   a. Check if it exists in business dictionary
   b. If match, extract:
      - Canonical term
      - Synonyms
      - Related terms
      - Domain
   c. If no match, treat as generic keyword

   Example:
   Query: "show profitability metrics"
   - "profitability" → matches dictionary domain
   - Extract terms: [Gross Profit Margin, Net Profit Margin, EBITDA, ROI, ...]

4. Query Expansion:

   A. Synonym expansion (from dictionary):
      - "profit" → ["profit", "earnings", "margin", "income"]

   B. Related term expansion:
      - "margin" → include "revenue" and "cost" (components)

   C. Phrase expansion:
      - "YoY" → "year over year"
      - "GM" → "gross margin"

5. Generate Query Structure:

   {
     "intent": "conceptual",
     "target_concepts": ["profitability metrics"],
     "keywords": ["profitability", "profit", "earnings", "margin", "metrics"],
     "formula_patterns": [],
     "filters": {},
     "comparison": null
   }
```

#### Step 2: Candidate Retrieval

```
Retrieve candidates using multiple strategies:

1. Dictionary-Based Retrieval:

   If query matched business dictionary:
   a. Get canonical terms from dictionary
   b. For each term, find concepts by:
      - Keyword match on canonical name
      - Keyword match on synonyms
      - Formula pattern match (if applicable)
   c. Score by:
      - Exact match: score = 1.0
      - Synonym match: score = 0.9
      - Related term: score = 0.7
      - Pattern match: score = 0.8

2. Keyword-Based Retrieval:

   For each expanded query keyword:
   a. Lookup in inverted index
   b. Get concept IDs with keyword
   c. Calculate TF-IDF score:
      - TF: How many times keyword appears in concept
      - IDF: log(N / df) where N = total concepts, df = concepts with keyword
   d. Boost by field:
      - Concept name: 3x
      - Header: 2x
      - Formula: 1x
      - Context: 1x

3. Formula Pattern Retrieval:

   If functional query:
   a. Identify target formula pattern
   b. Lookup concepts with that pattern
   c. Score by pattern complexity match

4. Phrase Matching:

   If query contains multi-word phrase:
   a. Search for exact phrase match (bonus score)
   b. Use n-gram index for partial matches
   c. Phrase match bonus: +0.3 to score

Result: Candidate set with initial scores (typically 50-200 concepts)
```

#### Step 3: Rule-Based Ranking

```
Re-rank candidates using explicit scoring rules:

Scoring Function (additive):

base_score = keyword_score (from TF-IDF)

Adjustments (add/multiply):

1. Exact Match Bonus (+0.5):
   - If concept name exactly matches query phrase

2. Dictionary Match Bonus (+0.3):
   - If concept matches canonical term in business dictionary

3. Formula Pattern Match (+0.4):
   - For functional queries, if formula pattern matches

4. Header Field Boost (×1.5):
   - If match is in header field

5. Importance Boost (×1.3):
   - If concept is marked as important (KPI, named range, etc.)

6. Sheet Position Boost (+0.1 to +0.3):
   - First sheet: +0.3
   - First 10 rows: +0.2
   - First 5 columns: +0.1

7. Recency Boost (+0.1):
   - If modified within last 7 days

8. Reference Count Boost (×1.1 to ×1.5):
   - Multiply by min(1.5, 1 + ref_count/20)

9. Format Boost (+0.2):
   - If cell has special formatting (bold, color, borders)

10. Named Range Boost (+0.3):
    - If concept is a named range

11. Comment Boost (+0.1):
    - If cell has a comment

12. Cross-Sheet Reference (for comparative queries):
    - If query is comparative and concept exists in multiple sheets: +0.4

Penalty Rules:

1. Missing Context Penalty (×0.7):
   - If concept has no clear header or section

2. Data Cell Penalty (×0.5):
   - If concept is just a data value without calculation

3. Low Confidence Penalty (×0.8):
   - If extraction confidence is low

Final Ranking:
1. Apply all rules to get final score
2. Filter: score > threshold (0.3)
3. Sort by score descending
4. Return top N (default N=10)
```

#### Step 4: Explanation Generation

```
For each result, generate explanation using templates:

1. Identify match reasons (rule-based):
   - Exact match: "Exact match on concept name"
   - Dictionary match: "Matches business term: [term]"
   - Keyword match: "Contains keyword: [keyword]"
   - Pattern match: "Uses [pattern type] calculation"
   - Cross-sheet: "Found across [sheet names]"

2. Fill explanation template:

   Template: "[Concept Name] in [Location]. [Match Reason]. [Formula]. [Value]."

   Example: "Gross Profit Margin in Q4 Financials sheet, row 15. Matches business
             term for profitability metric. Calculated as (Revenue-COGS)/Revenue.
             Current value: 42%"

3. Confidence level (rule-based):
   - score > 0.8: "High confidence"
   - score 0.5-0.8: "Medium confidence"
   - score < 0.5: "Low confidence" (may not show)

4. Add navigation info:
   - Cell reference with clickable link
   - Named range if exists
```

---

### 3. Multi-Sheet Understanding

```
Handle multi-sheet queries using heuristics:

1. Sheet Name Matching:

   Rule: If query contains words like "budget", "actual", "forecast", "variance"
   Action: Identify sheets with matching names

   Example:
   Query: "budget vs actual revenue"
   - Find sheets: "Budget" and "Actuals"
   - Search for "revenue" in both sheets
   - Return paired results

2. Structural Matching:

   Rule: Assume similar structure across related sheets
   Action: Find concepts at same position in different sheets

   Example:
   - "Revenue" at Budget!D5
   - Look for "Revenue" at Actuals!D5
   - If found, consider them related

3. Formula-Based Relationships:

   Rule: If formula references another sheet, they're related
   Action: Follow cross-sheet references

   Example:
   - Variance!D5 has formula =Actuals!D5-Budget!D5
   - All three cells are related

4. Naming Pattern Detection:

   Heuristics for related sheets:
   - Same prefix: "Sales_Q1", "Sales_Q2", "Sales_Q3"
   - Standard naming: "Budget", "Actual", "Forecast", "Variance"
   - Year/period suffixes: "Data_2023", "Data_2024"

5. Result Grouping:

   For comparative queries:
   - Group results by concept name
   - Show all sheets for each concept
   - Highlight differences/variances
```

---

## Technology Stack

### Simpler than Approach 1

1. **Search Engine:**
   - Elasticsearch or OpenSearch for inverted index
   - OR simple in-memory index (e.g., Python dict) for small sheets
   - BM25 scoring built-in

2. **Business Dictionary:**
   - JSON file or YAML configuration
   - Loaded into memory at startup
   - ~1-5 MB size for comprehensive dictionary

3. **Formula Parser:**
   - AST parser for Excel/Sheets formulas
   - Libraries: pycel, formulas (Python), grammar-based parser
   - Pattern matching with regex + AST traversal

4. **Rule Engine:**
   - Simple if-then rules in code
   - OR rule engine library (e.g., Python's Experta)
   - Easy to add/modify rules

5. **Storage:**
   - Primary database: PostgreSQL or SQLite (for smaller deployments)
   - No vector database needed
   - No graph database needed (can use SQL joins)

---

## Workflow Example

### Scenario: User queries "show profitability metrics"

```
1. Query Analysis:

   Input: "show profitability metrics"

   A. Pattern matching:
      - Matches pattern: "show [concept]" → conceptual query

   B. Dictionary lookup:
      - "profitability" found in finance domain
      - Category: profitability
      - Related terms: profit, earnings, margin, EBITDA, ROI

   C. Expansion:
      - Keywords: ["profitability", "profit", "earnings", "margin", "EBITDA", "ROI", "metrics"]

   D. Query structure:
      {
        "intent": "conceptual",
        "domain": "finance",
        "category": "profitability",
        "keywords": ["profitability", "profit", "earnings", "margin", "metrics"],
        "dictionary_terms": ["Gross Profit Margin", "Net Profit Margin", "EBITDA", "ROI"]
      }

2. Candidate Retrieval:

   A. Dictionary-based:
      - For each dictionary term, find concepts by keyword
      - "Gross Profit Margin" → [concept_123, concept_156]
      - "Net Profit Margin" → [concept_124]
      - "EBITDA" → [concept_125]
      - Scores: 0.9-1.0 (dictionary match)

   B. Keyword-based:
      - TF-IDF search for expanded keywords
      - "profit" → [concept_123, concept_124, concept_135, ...]
      - "margin" → [concept_123, concept_124, concept_127, ...]
      - Scores: 0.4-0.8 (keyword relevance)

   Union: ~50 candidates

3. Rule-Based Ranking:

   Candidate: "Gross Profit Margin" (Q4 Financials!D15)
   - base_score: 0.8 (keyword TF-IDF)
   - Dictionary match: +0.3 → 1.1
   - Exact match: no (query is "profitability metrics", not "gross profit margin")
   - Important: ×1.3 → 1.43
   - First sheet: +0.3 → 1.73
   - Named range: +0.3 → 2.03
   - Format (bold): +0.2 → 2.23
   - Final score: 2.23

   Candidate: "Operating Profit" (Details!H45)
   - base_score: 0.6 (keyword match on "profit")
   - Dictionary match: no
   - Important: ×1.3 → 0.78
   - Not first sheet: 0
   - Not named range: 0
   - Format: +0.2 → 0.98
   - Final score: 0.98

   After ranking all candidates:
   Top 5:
   1. Gross Profit Margin (2.23)
   2. Net Profit Margin (2.15)
   3. EBITDA Margin (1.94)
   4. Operating Profit (0.98)
   5. ROI (0.87)

4. Explanation Generation:

   Result 1: Gross Profit Margin
   {
     "concept": "Gross Profit Margin",
     "location": "Q4 Financials!D15",
     "value": "42%",
     "formula": "=(D13-D14)/D13",
     "explanation": "Gross Profit Margin in Q4 Financials sheet, row 15. Matches business term for profitability metric. Calculated as (Revenue-COGS)/Revenue. Current value: 42%",
     "confidence": "high",
     "score": 2.23,
     "match_reason": "Dictionary match: profitability metric"
   }
```

---

## Accuracy Factors

### What Makes This Approach Accurate?

1. **Explicit Domain Knowledge:**
   - Business dictionary encodes expert understanding
   - Precise synonym mappings
   - Clear formula pattern definitions

2. **Transparent Rules:**
   - Each scoring rule is explainable
   - Easy to debug why something ranked high/low
   - Users can understand match reasons

3. **Formula Pattern Matching:**
   - Accurately identifies calculation types
   - Works well for functional queries
   - Doesn't depend on text descriptions

4. **Reliable for Known Concepts:**
   - High precision on terms in dictionary
   - Consistent results for repeated queries
   - No randomness or model drift

### Accuracy Limitations:

1. **Limited Synonym Coverage:**
   - Only knows synonyms in dictionary
   - Misses novel terms or company-specific jargon
   - Example: "customer happiness score" won't match "NPS" unless explicitly mapped

2. **No Semantic Understanding:**
   - Can't infer relationships beyond explicit rules
   - "Revenue growth" might not match "(Current_Revenue - Prior_Revenue)/Prior_Revenue" if pattern isn't recognized
   - No understanding of context beyond keywords

3. **Maintenance Burden:**
   - Dictionary must be manually updated
   - New business concepts require new rules
   - Domain-specific customization is manual

4. **Brittle Pattern Matching:**
   - Formula patterns must be explicitly defined
   - Complex nested formulas may not match
   - Novel formula structures will be missed

5. **Poor Recall on Novel Queries:**
   - If query uses terms not in dictionary, results degrade
   - No ability to generalize from examples
   - Users must use "right" keywords

---

## Advantages of Rules-Based Approach

1. **Simplicity:**
   - No ML models, no embeddings
   - Easy to understand and debug
   - Lower technical complexity

2. **Low Cost:**
   - No API costs (OpenAI, etc.)
   - Minimal compute requirements
   - Can run on modest hardware

3. **Fast:**
   - No embedding generation latency
   - Simple index lookups
   - Typically <500ms response time

4. **Deterministic:**
   - Same query always returns same results
   - No model randomness
   - Easier to test and validate

5. **Privacy-Friendly:**
   - No data sent to external APIs
   - Everything runs locally
   - No model training on user data

6. **Explainable:**
   - Every score component is traceable
   - Clear match reasons
   - Easy to justify results to users

7. **Easy to Customize:**
   - Add new rules in code
   - Update dictionary JSON file
   - No model retraining needed

---

## Disadvantages of Rules-Based Approach

1. **Limited Generalization:**
   - Only handles patterns explicitly programmed
   - Poor on novel queries or concepts
   - Can't learn from examples

2. **High Maintenance:**
   - Dictionary must be curated and updated
   - Rules need tuning for each domain
   - Doesn't adapt automatically

3. **Lower Recall:**
   - Misses synonyms not in dictionary
   - Misses semantically similar but textually different concepts
   - Overly reliant on keyword matching

4. **Scaling to Multiple Domains:**
   - Each domain needs separate dictionary
   - Rules may conflict across domains
   - Hard to be comprehensive

5. **No Context Understanding:**
   - "Marketing ROI" vs "Financial ROI" treated identically unless explicitly differentiated
   - Can't disambiguate based on context

---

## When to Use Approach 2

**Good fit when:**
- Budget/resources are limited
- Spreadsheets follow standard business formats
- Queries use well-known business terminology
- Precision is more important than recall
- Fast response time is critical
- Privacy concerns prevent external API use
- Team prefers simpler, maintainable systems

**Poor fit when:**
- Users use varied/novel terminology
- Spreadsheets have custom/non-standard structures
- Queries are exploratory ("find things related to X")
- Need to handle many different business domains
- Want system to learn from user feedback

---

## Summary

**Approach 2 (Rules-Based)** is a solid fallback approach that:
- Achieves good precision on known patterns
- Is simple, fast, and cost-effective
- Provides excellent explainability
- Works well for standard business use cases

However, it has lower accuracy on novel queries and requires significant manual curation. Best used as:
- An MVP to validate the concept
- A baseline to compare against ML approaches
- A component within a hybrid system (e.g., Approach 1)
- The primary solution when constraints favor simplicity
