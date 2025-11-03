# Data Structures for Semantic Search

## Overview

The data structures are designed to support fast semantic search while preserving spreadsheet structure and enabling rich explanations. Key design principles:

1. **Multi-index architecture:** Different indexes for different query types
2. **Denormalization:** Trade storage for query speed
3. **Rich metadata:** Capture context for ranking and explanation
4. **Graph relationships:** Model cross-sheet and cell dependencies

---

## Core Data Structures

### 1. Cell Concept Index (Primary Structure)

Each semantically meaningful cell/range gets indexed as a "concept" with rich metadata.

```json
{
  "concept_id": "uuid-1234",
  "spreadsheet_id": "sheet-abc",
  "location": {
    "sheet_name": "Q4 Financials",
    "sheet_id": "sheet-1",
    "cell_range": "D15:D15",
    "display_reference": "Q4 Financials!D15",
    "row": 15,
    "col": 4,
    "named_range": "GrossProfitMargin"  // if exists
  },

  "semantic_metadata": {
    "concept_name": "Gross Profit Margin",  // Derived from context
    "concept_type": "metric",  // metric, dimension, calculation, data
    "business_domain": "finance",  // finance, marketing, sales, ops
    "metric_category": "profitability",  // if type=metric
    "aliases": ["gross margin", "profit margin", "margin %"]
  },

  "content": {
    "value": 0.42,
    "value_type": "percentage",
    "display_value": "42%",
    "formula": "=(D13-D14)/D13",
    "formula_ast": {...},  // Parsed formula structure
    "has_formula": true
  },

  "context": {
    "header": "Gross Profit Margin",  // Column or row header
    "header_path": ["Financials", "Profitability", "Gross Profit Margin"],  // Hierarchical
    "section": "Profitability Metrics",  // Detected section name
    "surrounding_headers": {
      "top": "Profitability Metrics",
      "left": "Q4 2024"
    },
    "table_name": "Financial Summary",  // If part of a table
    "nearby_keywords": ["revenue", "COGS", "margin", "profit"]
  },

  "formula_semantics": {
    "operation_type": "margin_calculation",  // Detected pattern
    "operation_pattern": "(revenue - cost) / revenue",
    "inputs": [
      {"cell": "D13", "concept": "Revenue"},
      {"cell": "D14", "concept": "COGS"}
    ],
    "complexity_score": 2,  // 1-10 scale
    "functions_used": ["division", "subtraction"]
  },

  "embeddings": {
    "concept_embedding": [0.123, 0.456, ...],  // 384-d or 768-d vector
    "formula_embedding": [0.789, 0.012, ...],
    "context_embedding": [0.345, 0.678, ...]
  },

  "relationships": {
    "depends_on": ["uuid-1122", "uuid-1133"],  // Cells this depends on
    "used_by": ["uuid-1155"],  // Cells that use this
    "related_concepts": ["uuid-2234", "uuid-2235"],  // Same concept, diff sheet
    "cross_sheet_refs": [
      {"sheet": "Budget", "concept": "Budgeted Gross Margin"}
    ]
  },

  "importance_signals": {
    "is_kpi": true,  // Detected as key metric
    "in_summary_section": true,
    "has_conditional_formatting": true,
    "cell_format": "percentage_2dp_bold",
    "has_comment": false,
    "reference_count": 5  // How many other cells reference this
  },

  "temporal_metadata": {
    "last_modified": "2024-11-01T10:30:00Z",
    "creation_date": "2024-10-01T09:00:00Z",
    "access_count": 45,  // If tracking enabled
    "last_accessed": "2024-11-03T14:20:00Z"
  },

  "confidence_scores": {
    "concept_extraction": 0.92,  // How confident are we about the concept name
    "formula_semantic": 0.88,
    "business_domain": 0.95
  }
}
```

**Why this structure:**
- **Rich context:** Multiple fields capture different aspects for ranking
- **Embeddings:** Enable semantic similarity search
- **Relationships:** Support multi-sheet understanding
- **Metadata:** Enable explanation generation
- **Denormalized:** Fast retrieval without joins

---

### 2. Inverted Index (Symbolic Search)

Traditional inverted index for fast keyword and pattern matching.

```json
{
  "term": "profit",
  "document_frequency": 12,
  "postings": [
    {
      "concept_id": "uuid-1234",
      "positions": [2, 5],  // Term positions in concept text
      "field": "concept_name",  // Which field matched
      "tf_idf_score": 0.73
    },
    {
      "concept_id": "uuid-1235",
      "positions": [0],
      "field": "header",
      "tf_idf_score": 0.65
    }
  ]
}
```

**Indexed fields:**
- Concept names
- Headers (column/row)
- Section names
- Formula text
- Cell comments
- Sheet names

**Why this structure:**
- **Fast exact matching:** O(1) term lookup
- **TF-IDF scoring:** Standard IR metric
- **Position information:** Enable phrase queries
- **Field-specific:** Can boost matches in certain fields

---

### 3. Formula Pattern Index

Index of formula patterns for functional queries ("show percentage calculations").

```json
{
  "pattern_id": "margin_calculation",
  "pattern_template": "(A - B) / A",
  "pattern_type": "margin_percentage",
  "semantic_meaning": "Calculates margin as (revenue - cost) / revenue",
  "examples": [
    {
      "concept_id": "uuid-1234",
      "actual_formula": "=(D13-D14)/D13",
      "variable_mapping": {"A": "D13:Revenue", "B": "D14:COGS"}
    }
  ],
  "match_count": 8,
  "confidence": 0.94
}
```

**Pattern categories:**
- Percentage calculations: `A/B`, `(A-B)/A`
- Growth/change: `(New-Old)/Old`
- Averages: `SUM()/COUNT()`, `AVERAGE()`
- Ratios: `A/B` where both are same unit
- Aggregations: `SUM`, `SUMIF`, `COUNTIF`
- Lookups: `VLOOKUP`, `INDEX/MATCH`

**Why this structure:**
- **Pattern matching:** Recognize calculation types regardless of exact formulas
- **Semantic labels:** Map formulas to business concepts
- **Reusable:** Same pattern may appear many times

---

### 4. Concept Taxonomy / Business Ontology

Hierarchical structure of business concepts and their relationships.

```json
{
  "domain": "finance",
  "categories": {
    "profitability": {
      "synonyms": ["profit", "earnings", "margins"],
      "metrics": [
        {
          "name": "Gross Profit Margin",
          "synonyms": ["gross margin", "GM%", "gross profit %"],
          "formula_patterns": ["margin_calculation"],
          "typical_values": {"type": "percentage", "range": [0, 1]},
          "related_metrics": ["Revenue", "COGS", "Net Profit Margin"]
        },
        {
          "name": "Net Profit Margin",
          "synonyms": ["net margin", "bottom line margin", "NM%"],
          "formula_patterns": ["margin_calculation"],
          "typical_values": {"type": "percentage", "range": [0, 1]},
          "related_metrics": ["Net Income", "Revenue", "Gross Profit Margin"]
        }
      ]
    },
    "efficiency": {
      "synonyms": ["efficiency", "productivity", "utilization"],
      "metrics": [
        {
          "name": "ROI",
          "synonyms": ["return on investment", "ROI%"],
          "formula_patterns": ["(Gain - Cost) / Cost"],
          "typical_values": {"type": "percentage", "range": [-1, 10]}
        }
      ]
    }
  }
}
```

**Why this structure:**
- **Synonym resolution:** "profit" → "earnings", "margin"
- **Concept hierarchies:** "profitability" contains specific metrics
- **Domain knowledge:** Encode business expertise
- **Query expansion:** Enrich user queries with related terms

---

### 5. Multi-Sheet Relationship Graph

Graph structure capturing relationships across sheets.

```json
{
  "nodes": [
    {
      "node_id": "concept-uuid-1234",
      "node_type": "concept",
      "sheet": "Actuals",
      "concept_name": "Q4 Revenue"
    },
    {
      "node_id": "concept-uuid-1235",
      "node_type": "concept",
      "sheet": "Budget",
      "concept_name": "Q4 Budgeted Revenue"
    },
    {
      "node_id": "concept-uuid-1236",
      "node_type": "concept",
      "sheet": "Variance Analysis",
      "concept_name": "Q4 Revenue Variance"
    }
  ],

  "edges": [
    {
      "source": "concept-uuid-1234",
      "target": "concept-uuid-1236",
      "relationship_type": "used_in_calculation",
      "formula_ref": "=Actuals!D15"
    },
    {
      "source": "concept-uuid-1235",
      "target": "concept-uuid-1236",
      "relationship_type": "used_in_calculation",
      "formula_ref": "=Budget!D15"
    },
    {
      "source": "concept-uuid-1234",
      "target": "concept-uuid-1235",
      "relationship_type": "semantic_similarity",
      "similarity_score": 0.95,
      "semantic_relation": "actual_vs_budget"
    }
  ],

  "clusters": [
    {
      "cluster_id": "revenue-analysis",
      "concepts": ["concept-uuid-1234", "concept-uuid-1235", "concept-uuid-1236"],
      "theme": "Q4 Revenue Tracking"
    }
  ]
}
```

**Why this structure:**
- **Cross-sheet queries:** "Find budget vs actual analysis"
- **Relationship detection:** Identify conceptually related cells
- **Navigation:** Help users explore related concepts
- **Clustering:** Group related metrics across sheets

---

### 6. Query Cache & Learning Store

Store query history and user feedback for learning.

```json
{
  "query_id": "query-uuid-789",
  "query_text": "find profitability metrics",
  "normalized_query": "profitability metrics",
  "query_embedding": [0.123, 0.456, ...],
  "intent": {
    "query_type": "conceptual",
    "business_domain": "finance",
    "metric_category": "profitability"
  },

  "results": [
    {
      "rank": 1,
      "concept_id": "uuid-1234",
      "score": 0.92,
      "clicked": true,
      "dwell_time_seconds": 45,
      "user_rating": 5
    },
    {
      "rank": 2,
      "concept_id": "uuid-1235",
      "score": 0.88,
      "clicked": false,
      "user_rating": null
    }
  ],

  "performance": {
    "retrieval_time_ms": 45,
    "ranking_time_ms": 12,
    "total_time_ms": 57
  },

  "timestamp": "2024-11-03T14:30:00Z",
  "user_id": "user-123",
  "spreadsheet_id": "sheet-abc"
}
```

**Why this structure:**
- **Learning:** Improve ranking from user behavior
- **Caching:** Speed up repeated queries
- **Analytics:** Understand query patterns
- **Personalization:** User-specific improvements

---

## Storage & Indexing Technologies

### For Production Implementation

1. **Vector Database** (Embeddings)
   - **Options:** Pinecone, Weaviate, Qdrant, or pgvector (PostgreSQL extension)
   - **Usage:** Store concept embeddings for semantic similarity search
   - **Query:** Approximate Nearest Neighbor (ANN) search

2. **Search Engine** (Inverted Index)
   - **Options:** Elasticsearch, OpenSearch, or Apache Solr
   - **Usage:** Fast keyword search, TF-IDF scoring, filtering
   - **Query:** Boolean queries with scoring

3. **Graph Database** (Relationships)
   - **Options:** Neo4j, Amazon Neptune, or ArangoDB
   - **Usage:** Multi-sheet relationships, dependency tracking
   - **Query:** Graph traversal, pattern matching

4. **Primary Database** (Metadata)
   - **Options:** PostgreSQL or MongoDB
   - **Usage:** Store cell metadata, concept details
   - **Query:** Structured queries, joins

5. **Cache Layer**
   - **Options:** Redis or Memcached
   - **Usage:** Query results, frequently accessed concepts
   - **Query:** Key-value lookups

---

## Data Structure Trade-offs

### Storage Overhead
- **High denormalization:** Concept index stores redundant data
- **Multiple indexes:** Same data in vector DB, search engine, graph DB
- **Trade-off:** Accept 3-5x storage overhead for query speed

### Update Complexity
- **Challenge:** Cell change requires updating multiple indexes
- **Solution:** Batch updates, incremental indexing
- **Real-time:** Update inverted index immediately, rebuild embeddings async

### Consistency
- **Challenge:** Multiple data stores may become inconsistent
- **Solution:** Event-driven architecture with eventual consistency
- **Critical path:** Inverted index must be consistent; embeddings can lag

---

## Indexing Strategy

### Initial Indexing (Full Spreadsheet)
1. **Parse spreadsheet:** Extract all cells, formulas, structure
2. **Detect concepts:** Identify meaningful cells/ranges
3. **Extract semantics:** Analyze formulas, headers, context
4. **Generate embeddings:** Create vector representations
5. **Build indexes:** Populate inverted index, vector DB, graph DB
6. **Estimate:** ~1-5 seconds for typical spreadsheet (1000 cells)

### Incremental Updates (Cell Edit)
1. **Change event:** User edits cell
2. **Update concept:** Recalculate affected concept metadata
3. **Update indexes:**
   - Inverted index: Immediate update (~10ms)
   - Vector DB: Async embedding generation + update (~100-500ms)
   - Graph DB: Update relationships (~50ms)
4. **Invalidate cache:** Clear affected query caches

### Batch Re-indexing
- **Trigger:** Major structural changes, new sheets
- **Frequency:** Daily background job for optimization
- **Process:** Full re-analysis, updated embeddings, graph rebuild

---

## Memory & Performance Considerations

### In-Memory Structures (Query Time)
- **Query embedding:** Generated once per query (~50ms)
- **Top-K candidates:** Retrieved from vector DB (~50-200ms)
- **Ranking:** Score all candidates (~10-50ms)
- **Total:** Target < 2 seconds end-to-end

### Disk Storage Estimates (per 1000 concepts)
- **Concept index:** ~500KB (JSON metadata)
- **Embeddings:** ~300KB (768-d float32)
- **Inverted index:** ~100KB (compressed)
- **Graph relationships:** ~200KB
- **Total:** ~1MB per 1000 concepts

### Scalability
- **Small spreadsheet:** 1K cells → ~100 concepts → 100KB storage
- **Large spreadsheet:** 100K cells → ~10K concepts → 10MB storage
- **Very large:** 1M cells → ~100K concepts → 100MB storage (rare)

---

## Summary: Why These Data Structures?

1. **Multi-index approach:** Each index optimized for specific query patterns
2. **Rich metadata:** Enables accurate ranking and good explanations
3. **Graph relationships:** Critical for multi-sheet understanding
4. **Embeddings:** Handle semantic similarity and novel queries
5. **Caching & learning:** Continuous improvement and performance
6. **Trade-offs:** Accept complexity and storage overhead for accuracy and speed
