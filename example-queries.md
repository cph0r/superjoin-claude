# Example Queries & Test Cases

This document provides comprehensive test queries across different categories to validate the semantic search system. Each example includes expected results and accuracy considerations.

---

## Sample Spreadsheet Context

To ground these examples, assume a financial reporting spreadsheet with:

**Sheets:**
- "Q4 2024 Financials" (actuals)
- "Budget 2024"
- "Q3 2024 Financials" (historical)
- "Variance Analysis"
- "KPI Dashboard"

**Key Metrics:**
- Revenue, COGS, Gross Profit, Operating Expenses
- Gross Profit Margin, Net Profit Margin, EBITDA Margin
- YoY Growth, QoQ Growth
- Customer Acquisition Cost (CAC), Customer Lifetime Value (CLV)

---

## Query Category 1: Conceptual Queries

Finding concepts by business meaning rather than exact keywords.

### Query 1.1: "find profitability metrics"

**Intent:** Conceptual search for profitability-related KPIs

**Expected Results:**
1. **Gross Profit Margin** (Q4 2024 Financials!D15)
   - Formula: =(Revenue-COGS)/Revenue
   - Value: 42%
   - Confidence: High
   - Match reason: Profitability metric, margin calculation

2. **Net Profit Margin** (Q4 2024 Financials!D20)
   - Formula: =Net_Income/Revenue
   - Value: 18%
   - Confidence: High
   - Match reason: Profitability metric, margin calculation

3. **EBITDA Margin** (KPI Dashboard!C8)
   - Formula: =EBITDA/Revenue
   - Value: 25%
   - Confidence: High
   - Match reason: Profitability metric, margin calculation

4. **Operating Profit** (Q4 2024 Financials!D18)
   - Formula: =Gross_Profit-Operating_Expenses
   - Value: $250,000
   - Confidence: Medium
   - Match reason: Related to profitability

5. **ROI** (KPI Dashboard!C12)
   - Formula: =(Gain-Cost)/Cost
   - Value: 35%
   - Confidence: Medium
   - Match reason: Efficiency metric related to profitability

**Accuracy Test:**
- Should NOT return: Revenue, Sales Volume, Customer Count (not profitability metrics)
- Precision target: 80%+ (4/5 results are true profitability metrics)
- Recall target: 90%+ (finds most major profitability metrics)

---

### Query 1.2: "show customer metrics"

**Intent:** Find metrics related to customers

**Expected Results:**
1. **Customer Acquisition Cost** (KPI Dashboard!B15)
   - Formula: =Marketing_Spend/New_Customers
   - Value: $125
   - Confidence: High

2. **Customer Lifetime Value** (KPI Dashboard!B16)
   - Formula: =Avg_Revenue_Per_Customer*Avg_Lifetime
   - Value: $2,400
   - Confidence: High

3. **Customer Churn Rate** (KPI Dashboard!B18)
   - Formula: =Lost_Customers/Total_Customers
   - Value: 5%
   - Confidence: High

4. **Total Customers** (Q4 2024 Financials!E25)
   - Value: 1,250
   - Confidence: Medium

5. **New Customers** (Q4 2024 Financials!E26)
   - Value: 180
   - Confidence: Medium

**Accuracy Test:**
- Should NOT return: Revenue (not directly a customer metric), Employee metrics
- Test synonym handling: "customer" vs "client" metrics

---

### Query 1.3: "where are my efficiency ratios"

**Intent:** Find operational efficiency metrics

**Expected Results:**
1. **Operating Margin** (KPI Dashboard!C10)
2. **Asset Turnover Ratio** (if exists)
3. **Inventory Turnover** (if exists)
4. **ROI** (KPI Dashboard!C12)
5. **ROAS** (Return on Ad Spend) (if exists)

**Accuracy Test:**
- Tests understanding of "efficiency" as a business concept
- Tests synonym: "ratio" vs "metric" vs "KPI"

---

## Query Category 2: Functional Queries

Finding cells based on formula characteristics.

### Query 2.1: "show all percentage calculations"

**Intent:** Find cells with percentage-type formulas

**Expected Results:**
1. **Gross Profit Margin** (Q4 2024 Financials!D15)
   - Formula: =(D10-D11)/D10
   - Pattern: (A-B)/A percentage

2. **YoY Growth Rate** (Variance Analysis!F5)
   - Formula: =(Q4_2024-Q4_2023)/Q4_2023
   - Pattern: Growth percentage

3. **Market Share** (KPI Dashboard!E8)
   - Formula: =Our_Revenue/Total_Market_Revenue
   - Pattern: Ratio/percentage

4. **Customer Churn Rate** (KPI Dashboard!B18)
   - Formula: =Lost_Customers/Total_Customers
   - Pattern: Ratio/percentage

5. **Completion Rate** (if exists)

**Accuracy Test:**
- Should match: A/B formulas, (A-B)/A formulas, cells formatted as %
- Should NOT match: SUM formulas, simple numbers
- Formula pattern recognition accuracy target: 85%+

---

### Query 2.2: "find lookup functions"

**Intent:** Find cells using VLOOKUP, INDEX/MATCH, XLOOKUP

**Expected Results:**
1. **Customer Name Lookup** (somewhere in sheets)
   - Formula: =VLOOKUP(A2,CustomerTable,2,FALSE)
   - Confidence: High

2. **Product Price Lookup** (somewhere in sheets)
   - Formula: =INDEX(PriceTable,MATCH(ProductID,ProductList,0))
   - Confidence: High

3. Any other cells using XLOOKUP, HLOOKUP

**Accuracy Test:**
- Should recognize all lookup function types
- Should NOT return: simple cell references (=A5)

---

### Query 2.3: "show conditional sums"

**Intent:** Find SUMIF, SUMIFS, COUNTIF type formulas

**Expected Results:**
1. **Sales by Region** (if exists)
   - Formula: =SUMIF(Region,"West",Sales)

2. **High-Value Customers** (if exists)
   - Formula: =COUNTIF(CustomerValue,">1000")

3. Any AVERAGEIF, COUNTIFS formulas

**Accuracy Test:**
- Pattern matching for conditional aggregations
- Distinguish from regular SUM/COUNT

---

## Query Category 3: Comparative Queries

Finding related concepts across sheets or time periods.

### Query 3.1: "budget vs actual revenue"

**Intent:** Compare budgeted and actual revenue

**Expected Results (Grouped):**

**Group 1: Q4 Revenue**
- **Budget:** Budget 2024!D5 = $1,000,000
- **Actual:** Q4 2024 Financials!D5 = $950,000
- **Variance:** Variance Analysis!D5 = -$50,000 (-5%)

**Explanation:**
"Found Q4 Revenue across Budget, Actuals, and Variance sheets. Budget was $1M, actual came in at $950K, resulting in a -5% variance."

**Accuracy Test:**
- Must identify correct sheets (Budget, Actuals, Variance)
- Must pair corresponding concepts across sheets
- Must show relationships (variance calculation)
- Cross-sheet matching accuracy target: 80%+

---

### Query 3.2: "compare Q3 and Q4 gross margin"

**Intent:** Compare same metric across time periods

**Expected Results (Grouped):**

**Group 1: Gross Profit Margin Comparison**
- **Q3 2024:** Q3 2024 Financials!D15 = 40%
- **Q4 2024:** Q4 2024 Financials!D15 = 42%
- **Change:** +2 percentage points (+5% improvement)

**Explanation:**
"Gross Profit Margin improved from 40% in Q3 to 42% in Q4, a 2 percentage point increase."

**Accuracy Test:**
- Identify correct time periods from sheet names
- Match same concept across periods
- Recognize "gross margin" as synonym for "gross profit margin"

---

### Query 3.3: "variance analysis for all KPIs"

**Intent:** Find all variance calculations

**Expected Results:**
1. **Revenue Variance** (Variance Analysis!D5)
2. **Gross Profit Variance** (Variance Analysis!D10)
3. **Operating Expense Variance** (Variance Analysis!D15)
4. **Net Income Variance** (Variance Analysis!D20)

All showing formula pattern: =Actual_Value - Budget_Value

**Accuracy Test:**
- Should find all variance formulas
- Should recognize variance = actual - budget pattern
- Should group related variances

---

## Query Category 4: Navigational Queries

Direct lookup of specific cells or concepts.

### Query 4.1: "where is the revenue cell"

**Intent:** Find specific concept location

**Expected Results:**
1. **Q4 2024 Revenue** (Q4 2024 Financials!D5)
   - Primary result, most recent
   - Confidence: High

2. **Budget Revenue** (Budget 2024!D5)
   - Alternative result
   - Confidence: Medium

3. **Q3 2024 Revenue** (Q3 2024 Financials!D5)
   - Historical result
   - Confidence: Medium

**Explanation:**
"Found Revenue in multiple sheets. Primary result is Q4 2024 Actuals (most recent)."

**Accuracy Test:**
- Should prioritize most recent/relevant sheet
- Should show all instances as alternatives

---

### Query 4.2: "show me the KPI dashboard"

**Intent:** Navigate to specific sheet

**Expected Results:**
- Return all key metrics from "KPI Dashboard" sheet
- Show sheet overview with main concepts

**Accuracy Test:**
- Sheet name recognition
- Return comprehensive view of sheet contents

---

## Query Category 5: Complex/Ambiguous Queries

Testing edge cases and disambiguation.

### Query 5.1: "growth"

**Ambiguous:** Could mean revenue growth, customer growth, margin expansion, etc.

**Expected Behavior:**
- **Option A:** Ask clarifying question
  - "Did you mean: Revenue growth, Customer growth, or Margin growth?"

- **Option B:** Show grouped results by growth type
  - Group 1: Revenue Growth metrics
  - Group 2: Customer Growth metrics
  - Group 3: Other growth metrics

**Accuracy Test:**
- System should recognize ambiguity
- Should not return low-confidence single result

---

### Query 5.2: "marketing ROI"

**Intent:** Marketing-specific ROI, not general ROI

**Expected Results:**
1. **ROAS** (Return on Ad Spend) (KPI Dashboard!E15)
   - Most relevant for marketing
   - Confidence: High

2. **Marketing ROI** (if explicitly named)
   - Confidence: High

3. **CAC** (Customer Acquisition Cost)
   - Related to marketing efficiency
   - Confidence: Medium

Should NOT return: General ROI, Financial ROI

**Accuracy Test:**
- Context-aware search ("marketing" context)
- Should differentiate from general ROI

---

### Query 5.3: "margins"

**Ambiguous:** Which margin? Gross, net, operating, EBITDA?

**Expected Behavior:**
Return all margin types, grouped:

1. **Gross Profit Margin** (42%)
2. **Net Profit Margin** (18%)
3. **Operating Margin** (23%)
4. **EBITDA Margin** (25%)

All with "Margin" in the concept name.

**Accuracy Test:**
- Should return all margin types
- Should NOT return non-margin metrics

---

## Query Category 6: Synonym and Alias Handling

Testing synonym recognition.

### Query 6.1: "earnings"

**Synonyms:** profit, income, revenue (context-dependent)

**Expected Results:**
1. **Net Income/Net Profit** (primary meaning)
2. **EBITDA** (earnings before...)
3. **Operating Income**
4. **Revenue** (sometimes called "earnings" in sales context)

**Accuracy Test:**
- Should handle multiple interpretations
- Context matters: financial statements vs sales reports

---

### Query 6.2: "sales" vs "revenue"

Should return same results (synonyms in most contexts)

**Accuracy Test:**
- Synonym dictionary coverage
- Context-specific handling (B2B vs retail)

---

### Query 6.3: "GM" or "NM"

**Abbreviations:**
- GM → Gross Margin
- NM → Net Margin

**Expected Results:**
Should expand abbreviations and return corresponding metrics.

**Accuracy Test:**
- Abbreviation dictionary
- Domain-specific acronyms (GM could also be "General Manager" in HR context)

---

## Query Category 7: Multi-Constraint Queries

Combining multiple requirements.

### Query 7.1: "percentage calculations in the budget sheet"

**Constraints:**
1. Formula type: percentage calculations
2. Location: Budget sheet only

**Expected Results:**
Only percentage formulas from "Budget 2024" sheet

**Accuracy Test:**
- Correctly apply both filters
- Should NOT return percentages from other sheets

---

### Query 7.2: "Q4 profitability metrics with variance"

**Constraints:**
1. Time: Q4
2. Type: Profitability metrics
3. Additional: With variance analysis

**Expected Results:**
Profitability metrics from Q4 with their variances:
1. Gross Margin (Q4 Actual + Variance)
2. Net Margin (Q4 Actual + Variance)
3. etc.

**Accuracy Test:**
- Complex query parsing
- Multiple constraint satisfaction

---

## Performance Test Queries

### Speed Tests

1. **Simple query:** "revenue" → Target: <1 second
2. **Complex query:** "budget vs actual for all profitability metrics" → Target: <2 seconds
3. **Broad query:** "show all metrics" → Target: <2 seconds
4. **Cross-sheet:** "variance analysis" → Target: <1.5 seconds

---

## Stress Test Queries

### Edge Cases

1. **Empty query:** "" → Return error or suggest example queries
2. **Very long query:** 200+ word question → Should extract key concepts
3. **Nonsense query:** "asdfasdf" → No results, suggest alternatives
4. **Typos:** "revenu" → Should correct to "revenue"
5. **Mixed case:** "ReVeNuE" → Should normalize
6. **Special characters:** "revenue!" → Should handle gracefully

---

## Test Evaluation Metrics

For each query, measure:

1. **Precision@5:** Of top 5 results, how many are relevant?
   - Target: >80%

2. **Recall@10:** Of all relevant results, how many in top 10?
   - Target: >70%

3. **MRR (Mean Reciprocal Rank):** Average rank of first relevant result
   - Target: >0.75 (first relevant result typically in top 2-3)

4. **Response Time:** End-to-end query latency
   - Target: <2 seconds for 90th percentile

5. **Confidence Accuracy:** Are high-confidence results actually relevant?
   - Target: >90% accuracy for "high confidence" results

---

## Test Spreadsheet Setup

To run these test cases, create sample spreadsheet with:

### Q4 2024 Financials Sheet
```
A         | B              | C       | D
----------|----------------|---------|----------
          | Q4 2024        |         |
----------|----------------|---------|----------
Financials|                |         |
----------|----------------|---------|----------
Revenue   |                |         | 1,000,000
COGS      |                |         |   600,000
Gross Profit|              |         |   400,000
...
Gross Margin % |           |         | =D5-D6)/D5  (40%)
Operating Expenses |        |         |   200,000
Operating Income |          |         |   200,000
Operating Margin % |        |         | =(D5-D10)/D5 (20%)
...
Net Income |                |         |   150,000
Net Margin % |              |         | =D13/D5  (15%)
```

### Budget 2024 Sheet
```
Similar structure with budget values
```

### Variance Analysis Sheet
```
Formulas: =Actuals!Dx - Budget!Dx
```

### KPI Dashboard Sheet
```
Key metrics pulled from other sheets:
- CAC, CLV, Churn Rate
- ROI, ROAS
- Growth rates
- Margins
```

---

## Success Criteria Summary

A successful semantic search system should:

1. ✅ Handle all conceptual queries with >80% precision
2. ✅ Correctly identify formula patterns for functional queries
3. ✅ Match corresponding concepts across sheets for comparative queries
4. ✅ Provide clear explanations for why results matched
5. ✅ Handle synonyms and abbreviations accurately
6. ✅ Disambiguate ambiguous queries appropriately
7. ✅ Respond within 2 seconds for 90% of queries
8. ✅ Gracefully handle edge cases and errors

---

## Testing Process

1. **Create test spreadsheet** with known ground truth
2. **Run all queries** in this document
3. **Manually annotate** expected results
4. **Measure accuracy** using Precision@K, Recall@K, MRR
5. **Analyze failures** and iterate on approach
6. **A/B test** different approaches (Hybrid vs Rules-based)
7. **Optimize weights** in hybrid scoring function
8. **Validate** with real user queries and feedback
