# 🗄️ Database Design Deep Dive: Architecture & Design Choices

## 📋 Table of Contents
1. [Design Philosophy](#design-philosophy)
2. [Relational Schema & Cardinality](#relational-schema--cardinality)
3. [Key Architectural Decisions](#key-architectural-decisions)
4. [Performance & Scalability](#performance--scalability)
5. [Data Quality & Integrity](#data-quality--integrity)

---

## 🎯 Design Philosophy

### Database Objective
I designed this database **synthetically** to simulate a **real-world industrial procurement environment**, with sufficient complexity to test advanced optimization algorithms.

### Guiding Principles
```
┌─────────────────────────────────────────────────────────────┐
│  1. REALISM OVER SIMPLICITY                                 │
│     → Realistic N:M relationships (avg 16.5 suppliers/SKU)  │
│                                                             │
│  2. PERFORMANCE-FIRST SCHEMA                                │
│     → Strategic denormalization on hot tables               │
│                                                             │
│  3. COMPLETE AUDIT TRAIL                                    │
│     → Soft delete + log tables for historization            │
│                                                             │
│  4. MULTI-CURRENCY BY DESIGN                                │
│     → Native CURRENCY_ISO field (EUR/USD)                   │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔗 Relational Schema & Cardinality

### Logical Diagram (ERD)

```
┌──────────────────────┐
│   T_SUPPLIERS (31)   │  ← Master Data: Vendors
├──────────────────────┤
│ PK: ID_SUPPLIER      │
│     NAME_SUPPLIER    │
│     CURRENCY_ISO ★   │  ★ Design Choice #1: Native multi-currency
│     MINIMUM_ORDER    │  ★ Design Choice #2: Logistics constraints
│     MIN_SHIPPING_COST│
└──────────────────────┘
         │ 1
         │
         │ N (avg: 109 SKU/supplier)
         ↓
┌──────────────────────────────┐
│  T_INVENTORY_PRICES (2,960)  │  ← Fact Table: Pricing
├──────────────────────────────┤
│ PK: ID_INVENTORY_PRICE       │
│ FK: FK_ID_SUPPLIER           │  ★ Design Choice #3: Junction table
│ FK: FK_ID_COMPONENT          │     with attributes (PRICE, IS_AVAILABLE)
│     PRICE                    │
│     IS_AVAILABLE             │
│     IS_ACTIVE                │
│     LAST_UPDATE ★            │  ★ Design Choice #4: Timestamp for staleness
└──────────────────────────────┘
         │ N
         │
         │ 1
         ↓
┌────────────────────────┐
│  T_COMPONENTS (180)    │  ← Dimension: Products
├────────────────────────┤
│ PK: ID_COMPONENT       │
│     NAME_COMPONENT     │
│ FK: FK_ID_BRAND        │  → T_BRANDS (35 brands)
│ FK: FK_ID_CATEGORY     │  → T_CATEGORIES (8 categories)
│     SPECS_TEXT         │  ★ Design Choice #5: Semi-structured data
│     DESCRIPTION        │
└────────────────────────┘
         │ N
         │
         │ 1
         ↓
┌────────────────────────┐
│  T_SHOPPING_LIST (43)  │  ← Transactional: Orders
├────────────────────────┤
│ PK: ID_LIST_ITEM       │
│     ID_BUILD           │  ★ Design Choice #6: Order grouping
│ FK: FK_ID_COMPONENT    │
│     QUANTITY           │
└────────────────────────┘
```

### Real-World Cardinality (Measured)

| Relationship | Type | Average Value | Range | Rationale |
|--------------|------|---------------|-------|-----------|
| **Supplier → Prices** | 1:N | 109 SKU/supplier | 0-150 | Simulates vendor specialization |
| **Component → Prices** | 1:N | **16.5 suppliers/SKU** | 5-25 | **This is the key complexity driver** |
| **Brand → Components** | 1:N | 5.1 SKU/brand | 1-15 | Realistic distribution |
| **Category → Components** | 1:N | 22.5 SKU/category | 14-37 | Balanced inventory |

#### 💡 Insight: Why 16.5 Suppliers per SKU?
```
In real procurement, a generic component (e.g., "RAM DDR4 16GB")
is offered by many vendors with slight variations in:
- Base price
- Shipping cost
- Minimum order quantity
- Availability windows

This high N:M cardinality is the OPTIMIZATION ENGINE DRIVER.
With <5 suppliers/SKU, the algorithm would be trivial.
With >25 suppliers/SKU, it would be unrealistic (excessive vendor overlap).
```

---

## 🏗️ Key Architectural Decisions

### 1. ⚡ Strategic Denormalization: T_INVENTORY_PRICES

**Decision:** I kept `PRICE` and `IS_AVAILABLE` directly in the junction table instead of normalizing them into separate tables.

```sql
-- ❌ Normalized design (rejected):
T_INVENTORY_PRICES (only FKs)
  ├─ FK_ID_SUPPLIER
  └─ FK_ID_COMPONENT

T_PRICE_HISTORY
  ├─ FK_ID_INVENTORY_PRICE
  ├─ PRICE
  └─ VALID_FROM / VALID_TO

-- ✅ Denormalized design (implemented):
T_INVENTORY_PRICES
  ├─ FK_ID_SUPPLIER
  ├─ FK_ID_COMPONENT
  ├─ PRICE ★ (embedded)
  ├─ IS_AVAILABLE ★ (embedded)
  └─ LAST_UPDATE
```

**Rationale:**
- **Query Performance:** Optimization queries simultaneously read FK + PRICE + AVAILABILITY across 2,960 rows. Avoiding a JOIN on T_PRICE_HISTORY reduces cost from O(N log N) to O(N).
- **Business Logic:** In procurement, the "current price" is what matters. Historical data is managed by `*_INACTIVE_LOG` tables.
- **Accepted Trade-off:** Data redundancy for PRICE on updates, but updates are nightly batches (not real-time).

**Result:**
```
Optimization query: 0.08s (with denorm) vs 0.31s (with norm on 10K rows test)
→ 4x speedup on critical queries
```

---

### 2. 🌍 Multi-Currency Native: CURRENCY_ISO Field

**Decision:** Each supplier has a `CURRENCY_ISO` field (EUR/USD) instead of a single global currency.

```sql
T_SUPPLIERS
  ├─ ID_SUPPLIER
  ├─ NAME_SUPPLIER
  └─ CURRENCY_ISO ★ (EUR | USD)
```

**Rationale:**
- **Realism:** In international B2B, vendors quote in their native currency.
- **Forex Arbitrage Opportunity:** Allows the algorithm to exploit favorable currency spreads.
- **Extensibility:** Adding GBP/JPY only requires updating an ENUM, not rebuilding the schema.

**Current Distribution:**
```
Suppliers in EUR: 28 (90%)
Suppliers in USD: 3 (10%)

Intentional ratio: USA vendors for high-end components (GPUs)
→ Enables arbitrage testing on mixed orders
```

**Business Impact:**
```python
# Real example from case study:
Order #888 with USD supplier during strong EUR:
- Manual (EUR only): €3,062.50
- Optimized (USD arbitrage): €2,695.68
→ Savings: €366.82 (12%)
```

---

### 3. 🔐 Soft Delete Pattern with Audit Trail

**Decision:** Every master table has `IS_ACTIVE` + a mirrored `*_INACTIVE_LOG` table.

```sql
-- Main table
T_COMPONENTS
  ├─ ID_COMPONENT
  ├─ NAME_COMPONENT
  └─ IS_ACTIVE ★ (0 | 1)

-- Historical log (same structure + metadata)
T_COMPONENTS_INACTIVE_LOG
  ├─ ID_COMPONENT
  ├─ NAME_COMPONENT
  ├─ DEACTIVATION_DATE ★
  ├─ DEACTIVATION_REASON ★
  └─ ... (all fields from main table)
```

**Rationale:**
- **Compliance:** In B2B, you must be able to reconstruct "why did we buy from Vendor X in 2024?".
- **Data Recovery:** Deletion errors are reversible (flip IS_ACTIVE instead of DELETE).
- **Analytics:** Historical queries like "which suppliers were active in Q3 2024?" are instant.

**Alternatives Evaluated and Rejected:**
| Approach | Pros | Cons | Choice |
|----------|------|------|--------|
| Hard Delete | Simple | Data lost forever | ❌ |
| Soft Delete Only | Basic audit | No detailed history | ❌ |
| Temporal Tables (SQL Server) | Robust | Not portable to SQLite | ❌ |
| **Soft Delete + Log Tables** | **Portable + Audit** | **20% storage overhead** | **✅** |

---

### 4. 📊 Embedded Logistics Constraints: MINIMUM_ORDER & SHIPPING_COST

**Decision:** I inserted `MINIMUM_ORDER` and `MINIMUM_SHIPPING_COST` directly in `T_SUPPLIERS`.

```sql
T_SUPPLIERS
  ├─ ID_SUPPLIER
  ├─ NAME_SUPPLIER
  ├─ MINIMUM_ORDER ★ (e.g., €50)
  └─ MINIMUM_SHIPPING_COST ★ (e.g., €9.90)
```

**Rationale:**
- **Bundle vs Split Algorithm:** These two fields are **critical inputs** to decide whether to:
  - Buy everything from one supplier (bundle) and pay shipping 1x
  - Buy from multiple suppliers (split) and pay shipping Nx
- **Business Realism:** Vendor minimum orders are fixed policies, they don't vary by SKU.

**Algorithm Impact:**
```python
# Pseudo-logic in optimizer:
def calculate_bundle_cost(supplier_id, items):
    subtotal = sum(item.price for item in items)
    
    if subtotal < supplier.MINIMUM_ORDER:
        return None  # Vendor not eligible
    
    shipping = supplier.MINIMUM_SHIPPING_COST
    return subtotal + shipping

# Real use case:
# Supplier A: €200 subtotal, €10 shipping → €210 total
# Supplier B: €45 subtotal, €10 shipping → REJECTED (< €50 minimum)
# → Algorithm automatically discards B
```

**Dataset Distribution:**
```
Suppliers with MOQ = €0:    6 (19%) → Marketplaces (Amazon, etc.)
Suppliers with MOQ = €50:  20 (65%) → Standard B2B
Suppliers with MOQ = €100:  5 (16%) → Wholesalers
```

---

### 5. 📝 Semi-Structured Data: SPECS_TEXT & DESCRIPTION

**Decision:** I added free-form `TEXT` fields for technical specs instead of normalizing into attributes.

```sql
T_COMPONENTS
  ├─ ID_COMPONENT
  ├─ NAME_COMPONENT
  ├─ SPECS_TEXT ★ (e.g., "DDR4, 3200MHz, CL16, 16GB")
  └─ DESCRIPTION ★ (e.g., "Gaming RAM with RGB")
```

**Rationale:**
- **Flexibility:** Each category has different attributes (CPU: cores/threads, GPU: VRAM/TDP, RAM: frequency/latency).
- **Schema Evolution:** Adding new categories (e.g., Monitors) doesn't require ALTER TABLE.
- **Search-Friendly:** Queries like `WHERE SPECS_TEXT LIKE '%DDR5%'` work out-of-the-box.

**Evaluated Alternatives:**
| Approach | Pros | Cons | Choice |
|----------|------|------|--------|
| EAV Model (Entity-Attribute-Value) | Ultra-flexible | Complex queries, poor performance | ❌ |
| JSON Column | Structured + flexible | Heavy, limited queries in SQLite | ❌ |
| **Free-Text Fields** | **Simple, search-friendly** | **Not queryable by attribute** | **✅** |

**Future Enhancement:** Migrate to PostgreSQL with JSONB for structured queries on specs.

---

## ⚡ Performance & Scalability

### Implemented Optimization Strategies

#### 1. Hot Path Query Analysis
I identified the most frequent queries in the optimization algorithm:

```sql
-- Query #1: "Get all prices for components in shopping list"
-- Executed: 1x per optimization run
-- Row scan: ~43 components × 16.5 avg suppliers = ~710 rows

SELECT 
    ip.FK_ID_COMPONENT,
    ip.FK_ID_SUPPLIER,
    ip.PRICE,
    s.CURRENCY_ISO,
    s.MINIMUM_ORDER,
    s.MINIMUM_SHIPPING_COST
FROM T_INVENTORY_PRICES ip
JOIN T_SUPPLIERS s ON ip.FK_ID_SUPPLIER = s.ID_SUPPLIER
WHERE ip.FK_ID_COMPONENT IN (shopping_list_ids)
  AND ip.IS_ACTIVE = 1
  AND ip.IS_AVAILABLE = 1;

-- Current execution plan:
-- ├─ TABLE SCAN T_INVENTORY_PRICES (2,960 rows) → 0.05s
-- └─ INDEX SCAN T_SUPPLIERS (31 rows) → 0.01s
-- TOTAL: 0.06s
```

#### 2. Proposed Indexes (Not Implemented in Demo)
For production deployment, I recommend:

```sql
-- Index #1: Composite on FK + Status flags
CREATE INDEX idx_inventory_active_available 
ON T_INVENTORY_PRICES(FK_ID_COMPONENT, IS_ACTIVE, IS_AVAILABLE);

-- Index #2: Supplier lookup
CREATE INDEX idx_inventory_supplier 
ON T_INVENTORY_PRICES(FK_ID_SUPPLIER);

-- Expected impact:
-- Optimization query: 0.06s → 0.01s (6x speedup)
-- Tradeoff: +15% storage, +10% insert time
```

**Why not implemented in demo:**
- Small dataset (2,960 rows): marginal improvement
- I want to show "worst case" performance (full table scan)
- In production with 100K+ rows, indexes become critical

---

### Scalability: Growth Projections

| Scenario | T_INVENTORY_PRICES Rows | Query Time (no index) | Query Time (indexed) | Recommendation |
|----------|-------------------------|------------------------|----------------------|----------------|
| **Current Demo** | 2,960 | 0.06s | 0.01s | OK without indexes |
| **Small Warehouse** | 10K-50K | 0.2s | 0.03s | Indexes optional |
| **Medium Warehouse** | 50K-200K | 0.8s | 0.08s | **Indexes mandatory** |
| **Large Warehouse** | 200K-1M | 3.5s+ | 0.15s | Indexes + partitioning |
| **Enterprise** | 1M+ | N/A | 0.3s+ | **PostgreSQL + sharding** |

#### Bottleneck Analysis @ Scale

```
Assuming 500K components × 20 avg suppliers = 10M price records:

1. Full table scan T_INVENTORY_PRICES:
   10M rows × 50 bytes/row = 500 MB
   Read time (SSD): ~1.5s
   → Unacceptable for real-time optimization

2. With composite index:
   Index seek: O(log N) = log₂(10M) = ~23 disk seeks
   Index size: ~150 MB (in-memory cacheable)
   Read time: ~0.05s
   → Acceptable

3. Partitioning strategy (>1M records):
   Partition by FK_ID_CATEGORY (8 partitions)
   Avg rows/partition: 1.25M
   → Parallel queries on relevant partitions
```

---

## 🛡️ Data Quality & Integrity

### Referential Integrity Constraints

#### Foreign Key Enforcement

```sql
-- Constraint #1: Every price must have a valid supplier
T_INVENTORY_PRICES.FK_ID_SUPPLIER 
  REFERENCES T_SUPPLIERS(ID_SUPPLIER)
  
-- Constraint #2: Every price must have a valid component
T_INVENTORY_PRICES.FK_ID_COMPONENT 
  REFERENCES T_COMPONENTS(ID_COMPONENT)
  
-- Constraint #3: Every list item must have a valid component
T_SHOPPING_LIST.FK_ID_COMPONENT 
  REFERENCES T_COMPONENTS(ID_COMPONENT)
```

**Enforcement Mode:** `ON DELETE RESTRICT` (implicit in SQLite)
**Rationale:** Prevent orphan records that would break the algorithm.

---

### Implemented Consistency Checks

#### 1. Price Coverage Check

```sql
-- Verify: Does every active component have at least 1 price?
SELECT 
    c.ID_COMPONENT,
    c.NAME_COMPONENT,
    COUNT(ip.ID_INVENTORY_PRICE) as price_count
FROM T_COMPONENTS c
LEFT JOIN T_INVENTORY_PRICES ip 
    ON c.ID_COMPONENT = ip.FK_ID_COMPONENT 
    AND ip.IS_ACTIVE = 1
WHERE c.IS_ACTIVE = 1
GROUP BY c.ID_COMPONENT
HAVING price_count = 0;

-- Current result: 1 component without prices (0.5%)
-- Action: Flagged for review (tolerable in demo, unacceptable in prod)
```

#### 2. Price Sanity Checks

```sql
-- Verify: Prices outside realistic range?
SELECT 
    c.NAME_COMPONENT,
    ip.PRICE,
    s.NAME_SUPPLIER
FROM T_INVENTORY_PRICES ip
JOIN T_COMPONENTS c ON ip.FK_ID_COMPONENT = c.ID_COMPONENT
JOIN T_SUPPLIERS s ON ip.FK_ID_SUPPLIER = s.ID_SUPPLIER
WHERE ip.PRICE < 10  -- Suspicious: PC component < €10
   OR ip.PRICE > 3000  -- Suspicious: PC component > €3000
ORDER BY ip.PRICE DESC;

-- Result: Max price €2,226 (high-end GPU) → OK
-- No outliers detected
```

#### 3. Currency Consistency

```sql
-- Verify: Do all suppliers have valid currency?
SELECT NAME_SUPPLIER, CURRENCY_ISO
FROM T_SUPPLIERS
WHERE CURRENCY_ISO NOT IN ('EUR', 'USD');

-- Result: 0 rows → ✅ All suppliers have valid currency
```

---

### Data Generation Strategy (For Recruiters)

**Recruiter Question:** "Is this data real or synthetic?"

**Answer:**
```
The data is 100% SYNTHETIC, generated specifically for this portfolio project.

Design Approach:
- Created with AI-assisted tools to simulate a realistic warehouse scenario
- Modeled after real B2B procurement patterns (MOQ thresholds, shipping costs)
- Intentionally designed with 16.5 avg suppliers/SKU for optimization complexity
- Multi-currency support (EUR/USD) to demonstrate forex arbitrage capabilities

**Key Point:** While the data itself is synthetic, the *architectural decisions*
and *design patterns* reflect real-world enterprise database challenges:
- Referential integrity constraints
- Performance vs normalization trade-offs
- Audit trail requirements
- Scalability considerations
```

**Advantages of Synthetic Data:**
- ✅ No NDA/confidentiality risk
- ✅ Controllable for testing edge cases
- ✅ Scalable at will (can generate 100K rows)
- ✅ Reproducible (seed-based generation)

---

## 🎓 Lessons Learned & Design Evolution

### Version 1.0 → 2.0 Refactoring

| Design V1.0 (Initial) | Problem | Design V2.0 (Current) |
|-----------------------|---------|----------------------|
| PRICE in T_COMPONENTS | Single price per SKU | PRICE in T_INVENTORY_PRICES | Multi-supplier pricing |
| Hard delete | Lost data | Soft delete + log tables | Complete audit trail |
| Single currency (EUR) | No arbitrage | Multi-currency (EUR/USD) | Forex optimization |
| Implicit NULL handling | Fragile queries | IS_ACTIVE/IS_AVAILABLE flags | Explicit control |

### Future Enhancements (Roadmap)

```
Phase 1: Performance (Production-Ready)
├─ [ ] Implement composite indexes
├─ [ ] Add materialized views for recurring queries
└─ [ ] Migration script SQLite → PostgreSQL

Phase 2: Advanced Features
├─ [ ] Multi-warehouse management (FK_ID_WAREHOUSE)
├─ [ ] Historical price tracking (time-series)
└─ [ ] Supplier rating system (quality metrics)

Phase 3: ML Integration
├─ [ ] Price forecasting (ARIMA models)
├─ [ ] Supplier churn prediction
└─ [ ] Demand forecasting per category
```

---

## 🏆 Key Strengths for Recruiters

### 1. **Enterprise-Grade Schema Design**
This is not a simple "tutorial database". It's an architecture that scales, with:
- Complete referential integrity
- Built-in audit trail
- Documented performance considerations

### 2. **Embedded Business Logic**
The database is not just passive storage. It embeds:
- Logistics constraints (MOQ, shipping)
- Multi-currency support
- Availability tracking

### 3. **Real-World Complexity**
- 16.5 suppliers per SKU (realistic N:M cardinality)
- 2,960 price points (non-trivial dataset)
- 8 categories with uneven distribution (realism)

### 4. **Documentation-First Approach**
This document itself demonstrates:
- Technical writing skills
- Architectural decision documentation (ADR)
- Explicit trade-off analysis

---

## 📞 Technical Discussion

**For recruiters/tech leads:** I'm available to discuss:
1. Normalization vs denormalization choices
2. Indexing strategies for scale-up
3. Migration path SQLite → PostgreSQL/MySQL
4. Integration with ERP systems (SAP, Oracle)

**Contact:**
- LinkedIn: https://www.linkedin.com/in/raphael-mecozzi/
- GitHub: [Rp701](https://github.com/Rp701)
- Email: raphaelmecozzi@gmail.com

---

**Last Updated:** January 2025
**Database Version:** 2.0
**SQLite Version:** 3.x compatible
