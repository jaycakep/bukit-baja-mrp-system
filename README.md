# 🏭 Bukit Baja Anugrah — Manufacturing Resource Planning (MRP) System

### System Design Case Study | Built from Scratch · 2013–2016

> **Role:** Co-Founder & CTO, CV BIOS
> **Client:** Bukit Baja Anugrah — Steel Pipe Manufacturer
> **Period:** 2013 – 2016
> **Stack:** PHP 5.x · MySQL 5.5 · Bootstrap · jQuery · XAMPP
> **Database:** `mrp_baja` · 70+ Tables
> **Users:** 15 concurrent users · 3 shifts · 24-hour operations

---

## 🔥 The Mess — Why This System Had to Be Built

Bukit Baja Anugrah is a steel pipe manufacturer operating three shifts a day, around the clock. Before this system existed, the entire production planning and scheduling operation ran on Excel spreadsheets — manually managed by the PPIC team.

The most damaging bottleneck was production scheduling. Every time a customer order came in, the PPIC team had to manually cross-reference raw material availability, machine capacity, crew assignments, and delivery deadlines across multiple spreadsheets. The process took **three to four days** to complete — and during those days, the production floor was either idle or running on estimates.

The slitting calculation problem was even more painful. Before cutting a mother coil into smaller slit coils, the PPIC team had to manually calculate combinations of cut widths on paper and Excel — trying permutation after permutation until they found a combination that minimized waste. This process alone took **three to four hours per coil** and was still prone to human error, meaning material waste that directly eroded the company's margins.

In a 24-hour, three-shift manufacturing operation, these delays were not inefficiencies — they were competitive liabilities.

---

## 🔍 The Diagnosis — What Needed to Be Solved

After a full operational assessment, the core problems fell into three categories:

**Planning & Scheduling Gap:** There was no system connecting customer orders to production schedules. The PPIC team was manually bridging the gap between sales, raw material inventory, machine capacity, and crew availability — in Excel, across multiple files, with no single source of truth.

**Material Optimization Gap:** Cutting mother coils without a combinatorial optimizer meant every slitting run was a guessing game. The goal was to minimize waste below 1% of coil width — but without a calculation engine, this was nearly impossible to achieve consistently.

**Operational Visibility Gap:** Management had no real-time view of production progress, QC results, machine downtime, or inventory movement across the three shifts. Decisions were made based on morning reports that were already hours out of date.

---

## 🏗️ The Architecture — How It Was Solved

The system was built on a **PHP + MySQL + XAMPP** stack deployed on the manufacturer's existing on-premise Windows Server — keeping infrastructure costs at zero while covering all operational domains: order management, BOM, production scheduling, slitting, QC, inventory, packing, shipping, and executive reporting.

### Deployment Topology

```
[ 15 Users across 3 Shifts — Admin, PPIC, Operator, QC, Gudang, Shipping, Manager ]
                    ↓ (Internal LAN — 192.168.x.x)
              [ Apache (XAMPP) ]
                    ↓
         [ PHP Application Layer ]
         api.php (Front Controller)
         common_*.php (CRUD Engine)
         hitungsemuav2.php (Slitter Calculator)
                    ↓
         [ MySQL 5.5 — mrp_baja ]
              (70+ Tables)
                    ↓
         [ File System + Network Printer ]
         (PDFs, Excel exports, Surat Jalan)
```

### The Order-to-Production Flow

Every customer order triggers a chain: order entry → Bill of Materials input → production scheduling → crew and machine assignment → operator job execution → QC recording → inventory update → packing → shipping document generation. The system manages this entire chain in one unified platform, with each role seeing only what they need.

### Multi-Role Access Control

The system implements a **6-permission RBAC** model scoped per menu item:

| Permission | Code        |
| ---------- | ----------- |
| Read       | `r`       |
| Create     | `c`       |
| Update     | `u`       |
| Delete     | `d`       |
| Print      | `p`       |
| Approve    | `approve` |

Eight distinct roles — Admin, PPIC, Operator, QC Inspector, Warehouse, Packing, Shipping, Manager — each with precisely scoped permissions across 36 use cases.

### Database Conventions

The schema follows strict naming conventions across 70+ tables:

- `mt_` prefix for master tables (customers, crew, machines, materials)
- `dt_` prefix for transaction tables (orders, production, inventory movement)
- `sys_` / `master_` prefix for system configuration tables
- Every table carries a `del` soft-delete flag — no hard deletes in production
- Every transaction table carries full audit columns: `create_user`, `create_date`, `edit_user`, `edit_date`

---

## ⭐ The Flagship Feature: Slitter Cutting Optimization Engine

This is the most technically significant component of the system — and the one with the most direct business impact.

### The Problem

A mother coil arrives with a fixed width (e.g., 1,250mm) and must be cut into multiple slit coils of varying widths to fulfill customer orders. The goal is to find the combination of cut widths that:

1. Uses all mandatory (primer) widths the customer requires
2. Fills the coil as completely as possible
3. Leaves waste below 1% of total coil width

Before this engine existed, a PPIC operator would spend **3–4 hours** manually trying combinations on paper and Excel — and still frequently miss the optimal cut, leaving excess material waste that directly impacted production cost.

### The Solution: Recursive Combinatorial Search

`hitungsemuav2.php` implements a **depth-first recursive combinatorial search** that explores all valid cutting combinations and returns only those meeting the waste threshold.

```
INPUT
─────
Coil Width  : 1,250 mm
Coil Weight : 5,000 Kg
Slit Sizes  : [75, 90.5, 140, 185, 112, ...] (up to 10 widths)
Primer (✓)  : [140, 185] ← must appear in every valid result

ALGORITHM (hitungsemuav2.php)
─────────────────────────────
1. Separate PRIMER widths from SECONDARY widths
2. Sort PRIMER descending
3. For each PRIMER combination:
   - Calculate max_qty = floor(coil_width / size)
   - Loop qty from max → 1
   - Calculate remaining width after primer placement
   - If remainder < 1% → store as valid combination
   - If not → recurse: cari_kombinasi() with secondary widths
4. Recursion: try each SECONDARY width with decreasing qty
   - Accept only if remainder < 1% AND >= 4mm
5. Deduplicate all results
6. Calculate weight per slit = (slit_width / coil_width) × coil_weight

OUTPUT (example)
────────────────
Sisa    : 3.00 mm (0.24% — well under 1% threshold)

┌────────────┬─────┬──────────────┬────────────┐
│ Ukuran     │ Qty │ Total Width  │ Weight(Kg) │
├────────────┼─────┼──────────────┼────────────┤
│ 185        │  4  │ 740 mm       │ 2,960.00   │
│ 140        │  3  │ 420 mm       │ 1,680.00   │
│ 90.5       │  1  │ 90.5 mm      │   362.00   │
└────────────┴─────┴──────────────┴────────────┘
```

**Algorithm complexity:** O(n! × m) worst case, where n = number of secondary sizes and m = max qty per size. The 4mm floor and 1% waste ceiling prune the majority of branches early, keeping the recursion tractable within the 10-input constraint.

The result: what previously took **3–4 hours of manual calculation** now completes in **seconds**, with mathematically guaranteed waste minimization below 1%.

---

## ⚠️ Edge Cases — Where Manufacturing Gets Hard

### The Concurrent Shift Handover Problem

With three shifts running 24 hours and 15 concurrent users, the most critical integrity risk was two operators from overlapping shifts modifying the same production job or inventory record simultaneously.

The system handles this through:

- Soft-delete pattern (`del` flag) preventing accidental overwrites
- Full audit trail on every record — `edit_user` and `edit_date` on every UPDATE
- Job assignment at the schedule level (`dt_jprod`) — each job is assigned to a specific crew before execution begins, eliminating ambiguity about ownership

### The Accumulator Monitoring Problem

The pipe production process uses an accumulator — a buffer that must be monitored continuously. If the accumulator state is not recorded accurately across shift changes, production continuity breaks down. The system provides a dedicated accumulator monitoring module that operators update in real time, giving the incoming shift accurate state data before they take over.

### The Slitter Waste Threshold Problem

The recursive engine enforces a hard floor of 4mm minimum remainder and a 1% waste ceiling. This prevents the algorithm from returning technically valid but operationally useless combinations — for example, a combination that uses 1,249mm of a 1,250mm coil with a 1mm sliver that cannot be processed further. Every result the engine returns is actionable.

---

## 📊 The Impact — What Changed

| Before                                                | After                                                  |
| ----------------------------------------------------- | ------------------------------------------------------ |
| Production scheduling: 3–4 days manual in Excel      | Production scheduling: under 30 minutes                |
| Slitter calculation: 3–4 hours per coil, manual      | Slitter calculation: seconds, mathematically optimized |
| Waste rate: inconsistent, human-estimated             | Waste rate: below 1% — guaranteed by algorithm        |
| No real-time production visibility across shifts      | Live job tracking across all 3 shifts, 15 users        |
| No inventory movement audit trail                     | Full mutation history with user and timestamp          |
| Manager decisions based on stale morning reports      | Executive dashboard with real-time production KPIs     |
| 8 operational domains managed in separate Excel files | Single unified system — 70+ normalized tables         |

The scheduling improvement alone — from 3–4 days to under 30 minutes — meant the production floor could respond to customer orders in near-real-time instead of operating on multi-day lag. In a competitive manufacturing environment, that is not a productivity gain. It is a structural advantage.

---

## 💡 Key Architectural Decisions

**Why a recursive combinatorial engine instead of a greedy algorithm?**
A greedy approach (always pick the largest fitting width first) is fast but does not guarantee waste minimization when primer constraints are present. The recursive exhaustive search ensures every valid combination is evaluated — critical when material cost makes even 1% waste significant at scale.

**Why PHP + XAMPP on-premise instead of a cloud deployment?**
The manufacturer operated in an environment with no reliable internet connectivity on the production floor. An on-premise intranet deployment was not a budget constraint — it was the correct architecture for the operational environment. Cloud dependency would have introduced single points of failure in a 24-hour production operation.

**Why soft deletes across all 70+ tables?**
In a manufacturing context, hard-deleting a production record or inventory movement creates unrecoverable audit gaps. Every deletion in this system is a flag, not a destruction. This ensures the full history of every order, every job, and every inventory mutation is always reconstructable.

**Why the data-driven form engine (same pattern as KOPKAR RSI)?**
With 36 use cases across 8 roles and 70+ tables, hard-coding each module's form and list view would have been operationally unsustainable. The same `form_attr` / `form_entry_std` / `Common_CUQuery` pattern from the cooperative system was applied here — configuration-driven CRUD that allowed new modules to be added without writing new PHP files.

---

## 📐 System Design Diagrams

For the complete system architecture including Use Case Diagram (36 use cases, 8 actors), Sequence Diagrams (Order-to-Production, Slitting, Inventory, Shipping, QC), Domain Model, Entity-Relationship Diagram (70+ tables), Data Flow Diagrams, Activity Diagrams, and Deployment Diagram — see:

📄 **[DESIGN_SYSTEM.md](./DESIGN_SYSTEM.md)**

---

*This case study is part of my system design portfolio. The client is an active steel pipe manufacturer. All diagrams reflect the actual system architecture as designed and built between 2013 and 2016 under CV BIOS.*
