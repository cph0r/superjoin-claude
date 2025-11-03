# Edge Cases, Failure Modes & Spreadsheet Nuances

## Overview

This document catalogs the challenging scenarios, edge cases, and failure modes that a semantic search system must handle to achieve high accuracy. Understanding these nuances is critical for building a robust solution.

---

## Category 1: Spreadsheet Structure Challenges

### 1.1 No Clear Headers

**Scenario:**
```
A        | B      | C
---------|--------|--------
45000    | 50000  | 55000
0.42     | 0.45   | 0.48
```

No labels, just numbers. Hard to determine what these represent.

**Impact on Accuracy:**
- Cannot extract concept names from headers
- Ambiguous semantic meaning
- Low confidence in concept extraction

**Mitigation Strategies:**
1. **Context from formulas:** If cells are used in formulas elsewhere, extract meaning from usage
   - Example: If another cell has `=B1-B2`, might infer B1 and B2 are related metrics
2. **Position heuristics:** Top-left cells more likely to be important
3. **Format inference:** Cell format (currency, percentage) provides hints
4. **Named ranges:** Check if cells are part of named ranges
5. **Fallback:** Low confidence, mark as "unknown concept", avoid returning in search
6. **User feedback:** Learn from user interactions

**Expected Behavior:**
- Return with LOW confidence or not at all
- Suggest query refinement: "Could you describe what you're looking for?"

---

### 1.2 Headers Not in First Row

**Scenario:**
```
Row 1:  Company Financial Report Q4 2024
Row 2:  Prepared by Finance Team
Row 3:  [empty]
Row 4:  Revenue | COGS | Gross Profit
Row 5:  1000000 | 600000 | 400000
```

Headers in row 4, not row 1.

**Impact on Accuracy:**
- Header detection heuristics may fail if they assume row 1
- May miss concept names

**Mitigation Strategies:**
1. **Multi-row header detection:**
   - Check first N rows (N=5-10) for text-only rows followed by data
   - Look for data type changes (text → numbers)
2. **Pattern matching:**
   - Detect "title" patterns in first few rows
   - Skip decorative rows
3. **Bold/format detection:**
   - Headers often bold or formatted differently

**Expected Behavior:**
- Correctly identify row 4 as headers
- Extract concept names: Revenue, COGS, Gross Profit

---

### 1.3 Hierarchical/Merged Headers

**Scenario:**
```
         | Q1          | Q1         | Q2         | Q2
         | Budget      | Actual     | Budget     | Actual
---------|-------------|------------|------------|------------
Revenue  | 1000000     | 950000     | 1100000    | 1050000
```

Two-level headers (time period + actual/budget).

**Impact on Accuracy:**
- Need to understand hierarchical structure
- Concept is "Q1 Budget Revenue", not just "Revenue"
- Critical for comparative queries

**Mitigation Strategies:**
1. **Detect merged cells:** Spreadsheet APIs can detect merged cells
2. **Build header path:** Traverse upward to build full context
   - Cell B3 → "Q1 / Budget / Revenue"
3. **Store hierarchical context:** Keep header_path in concept metadata
4. **Query matching:** Match at any level of hierarchy
   - "Q1 revenue" should match all Q1 revenue concepts

**Expected Behavior:**
- Extract full concept: "Q1 Budget Revenue"
- Match queries like "Q1 revenue" or "budget revenue"

---

### 1.4 Transposed Layouts (Row-based vs Column-based)

**Scenario A (Column-based - typical):**
```
Metric         | Value
---------------|--------
Revenue        | 1M
Gross Margin   | 42%
```

**Scenario B (Row-based):**
```
Metric   | Revenue | Gross Margin | Net Margin
---------|---------|--------------|------------
Value    | 1M      | 42%          | 18%
```

**Impact on Accuracy:**
- Header detection must work for both orientations
- Need to determine if headers are in rows or columns

**Mitigation Strategies:**
1. **Detect orientation:**
   - If first column is text and first row is text, check which has more variation
   - Column headers: first row has unique text values, subsequent rows are data
   - Row headers: first column has unique text values, subsequent columns are data
2. **Try both:** Extract concepts assuming both orientations, keep higher confidence
3. **Format hints:** Bold formatting indicates headers

**Expected Behavior:**
- Correctly identify header orientation
- Extract concepts regardless of layout

---

### 1.5 Multiple Tables in One Sheet

**Scenario:**
```
Table 1: Revenue Breakdown (A1:C10)
[empty rows]
Table 2: Expense Breakdown (A15:C25)
[empty rows]
Table 3: Profitability Metrics (A30:C35)
```

**Impact on Accuracy:**
- Need to detect table boundaries
- Headers for one table shouldn't be applied to another
- Context for concepts should be table-specific

**Mitigation Strategies:**
1. **Detect table boundaries:**
   - Look for empty rows/columns
   - Look for formatting changes (borders, background color)
   - Detect contiguous data regions
2. **Segment sheet into tables:** Each table is a separate context
3. **Independent header detection:** Find headers for each table
4. **Table naming:** Extract table names from preceding rows or table titles

**Expected Behavior:**
- Segment into 3 tables
- Each concept knows which table it belongs to
- Search within appropriate context

---

## Category 2: Formula Complexity Challenges

### 2.1 Deeply Nested Formulas

**Scenario:**
```
=IF(SUM(FILTER(Table1[Amount],Table1[Category]="Sales"))>1000000,
   (SUM(FILTER(Table1[Amount],Table1[Category]="Sales"))-
    SUM(FILTER(Table2[Cost],Table2[Category]="Sales")))/
    SUM(FILTER(Table1[Amount],Table1[Category]="Sales")),
   0)
```

Complex conditional margin calculation.

**Impact on Accuracy:**
- Hard to extract semantic meaning
- Pattern matching may fail
- AST parsing is complex

**Mitigation Strategies:**
1. **Recursive AST analysis:** Parse into tree, analyze sub-expressions
2. **Identify core pattern:** Despite complexity, it's still a margin calculation: (Revenue-Cost)/Revenue
3. **Extract simplified version:** "(Sales Revenue - Sales Cost) / Sales Revenue if Revenue > 1M, else 0"
4. **Complexity scoring:** Assign complexity score, use in ranking
5. **LLM assistance:** Use LLM to interpret complex formulas

**Expected Behavior:**
- Extract pattern: "conditional margin calculation"
- Generate description: "Calculates sales margin if revenue exceeds $1M"
- Mark as high complexity
- Still match query "show margin calculations"

---

### 2.2 Array Formulas and Dynamic Arrays

**Scenario:**
```
=SORT(FILTER(Table[Revenue],Table[Region]="West"),1,-1)
```

Modern Excel array formula returning multiple values.

**Impact on Accuracy:**
- Returns array, not single value
- Need to understand array operations
- Pattern matching is different

**Mitigation Strategies:**
1. **Detect array formulas:** Check formula type in spreadsheet API
2. **Understand array operations:** Build patterns for FILTER, SORT, UNIQUE, etc.
3. **Conceptualize appropriately:** This is "Sorted West Region Revenue"
4. **Index spill range:** All cells in spill range are part of same concept

**Expected Behavior:**
- Recognize as filter + sort operation
- Extract concept: "West Region Revenue (sorted)"
- Match queries about "West region" or "sorted revenue"

---

### 2.3 Circular References

**Scenario:**
```
Cell A1: =B1+10
Cell B1: =A1+5
```

Circular dependency (usually an error, but sometimes intentional in iterative calculations).

**Impact on Accuracy:**
- Dependency graph has cycles
- Cannot fully resolve formula semantics
- May cause indexing to hang or error

**Mitigation Strategies:**
1. **Detect cycles:** Graph cycle detection algorithm
2. **Break cycles:** Ignore back-edges in dependency graph
3. **Mark as error:** Flag cells with circular references
4. **Don't index:** Skip circular reference cells or index with low confidence
5. **Warn user:** Surface circular references as potential issues

**Expected Behavior:**
- Detect circular reference
- Don't crash indexing
- Low confidence or skip indexing these cells

---

### 2.4 External References

**Scenario:**
```
='[OtherWorkbook.xlsx]Sheet1'!A5
```

Reference to another workbook file.

**Impact on Accuracy:**
- Cannot resolve external reference if file not available
- Semantic meaning is incomplete
- May be critical for understanding

**Mitigation Strategies:**
1. **Detect external references:** Parse formula for external file syntax
2. **Attempt to resolve:** If other file is available, load and resolve
3. **Store unresolved:** Keep external reference as text
4. **Extract what's possible:** Reference might give context (file name, sheet name)
5. **Mark as incomplete:** Flag concept as having external dependencies

**Expected Behavior:**
- Recognize external reference
- Extract file/sheet/cell info
- Mark as "references external data"
- Lower confidence due to incompleteness

---

### 2.5 User-Defined Functions (UDFs) / Macros

**Scenario:**
```
=CustomProfitCalc(A1, A2, 0.15)
```

Custom function defined in VBA/Apps Script.

**Impact on Accuracy:**
- Cannot understand function logic without executing/analyzing code
- Function name might be semantic clue
- Parameters might provide context

**Mitigation Strategies:**
1. **Extract function name:** "CustomProfitCalc" suggests profit calculation
2. **Analyze parameters:** If parameters are cell references, resolve them
3. **Pattern library:** Maintain mapping of known UDFs to semantic meanings
4. **User annotations:** Allow users to describe UDFs
5. **Execution (risky):** Execute UDF in sandbox to understand behavior (security risk!)

**Expected Behavior:**
- Extract function name: "CustomProfitCalc"
- Infer meaning: likely related to profit
- Match queries about "profit" with medium confidence
- Mark as "custom function"

---

## Category 3: Data Quality Issues

### 3.1 Inconsistent Naming

**Scenario:**
```
Sheet1: "Revenue"
Sheet2: "Total Revenue"
Sheet3: "Rev"
Sheet4: "Sales"
```

Same concept, different names across sheets.

**Impact on Accuracy:**
- Hard to recognize as same concept
- Cross-sheet matching fails
- Comparative queries miss results

**Mitigation Strategies:**
1. **Synonym dictionary:** Map "revenue", "sales", "rev" to same canonical concept
2. **Fuzzy matching:** Use string similarity (edit distance, Jaro-Winkler)
3. **Semantic similarity:** Embeddings will cluster similar names
4. **Value pattern matching:** If values are similar across sheets, likely same concept
5. **Structural matching:** Same position in similar sheets

**Expected Behavior:**
- Recognize all 4 as revenue-related
- Group together for comparative queries
- Prefer "Revenue" as canonical name
- Show alternatives in results

---

### 3.2 Typos and Spelling Errors

**Scenario:**
```
Header: "Grooss Profit Margn"
```

Typo in header.

**Impact on Accuracy:**
- Exact keyword match fails
- May not match "Gross Profit Margin" query

**Mitigation Strategies:**
1. **Spell check during indexing:** Correct common typos
2. **Fuzzy matching:** Allow edit distance of 1-2 characters
3. **Query spell correction:** Fix typos in user queries
4. **Embeddings help:** "Grooss Profit Margn" embedding will be close to "Gross Profit Margin"

**Expected Behavior:**
- Index as "Gross Profit Margin" (corrected)
- Note original spelling in metadata
- Still match queries for "gross profit margin"

---

### 3.3 Mixed Languages

**Scenario:**
```
Sheet1 (English): "Revenue", "Cost", "Profit"
Sheet2 (Spanish): "Ingresos", "Costo", "Ganancia"
```

Multilingual spreadsheet.

**Impact on Accuracy:**
- Synonym dictionaries won't match across languages
- Embeddings might not align across languages (depends on model)
- User query language may not match sheet language

**Mitigation Strategies:**
1. **Language detection:** Detect language per sheet/concept
2. **Translation layer:** Translate to common language (English) for indexing
3. **Multilingual embeddings:** Use models trained on multiple languages (e.g., multilingual BERT)
4. **Query translation:** Translate query to detected language
5. **Language preference:** User specifies preferred language

**Expected Behavior:**
- Detect Spanish in Sheet2
- Translate or use multilingual embeddings
- Match "revenue" query to "Ingresos"

---

### 3.4 Numeric Formatting Issues

**Scenario:**
```
Cell displays: "42%"
Actual value: 0.42
```

vs

```
Cell displays: "42"
Actual value: 42
Formula: =Revenue/100
```

Different representations of percentage.

**Impact on Accuracy:**
- Need to distinguish between 42% (0.42) and 42 (42)
- Formula semantics change interpretation
- Search for "percentage calculations" should match first, not second

**Mitigation Strategies:**
1. **Use cell format:** Check formatting to distinguish percentage vs number
2. **Analyze formula:** If dividing by 100, likely converting to percentage
3. **Value normalization:** Normalize percentages to 0-1 range
4. **Pattern detection:** Margin calculations typically result in 0-1, not 0-100

**Expected Behavior:**
- Correctly identify percentage vs number
- Formula pattern matching uses format info
- "percentage calculations" query matches formatted percentages

---

### 3.5 Empty or #ERROR Cells

**Scenario:**
```
Cell A1: #DIV/0!
Cell B1: #N/A
Cell C1: [empty]
```

Errors or empty values.

**Impact on Accuracy:**
- Should not be indexed as concepts
- But formula might still be meaningful
- Errors might indicate data issues

**Mitigation Strategies:**
1. **Detect errors:** Check cell error state
2. **Index formula, not value:** Formula might be valid, just current data causes error
3. **Mark as error state:** Concept metadata includes "current error"
4. **Filter from results:** Don't return error cells unless explicitly queried
5. **Surface to user:** "Found match but cell currently has error"

**Expected Behavior:**
- Index formula if it's meaningful
- Mark as error state
- Don't show in standard results
- Special error query: "show cells with errors"

---

## Category 4: Query Understanding Challenges

### 4.1 Ambiguous Queries

**Scenario:**
Query: "growth"

Could mean:
- Revenue growth
- Customer growth
- Profit growth
- YoY, MoM, QoQ growth
- Absolute or percentage growth

**Impact on Accuracy:**
- Multiple valid interpretations
- Showing wrong results frustrates users
- Low confidence in any single interpretation

**Mitigation Strategies:**
1. **Detect ambiguity:** If query matches multiple concepts in different domains
2. **Ask clarifying questions:**
   - "Did you mean: (A) Revenue growth, (B) Customer growth, or (C) Other?"
3. **Show grouped results:**
   - Group 1: Revenue Growth Metrics
   - Group 2: Customer Growth Metrics
   - Group 3: Profit Growth Metrics
4. **Use context:** If user previously searched "revenue", prioritize revenue growth
5. **Confidence scores:** Show all with appropriate confidence levels

**Expected Behavior:**
- Recognize ambiguity
- Either ask clarification OR show grouped results
- Don't confidently return one interpretation

---

### 4.2 Overly Broad Queries

**Scenario:**
Query: "show me everything"

**Impact on Accuracy:**
- Hundreds of potential results
- No clear relevance ranking
- User overwhelmed

**Mitigation Strategies:**
1. **Detect broad queries:** Pattern matching for "everything", "all", "show me"
2. **Suggest refinement:** "Please be more specific. Try: 'show all metrics' or 'show Q4 data'"
3. **Show summary:** Instead of all cells, show sheet/section overview
4. **Smart sampling:** Show diverse sample (one from each category/sheet)
5. **Filters:** Offer filters: "Filter by: KPIs, Formulas, Sheets"

**Expected Behavior:**
- Recognize as too broad
- Suggest refinement or show smart summary
- Don't dump 500 results

---

### 4.3 Negation Queries

**Scenario:**
Query: "metrics NOT related to revenue"

**Impact on Accuracy:**
- Need to handle negation logic
- Find all metrics, then filter out revenue-related
- Negation in NLU is challenging

**Mitigation Strategies:**
1. **Detect negation:** Parse for "not", "without", "excluding"
2. **Two-stage retrieval:**
   - Stage 1: Find all metrics
   - Stage 2: Find revenue-related metrics
   - Result: Stage 1 - Stage 2
3. **Semantic negation:** Embeddings don't naturally handle negation
4. **Explicit filtering:** Apply filter at ranking stage

**Expected Behavior:**
- Detect "NOT revenue"
- Return metrics unrelated to revenue
- Explain: "Showing metrics excluding revenue-related"

---

### 4.4 Context-Dependent Queries

**Scenario:**
Query 1: "show revenue"
→ Returns Q4 2024 Revenue

Query 2 (follow-up): "what about Q3?"
→ Should return Q3 2024 Revenue (context: "revenue")

**Impact on Accuracy:**
- Need conversation context
- "what about Q3" alone is ambiguous
- Session/conversation tracking required

**Mitigation Strategies:**
1. **Session tracking:** Maintain conversation context
2. **Resolve pronouns:** "it", "that", "these" refer to previous results
3. **Context expansion:** "Q3" → "Q3 Revenue" (from previous query)
4. **Time-limited context:** Context valid for N minutes or M queries
5. **Explicit context reset:** User can say "new topic"

**Expected Behavior:**
- Understand "Q3" refers to Q3 revenue
- Return Q3 Revenue
- Show context: "Showing Q3 Revenue (context from previous query)"

---

### 4.5 Domain-Specific Jargon

**Scenario:**
Query: "show me the T-12 EBITDA"

"T-12" = Trailing 12 months (industry jargon)

**Impact on Accuracy:**
- Generic NLU won't understand "T-12"
- Business dictionary might not have every jargon term
- Company-specific terms

**Mitigation Strategies:**
1. **Domain dictionary:** Curate finance, marketing, sales jargon
2. **Company-specific learning:** Learn from user feedback
3. **Acronym expansion:** T-12 → Trailing 12 months
4. **Context clues:** If spreadsheet has "Trailing 12 Months" header, match to "T-12"
5. **Ask user:** "What does T-12 mean in your context?"

**Expected Behavior:**
- Expand "T-12" to "Trailing 12 months"
- Search for "Trailing 12 months EBITDA"
- Or ask user: "Did you mean Trailing 12 months?"

---

## Category 5: Ranking and Relevance Challenges

### 5.1 Recency vs Importance Trade-off

**Scenario:**
- Old KPI Dashboard (1 year old) has "Gross Margin"
- Recent ad-hoc sheet (1 day old) also has "Gross Margin"

Query: "gross margin"

Which to prioritize?

**Impact on Accuracy:**
- Recency suggests new sheet
- Importance suggests KPI Dashboard
- User likely wants KPI Dashboard

**Mitigation Strategies:**
1. **Balanced scoring:** Don't overweight recency
2. **Importance signals:** Named ranges, dashboard sheets, formatted cells
3. **Sheet naming:** "Dashboard" or "KPI" in name → higher importance
4. **Usage patterns:** If available, prioritize frequently accessed
5. **User preference:** Allow user to specify preference

**Expected Behavior:**
- KPI Dashboard result ranks higher (importance > recency)
- Still show recent sheet as alternative
- Explain: "From KPI Dashboard (primary)" vs "From recent analysis (alternative)"

---

### 5.2 Exact Match vs Semantic Match

**Scenario:**
Query: "profit margin"

Results:
- "Profit Margin" (exact match, minor metric)
- "Gross Profit Margin" (semantic match, major KPI)
- "Net Profit Margin" (semantic match, major KPI)

Which to rank highest?

**Impact on Accuracy:**
- Exact match might be irrelevant
- Semantic matches might be more important

**Mitigation Strategies:**
1. **Don't over-boost exact match:** Balance with importance
2. **Concept specificity:** "Profit Margin" is ambiguous, specific margins (Gross, Net) are clearer
3. **Importance wins:** If semantic match has high importance, rank above exact match
4. **Show all:** User can choose which they meant

**Expected Behavior:**
- Gross Profit Margin ranked #1 (semantic + importance)
- Net Profit Margin ranked #2 (semantic + importance)
- Profit Margin ranked #3 (exact but less important)

---

### 5.3 Formula Complexity Preference

**Scenario:**
Query: "show revenue calculations"

Results:
- Simple: =SUM(Monthly_Revenue)
- Complex: =SUMIFS(Data[Revenue], Data[Date], ">="&StartDate, Data[Date], "<="&EndDate, Data[Region], Selected_Region)

Which to prefer?

**Impact on Accuracy:**
- User might want either
- "Calculations" suggests complexity
- But simple might be the main metric

**Mitigation Strategies:**
1. **Query analysis:** "calculations" → user wants formulas
2. **Complexity scoring:** Include both, but highlight complexity
3. **Explain difference:** "Simple sum" vs "Filtered sum with criteria"
4. **User feedback:** Learn if users prefer simple or complex

**Expected Behavior:**
- Show both
- Rank complex higher for "calculations" query
- Rank simple higher for just "revenue" query
- Explain complexity in results

---

## Category 6: Multi-Sheet Nuances

### 6.1 Same Sheet Names Across Workbooks

**Scenario:**
- Workbook A has "Q4 Financials" sheet
- Workbook B has "Q4 Financials" sheet

Query: "Q4 revenue"

Which sheet?

**Impact on Accuracy:**
- Ambiguity if multiple workbooks in scope
- Need to distinguish

**Mitigation Strategies:**
1. **Scope to current workbook:** Default to active workbook
2. **Workbook prefixes:** Include workbook name in results
3. **User specification:** "In workbook A, show Q4 revenue"
4. **Recency:** Prioritize recently opened workbook

**Expected Behavior:**
- Scope to current workbook by default
- Show results from other workbooks as "Other matches"
- Display: "[Workbook A] Q4 Financials!D5"

---

### 6.2 Hidden Sheets

**Scenario:**
Sheet "Calculations" is hidden (contains intermediate formulas).

Query: "revenue calculation"

Should hidden sheet results be shown?

**Impact on Accuracy:**
- Hidden sheets often contain internals, not user-facing
- But might have relevant content
- User might not want to see hidden sheets

**Mitigation Strategies:**
1. **Default: exclude hidden:** Don't index hidden sheets by default
2. **Option to include:** User setting "Include hidden sheets"
3. **Lower ranking:** If included, rank hidden sheet results lower
4. **Indicate hidden:** Mark results from hidden sheets: "From hidden sheet 'Calculations'"

**Expected Behavior:**
- Don't show hidden sheet results by default
- Option to include if needed
- Indicate source when shown

---

### 6.3 Protected/Locked Cells

**Scenario:**
Cells are locked/protected (read-only).

Impact on search?

**Impact on Accuracy:**
- Protection doesn't affect search, only editing
- But might indicate importance (protected = important)

**Mitigation Strategies:**
1. **Index normally:** Protection status doesn't affect semantics
2. **Use as signal:** Protected cells might be KPIs (importance boost)
3. **Indicate in results:** Show lock icon for protected cells

**Expected Behavior:**
- Index protected cells normally
- Slight importance boost if protected
- Indicate protection status in UI

---

## Category 7: Performance and Scale Challenges

### 7.1 Very Large Spreadsheets

**Scenario:**
Spreadsheet with 100K+ cells, 50+ sheets.

**Impact on Accuracy:**
- Indexing time is high
- Query latency increases
- Memory constraints

**Mitigation Strategies:**
1. **Selective indexing:** Only index meaningful cells (formulas, headers, named ranges)
2. **Batch processing:** Index in background
3. **Incremental updates:** Only re-index changed cells
4. **Sampling:** For very large sheets, sample data regions
5. **Distributed indexing:** Use multiple workers for parallel processing

**Expected Behavior:**
- Index completes in reasonable time (<5 min for 100K cells)
- Query latency stays <2s
- Progressive indexing: partial results while indexing completes

---

### 7.2 Rapid Updates

**Scenario:**
User is actively editing, cells changing every few seconds.

**Impact on Accuracy:**
- Index constantly out of date
- Indexing overhead impacts performance
- Querying during indexing might miss recent changes

**Mitigation Strategies:**
1. **Debouncing:** Wait for edit pause before re-indexing
2. **Priority queue:** Prioritize recent edits
3. **Eventual consistency:** Accept slight lag (1-2 seconds)
4. **Lock-free reads:** Query can run during indexing
5. **Incremental updates:** Only re-index changed concepts

**Expected Behavior:**
- Updates reflected within 2-3 seconds
- Query results might lag slightly
- Show "as of [timestamp]" for index freshness

---

### 7.3 Extremely Long Formulas

**Scenario:**
Formula with 10,000+ characters (auto-generated).

**Impact on Accuracy:**
- Parsing time is high
- Embedding input truncation
- Hard to extract semantics

**Mitigation Strategies:**
1. **Truncation:** Truncate formula text for embedding (keep first 500 chars)
2. **Timeout:** Set parsing timeout, skip if too long
3. **Simplified representation:** Extract key functions, ignore details
4. **Mark as complex:** Flag as "very complex formula"
5. **Store full formula:** Keep in DB, but use simplified version for search

**Expected Behavior:**
- Parse successfully (with timeout)
- Extract simplified semantics
- Mark as very complex
- Lower confidence

---

## Category 8: Security and Privacy

### 8.1 Sensitive Data in Formulas

**Scenario:**
Formula contains hardcoded credentials (bad practice but happens):
```
=ImportData("https://user:password@api.com/data")
```

**Impact on Accuracy:**
- Security risk if formulas exposed in search results
- Need to sanitize

**Mitigation Strategies:**
1. **Pattern detection:** Detect patterns like "password", "apikey" in formulas
2. **Redaction:** Redact sensitive parts before indexing
3. **Warnings:** Warn user about sensitive data in spreadsheet
4. **Access control:** Respect spreadsheet permissions in search results

**Expected Behavior:**
- Detect sensitive data
- Redact from search index
- Warn user: "Spreadsheet contains potential credentials"

---

### 8.2 PII (Personally Identifiable Information)

**Scenario:**
Spreadsheet contains customer names, emails, phone numbers.

Query: "show customer data"

**Impact on Accuracy:**
- PII should be handled carefully
- May need to comply with GDPR, CCPA, etc.

**Mitigation Strategies:**
1. **PII detection:** Detect columns with emails, phone numbers, SSNs
2. **Exclusion:** Don't index PII values, only schema
3. **Access control:** Only show to authorized users
4. **Anonymization:** Replace PII with placeholders in index

**Expected Behavior:**
- Index schema ("Customer Email column") not values
- Apply access controls
- Comply with privacy regulations

---

## Category 9: User Behavior Patterns

### 9.1 Exploratory Queries

**Scenario:**
User doesn't know what they're looking for:
"What metrics are in this spreadsheet?"

**Impact on Accuracy:**
- Not a specific search
- Need to provide overview, not specific results

**Mitigation Strategies:**
1. **Detect exploratory intent:** "what", "overview", "show me"
2. **Provide summary:** List all concept categories
3. **Faceted navigation:** "You have: 15 profitability metrics, 10 growth metrics, ..."
4. **Suggest queries:** "Try: 'show profitability metrics' or 'find revenue data'"

**Expected Behavior:**
- Recognize exploratory query
- Provide overview/summary
- Suggest specific queries

---

### 9.2 Iterative Refinement

**Scenario:**
Query 1: "revenue"
→ Too many results

Query 2: "Q4 revenue"
→ Better

Query 3: "Q4 revenue by region"
→ Perfect

**Impact on Accuracy:**
- Users refine queries based on results
- System should learn from refinements

**Mitigation Strategies:**
1. **Query suggestion:** After first query, suggest refinements
2. **Filters:** Offer quick filters "Filter by: Sheet, Time Period, Category"
3. **Learn patterns:** If user refines A → B, suggest B directly next time
4. **Session context:** Use previous query to inform next

**Expected Behavior:**
- After "revenue" query, suggest: "Did you mean Q4 revenue, Budget revenue, ...?"
- Remember user's refinement pattern

---

## Summary: Accuracy Impact Analysis

### High Impact on Accuracy (Must Handle)
1. ✅ No clear headers → Use formula context, format clues
2. ✅ Multiple tables in sheet → Segment and detect boundaries
3. ✅ Inconsistent naming → Synonym dictionaries, semantic similarity
4. ✅ Ambiguous queries → Clarification or grouped results
5. ✅ Cross-sheet matching → Relationship detection

### Medium Impact (Should Handle)
1. ⚠ Complex formulas → Recursive analysis, LLM assistance
2. ⚠ Typos → Spell correction, fuzzy matching
3. ⚠ Hidden sheets → Option to include/exclude
4. ⚠ Recency vs importance → Balanced scoring
5. ⚠ Large spreadsheets → Selective indexing, optimization

### Low Impact (Nice to Have)
1. ℹ️ User-defined functions → Best-effort interpretation
2. ℹ️ External references → Mark as incomplete
3. ℹ️ Mixed languages → Multilingual models
4. ℹ️ Exploratory queries → Provide summaries
5. ℹ️ Context-dependent queries → Session tracking

---

## Testing Strategy

For each edge case:
1. **Create test case:** Build sample spreadsheet with edge case
2. **Define expected behavior:** What should happen?
3. **Measure accuracy:** Does system handle it correctly?
4. **Iterate:** Improve handling based on failures
5. **Regression test:** Ensure fixes don't break other cases

---

## Failure Mode Detection

Implement monitoring to detect:
1. **Low confidence results:** Many results below threshold
2. **No results:** Query returns empty
3. **High latency:** Query takes >5 seconds
4. **Parsing errors:** Formula parsing failures
5. **Index staleness:** Index not updating

Alert and investigate when failures occur frequently.
