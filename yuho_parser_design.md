# Document Intelligence Parser: 連結貸借対照表 (Consolidated Balance Sheet)
## Technical Deconstruction & System Design

---

## 1. Structural & Layout Breakdown

### 1.1 Visual & Syntactic Hierarchy

The document exhibits a **5-level indented hierarchy** encoded purely through horizontal whitespace and font weight — not cell borders or XML nesting. There are no separating grid lines between rows.

```
Level 0 │ Document Header   → (訂正後) 3. 連結財務諸表及び主な注記
Level 1 │ Statement Title   → （1）連結貸借対照表
Level 2 │ Section Header    → 資産の部  (Assets Section)
Level 3 │ Category Header   → 流動資産 / 固定資産 / 有形固定資産 / …
Level 4 │ Line Item         → 現金及び預金, 建物及び構築物, …
Level 4a│  ├─ Gross Value   → 建物及び構築物                2,121,360 / 1,880,082
Level 4b│  ├─ Contra-Asset  → 　減価償却累計額               △1,923,585 / △1,640,253
Level 4c│  └─ Net Value     → 　建物及び構築物（純額）          197,775 / 239,828
Level 5 │ Subtotal          → 流動資産合計, 有形固定資産合計, …
Level 5a│ Roll-up Total     → 固定資産合計, 資産合計
```

**Column layout (3 data columns):**

| Column | Content | Format |
|---|---|---|
| C0 | Line-item label (Japanese) | Left-aligned, indent-encoded depth |
| C1 | 前連結会計年度 (FY ending 2025-03-31) | Right-aligned, comma-thousands |
| C2 | 当連結会計年度 (FY ending 2026-03-31) | Right-aligned, comma-thousands |

**Special value tokens:**
- `△` prefix = negative/contra (e.g., `△1,923,585`)  — NOT a standard minus sign
- `—` (em-dash) = nil/zero balance (e.g., 建設仮勘定 C1, 長期貸付金 C1)
- Blank cell = not applicable for that period
- Underline on some values = footnote reference or calculated subtotal

---

### 1.2 Failure Modes for Naive Extractors

#### FM-1: △ (Triangle) as Negation vs. OCR Artifact

The document uses the full-width triangle character `△` (U+25B3) as the sign indicator for contra-asset deductions and negative balances. Standard OCR engines (Tesseract, PaddleOCR) frequently:
- Drop the character entirely, producing `1,923,585` instead of `△1,923,585`
- Confuse it with `A` or `Δ` (Greek delta, U+0394)
- Produce `▲` (filled triangle, U+25BC) which in other J-GAAP documents means a *different* kind of negative (net loss vs. deduction)

**Evidence in document:** `△1,923,585`, `△3,634,165`, `△147,599`, `△2,979`, `△0`

**Impact:** Sign flip on contra-asset lines causes total row arithmetic to fail catastrophically. A naive sum of the PP&E block would compute `2,121,360 + 1,923,585 = 4,044,945` instead of the correct `197,775`.

---

#### FM-2: Indentation as Sole Hierarchy Signal (No Borders)

This is a borderless table. The only structural signal is **pixel-level x-coordinate offset** of label text. Boundary-box extractors (e.g., pdfplumber's `extract_table()`, Camelot in lattice mode) will:
- Fail entirely (lattice mode: no lines to detect)
- Merge all rows into a flat list (stream mode: no column separators)
- Misassign label depth — e.g., treating `減価償却累計額` (a contra-asset indented one extra level) as a sibling of `建物及び構築物` rather than its child

The three sub-items of each PP&E component (`gross → contra → net`) are visually differentiated *only* by relative x-position of the label, not by any color or weight distinction.

---

#### FM-3: "—" (Em-dash) as Null vs. Zero vs. Not-Applicable

The document uses an em-dash `—` in at least two semantically distinct situations:
1. **Zero balance that existed**: `建設仮勘定` C1 = `—` (no balance in prior year, but 4,553 in current)
2. **Structurally absent**: `長期貸付金` C1 = `—` (line item didn't exist in scope for prior year)

Naive extractors either:
- Map `—` → `0` (wrong: conflates zero with not-applicable)
- Drop the cell, causing column-count mismatches that misalign subsequent rows

This is especially dangerous for arithmetic validation: a zero-valued cell *should* participate in subtotal checks; a not-applicable cell *must not*.

---

#### FM-4: Merged Semantic Rows vs. Physical Rows (Contra-Asset Triples)

Each depreciable asset in the PP&E block occupies **three physical rows** that together represent one *logical accounting triple*:

```
Row A:  建物及び構築物           2,121,360    1,880,082   ← Gross (取得価額)
Row B:    減価償却累計額        △1,923,585   △1,640,253   ← Accumulated Depreciation
Row C:    建物及び構築物（純額）    197,775      239,828    ← Net Book Value = A + B
```

Standard table extractors produce three independent rows. A downstream model consuming flat rows will:
- Treat the net-value row as an independent asset line (double-counting)
- Fail to reconstruct the `gross + accum_dep = net` constraint
- Lose the pairing linkage between `Row B` and `Row A` (which asset the depreciation belongs to)

This pattern repeats for: `機械装置及び運搬具`, `リース資産 (有形)`, `その他 (有形)` — four separate triples on this page alone.

---

#### FM-5 (Bonus): Multi-Tier Subtotal Folding

The hierarchy contains subtotals that roll up into higher subtotals:

```
建物及び構築物（純額）
機械装置及び運搬具（純額）
土地
リース資産（純額）
建設仮勘定
その他（純額）
─────────────────────
有形固定資産合計       1,429,094   1,352,610   ← Level-3 subtotal

有形固定資産合計
無形固定資産合計
投資その他の資産合計
─────────────────────
固定資産合計          1,670,426   1,757,841   ← Level-2 subtotal

固定資産合計 + 流動資産合計
─────────────────────
資産合計              7,534,110   5,135,249   ← Top-level total
```

Naive extractors produce a flat list. Without hierarchy metadata, it's impossible to determine *which* subtotal validates *which* group, and rolling arithmetic checks become ambiguous.

---

## 2. Formal Output Representation

### 2.1 JSON Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "FinancialStatementTable",
  "description": "Hierarchical + tabular representation of a J-GAAP balance sheet",
  "type": "object",
  "required": ["metadata", "columns", "nodes"],
  "properties": {

    "metadata": {
      "type": "object",
      "required": ["document_type", "entity", "correction_flag", "unit", "currency", "periods"],
      "properties": {
        "document_type":    { "type": "string", "enum": ["BS", "PL", "CF", "SE"] },
        "statement_label":  { "type": "string", "description": "e.g. 連結貸借対照表" },
        "entity":           { "type": "string", "description": "Filing entity name" },
        "correction_flag":  { "type": "boolean", "description": "True if 訂正後 header present" },
        "unit":             { "type": "string", "enum": ["円", "千円", "百万円"] },
        "currency":         { "type": "string", "default": "JPY" },
        "consolidation":    { "type": "string", "enum": ["consolidated", "standalone"] },
        "periods": {
          "type": "array",
          "items": {
            "type": "object",
            "required": ["column_id", "label_ja", "period_end_date", "period_type"],
            "properties": {
              "column_id":      { "type": "string" },
              "label_ja":       { "type": "string" },
              "period_end_date":{ "type": "string", "format": "date" },
              "period_type":    { "type": "string", "enum": ["annual", "semi-annual", "quarterly"] }
            }
          }
        }
      }
    },

    "columns": {
      "type": "array",
      "description": "Ordered list of value columns (mirrors metadata.periods)",
      "items": { "type": "string" }
    },

    "nodes": {
      "type": "array",
      "description": "Tree nodes in document order; parent-child via node_id/parent_id",
      "items": { "$ref": "#/definitions/FinancialNode" }
    }
  },

  "definitions": {

    "FinancialNode": {
      "type": "object",
      "required": ["node_id", "depth", "node_type", "label_ja"],
      "properties": {
        "node_id":      { "type": "string", "description": "Unique stable ID, e.g. assets.ppe.buildings" },
        "parent_id":    { "type": ["string", "null"] },
        "depth":        { "type": "integer", "minimum": 0 },
        "node_type": {
          "type": "string",
          "enum": [
            "section_header",     // 資産の部
            "category_header",    // 流動資産, 有形固定資産
            "line_item_gross",    // 建物及び構築物 (取得価額)
            "line_item_contra",   // 減価償却累計額
            "line_item_net",      // 建物及び構築物（純額）
            "line_item_simple",   // 現金及び預金 (no depreciation)
            "subtotal",           // 流動資産合計, 有形固定資産合計
            "total"               // 固定資産合計, 資産合計
          ]
        },
        "label_ja":     { "type": "string" },
        "label_en":     { "type": ["string", "null"] },
        "xbrl_element": { "type": ["string", "null"], "description": "EDINET XBRL tag if mappable" },
        "values": {
          "type": "object",
          "description": "Keyed by column_id from metadata.periods",
          "additionalProperties": { "$ref": "#/definitions/CellValue" }
        },
        "contra_ref":  {
          "type": ["string", "null"],
          "description": "For line_item_contra: node_id of the gross line this contra belongs to"
        },
        "gross_ref":   {
          "type": ["string", "null"],
          "description": "For line_item_net: node_id of the gross line this net derives from"
        },
        "children":    {
          "type": "array",
          "items": { "type": "string" },
          "description": "Ordered list of child node_ids"
        },
        "validation_rules": {
          "type": "array",
          "items": { "$ref": "#/definitions/ValidationRule" }
        }
      }
    },

    "CellValue": {
      "type": "object",
      "required": ["raw_text", "value_type"],
      "properties": {
        "raw_text":     { "type": "string", "description": "Exact text from PDF, e.g. △1,923,585" },
        "numeric":      { "type": ["number", "null"], "description": "Signed numeric, e.g. -1923585" },
        "value_type":   { "type": "string", "enum": ["numeric", "nil", "dash", "not_applicable"] },
        "sign_source":  { "type": "string", "enum": ["explicit_triangle", "explicit_minus", "positive", "none"] },
        "confidence":   { "type": "number", "minimum": 0, "maximum": 1 },
        "bbox":         {
          "type": ["object", "null"],
          "description": "Bounding box in PDF points for traceability",
          "properties": {
            "page": { "type": "integer" },
            "x0": { "type": "number" }, "y0": { "type": "number" },
            "x1": { "type": "number" }, "y1": { "type": "number" }
          }
        }
      }
    },

    "ValidationRule": {
      "type": "object",
      "required": ["rule_id", "rule_type", "formula"],
      "properties": {
        "rule_id":    { "type": "string" },
        "rule_type":  { "type": "string", "enum": ["sum_check", "contra_net_check", "sign_check"] },
        "formula":    { "type": "string", "description": "Human-readable, e.g. 'gross + contra = net'" },
        "operands":   { "type": "array", "items": { "type": "string" } },
        "tolerance":  { "type": "number", "default": 0, "description": "Rounding tolerance in base unit" }
      }
    }
  }
}
```

---

### 2.2 Instantiation: 有形固定資産 / Property, Plant & Equipment

This is the most complex section — three contra-asset triples plus simple lines.

```json
{
  "metadata": {
    "document_type": "BS",
    "statement_label": "連結貸借対照表",
    "entity": "(filing entity)",
    "correction_flag": true,
    "unit": "千円",
    "currency": "JPY",
    "consolidation": "consolidated",
    "periods": [
      {
        "column_id": "fy2025",
        "label_ja": "前連結会計年度（2025年3月31日）",
        "period_end_date": "2025-03-31",
        "period_type": "annual"
      },
      {
        "column_id": "fy2026",
        "label_ja": "当連結会計年度（2026年3月31日）",
        "period_end_date": "2026-03-31",
        "period_type": "annual"
      }
    ]
  },
  "columns": ["fy2025", "fy2026"],
  "nodes": [
    {
      "node_id": "assets.fixed.ppe",
      "parent_id": "assets.fixed",
      "depth": 3,
      "node_type": "category_header",
      "label_ja": "有形固定資産",
      "label_en": "Property, Plant and Equipment",
      "xbrl_element": "jpcrp_cor:PropertyPlantAndEquipmentNetMember",
      "values": {},
      "children": [
        "assets.fixed.ppe.buildings.gross",
        "assets.fixed.ppe.buildings.contra",
        "assets.fixed.ppe.buildings.net",
        "assets.fixed.ppe.machinery.gross",
        "assets.fixed.ppe.machinery.contra",
        "assets.fixed.ppe.machinery.net",
        "assets.fixed.ppe.land",
        "assets.fixed.ppe.lease.gross",
        "assets.fixed.ppe.lease.contra",
        "assets.fixed.ppe.lease.net",
        "assets.fixed.ppe.construction_in_progress",
        "assets.fixed.ppe.other.gross",
        "assets.fixed.ppe.other.contra",
        "assets.fixed.ppe.other.net",
        "assets.fixed.ppe.subtotal"
      ],
      "validation_rules": []
    },

    {
      "node_id": "assets.fixed.ppe.buildings.gross",
      "parent_id": "assets.fixed.ppe",
      "depth": 4,
      "node_type": "line_item_gross",
      "label_ja": "建物及び構築物",
      "label_en": "Buildings and Structures (gross)",
      "xbrl_element": "jpcrp_cor:BuildingsAndStructuresGross",
      "values": {
        "fy2025": {
          "raw_text": "2,121,360",
          "numeric": 2121360,
          "value_type": "numeric",
          "sign_source": "positive",
          "confidence": 0.99,
          "bbox": { "page": 1, "x0": 398.2, "y0": 287.4, "x1": 455.1, "y1": 297.8 }
        },
        "fy2026": {
          "raw_text": "1,880,082",
          "numeric": 1880082,
          "value_type": "numeric",
          "sign_source": "positive",
          "confidence": 0.99,
          "bbox": { "page": 1, "x0": 523.0, "y0": 287.4, "x1": 579.9, "y1": 297.8 }
        }
      },
      "contra_ref": null,
      "gross_ref": null,
      "children": [],
      "validation_rules": []
    },

    {
      "node_id": "assets.fixed.ppe.buildings.contra",
      "parent_id": "assets.fixed.ppe",
      "depth": 4,
      "node_type": "line_item_contra",
      "label_ja": "減価償却累計額",
      "label_en": "Accumulated Depreciation — Buildings and Structures",
      "xbrl_element": "jpcrp_cor:AccumulatedDepreciationBuildingsAndStructures",
      "values": {
        "fy2025": {
          "raw_text": "△1,923,585",
          "numeric": -1923585,
          "value_type": "numeric",
          "sign_source": "explicit_triangle",
          "confidence": 0.97,
          "bbox": { "page": 1, "x0": 385.0, "y0": 299.2, "x1": 455.1, "y1": 309.6 }
        },
        "fy2026": {
          "raw_text": "△1,640,253",
          "numeric": -1640253,
          "value_type": "numeric",
          "sign_source": "explicit_triangle",
          "confidence": 0.97,
          "bbox": { "page": 1, "x0": 510.0, "y0": 299.2, "x1": 579.9, "y1": 309.6 }
        }
      },
      "contra_ref": "assets.fixed.ppe.buildings.gross",
      "gross_ref": null,
      "children": [],
      "validation_rules": [
        {
          "rule_id": "contra_sign_buildings",
          "rule_type": "sign_check",
          "formula": "contra.numeric < 0",
          "operands": ["assets.fixed.ppe.buildings.contra"],
          "tolerance": 0
        }
      ]
    },

    {
      "node_id": "assets.fixed.ppe.buildings.net",
      "parent_id": "assets.fixed.ppe",
      "depth": 4,
      "node_type": "line_item_net",
      "label_ja": "建物及び構築物（純額）",
      "label_en": "Buildings and Structures (net book value)",
      "xbrl_element": "jpcrp_cor:BuildingsAndStructuresNet",
      "values": {
        "fy2025": {
          "raw_text": "197,775",
          "numeric": 197775,
          "value_type": "numeric",
          "sign_source": "positive",
          "confidence": 0.99,
          "bbox": { "page": 1, "x0": 398.2, "y0": 310.5, "x1": 455.1, "y1": 320.9 }
        },
        "fy2026": {
          "raw_text": "239,828",
          "numeric": 239828,
          "value_type": "numeric",
          "sign_source": "positive",
          "confidence": 0.99,
          "bbox": { "page": 1, "x0": 523.0, "y0": 310.5, "x1": 579.9, "y1": 320.9 }
        }
      },
      "contra_ref": null,
      "gross_ref": "assets.fixed.ppe.buildings.gross",
      "children": [],
      "validation_rules": [
        {
          "rule_id": "net_check_buildings_fy2025",
          "rule_type": "contra_net_check",
          "formula": "gross.numeric + contra.numeric == net.numeric",
          "operands": [
            "assets.fixed.ppe.buildings.gross:fy2025",
            "assets.fixed.ppe.buildings.contra:fy2025",
            "assets.fixed.ppe.buildings.net:fy2025"
          ],
          "tolerance": 1
        },
        {
          "rule_id": "net_check_buildings_fy2026",
          "rule_type": "contra_net_check",
          "formula": "gross.numeric + contra.numeric == net.numeric",
          "operands": [
            "assets.fixed.ppe.buildings.gross:fy2026",
            "assets.fixed.ppe.buildings.contra:fy2026",
            "assets.fixed.ppe.buildings.net:fy2026"
          ],
          "tolerance": 1
        }
      ]
    },

    {
      "node_id": "assets.fixed.ppe.land",
      "parent_id": "assets.fixed.ppe",
      "depth": 4,
      "node_type": "line_item_simple",
      "label_ja": "土地",
      "label_en": "Land",
      "xbrl_element": "jpcrp_cor:LandNet",
      "values": {
        "fy2025": { "raw_text": "768,976", "numeric": 768976, "value_type": "numeric", "sign_source": "positive", "confidence": 0.99, "bbox": null },
        "fy2026": { "raw_text": "705,403", "numeric": 705403, "value_type": "numeric", "sign_source": "positive", "confidence": 0.99, "bbox": null }
      },
      "contra_ref": null, "gross_ref": null,
      "children": [], "validation_rules": []
    },

    {
      "node_id": "assets.fixed.ppe.construction_in_progress",
      "parent_id": "assets.fixed.ppe",
      "depth": 4,
      "node_type": "line_item_simple",
      "label_ja": "建設仮勘定",
      "label_en": "Construction in Progress",
      "xbrl_element": "jpcrp_cor:ConstructionInProgress",
      "values": {
        "fy2025": {
          "raw_text": "—",
          "numeric": null,
          "value_type": "dash",
          "sign_source": "none",
          "confidence": 0.95,
          "bbox": null
        },
        "fy2026": {
          "raw_text": "4,553",
          "numeric": 4553,
          "value_type": "numeric",
          "sign_source": "positive",
          "confidence": 0.99,
          "bbox": null
        }
      },
      "contra_ref": null, "gross_ref": null,
      "children": [], "validation_rules": []
    },

    {
      "node_id": "assets.fixed.ppe.subtotal",
      "parent_id": "assets.fixed.ppe",
      "depth": 4,
      "node_type": "subtotal",
      "label_ja": "有形固定資産合計",
      "label_en": "Total Property, Plant and Equipment",
      "xbrl_element": "jpcrp_cor:PropertyPlantAndEquipmentNet",
      "values": {
        "fy2025": { "raw_text": "1,429,094", "numeric": 1429094, "value_type": "numeric", "sign_source": "positive", "confidence": 0.99, "bbox": null },
        "fy2026": { "raw_text": "1,352,610", "numeric": 1352610, "value_type": "numeric", "sign_source": "positive", "confidence": 0.99, "bbox": null }
      },
      "contra_ref": null, "gross_ref": null,
      "children": [],
      "validation_rules": [
        {
          "rule_id": "ppe_subtotal_sum_fy2025",
          "rule_type": "sum_check",
          "formula": "sum(net_lines) == subtotal",
          "operands": [
            "assets.fixed.ppe.buildings.net:fy2025",
            "assets.fixed.ppe.machinery.net:fy2025",
            "assets.fixed.ppe.land:fy2025",
            "assets.fixed.ppe.lease.net:fy2025",
            "assets.fixed.ppe.construction_in_progress:fy2025",
            "assets.fixed.ppe.other.net:fy2025",
            "assets.fixed.ppe.subtotal:fy2025"
          ],
          "tolerance": 1
        }
      ]
    }
  ]
}
```

> **Note on `dash` semantics:** `value_type: "dash"` records a displayed em-dash. The `numeric` is `null` (not `0`). When this node is referenced in a `sum_check`, the rule engine must treat `null` as `0` for arithmetic but must NOT create a `contra_net_check` rule pair. This distinction is critical for audit trail fidelity.

---

## 3. Evaluation & Correctness Framework

### 3.1 Multi-Tier Evaluation Architecture

```
Tier 0: Raw OCR Quality
Tier 1: Structural Correctness (TEDS-based)
Tier 2: Cell-Level Extraction (Semantic Triple F1)
Tier 3: Hierarchical Semantics (Depth & Node-Type Accuracy)
Tier 4: Domain Validation (Arithmetic Invariants)
Tier 5: Cross-Document Consistency (Period-over-Period Audit)
```

---

### Tier 0 — Raw OCR Quality

**Metrics:**
- **CER** (Character Error Rate) on label text: `CER = (S + D + I) / N` where S=substitutions, D=deletions, I=insertions, N=ground-truth chars
- **△-Preservation Rate**: `TPR_△ = (correctly_captured_△_tokens) / (total_△_tokens_in_GT)`
- **Numeric CER**: Evaluated separately on digit sequences to isolate comma/period confusion

**Targets:** CER < 2% on JP text; △-Preservation Rate > 99%; Numeric CER < 0.5%

---

### Tier 1 — Structural Correctness (TEDS)

Apply **Tree Edit Distance Similarity (TEDS)** from the TableFormer/PubTabNet literature, adapted for hierarchical accounting trees:

```
TEDS(T_pred, T_gt) = 1 - EditDist(T_pred, T_gt) / max(|T_pred|, |T_gt|)
```

**Operations costed:**
| Operation | Cost |
|---|---|
| Insert/delete `section_header` or `total` node | 1.5 (high, structural) |
| Insert/delete `line_item_*` node | 1.0 |
| Relabel `node_type` (e.g., `line_item_simple` → `line_item_gross`) | 0.8 |
| Reparent node (wrong depth) | 1.2 |
| Insert/delete column | 2.0 (catastrophic) |

**Decomposed TEDS variants:**
- `TEDS-S`: Structure only (ignore cell values)
- `TEDS-V`: Structure + values
- `TEDS-H`: Hierarchy edges only (parent-child fidelity)

**Target:** TEDS-S > 0.95; TEDS-H > 0.90 on a held-out set of 200 pages.

---

### Tier 2 — Cell-Level Extraction (Semantic Triple F1)

Represent each extracted datum as a **semantic triple**: `(node_id, column_id, numeric_value)`.

For the example document:
```
GT triples (subset):
  (assets.fixed.ppe.buildings.gross, fy2025, 2121360)
  (assets.fixed.ppe.buildings.contra, fy2025, -1923585)
  (assets.fixed.ppe.buildings.net, fy2025, 197775)
  (assets.fixed.ppe.land, fy2025, 768976)
  ...
```

**Matching criteria** (exact match unless tolerance specified):
- **Strict match:** `|pred_value - gt_value| <= tolerance` AND node label similarity ≥ 0.9
- **Soft match:** node label similarity ≥ 0.7 (label OCR variance), value exact

**Metrics:**
```
Precision  = |TP| / (|TP| + |FP|)
Recall     = |TP| / (|TP| + |FN|)
F1         = 2·P·R / (P + R)

Sign-aware F1: penalize sign-flip errors (△ dropped) with weight w=2
  → FP_sign = predicted positive, GT negative (counted as 2×FP)
```

**Per-category breakdown** (critical for Yuho analysis):
- `F1_simple`: Simple line items (no contra)
- `F1_contra`: Contra-asset lines (sign-sensitive)
- `F1_net`: Net-value lines
- `F1_dash`: Em-dash / nil cell accuracy
- `F1_subtotal`: Subtotal/total rows

**Target:** Overall F1 > 0.97; F1_contra > 0.99 (sign errors are accounting errors).

---

### Tier 3 — Hierarchical Semantic Accuracy

**Node-type classification:**

```
Confusion matrix over {section_header, category_header, line_item_gross,
                       line_item_contra, line_item_net, line_item_simple,
                       subtotal, total}

Key metric: macro-F1 over node_type labels
Critical off-diagonal: line_item_simple ↔ line_item_gross (most common error)
```

**Depth accuracy:**
```
Depth_MAE = mean(|pred_depth - gt_depth|) over all nodes
```
Target: Depth_MAE < 0.1 (> 90% of nodes at exact correct depth).

**Contra-pairing accuracy:**
```
Pairing_Accuracy = |correctly_linked (contra, gross) pairs| / |total GT pairs|
```
Each PP&E triple must correctly link `line_item_contra.contra_ref → line_item_gross.node_id`.

Target: Pairing_Accuracy > 0.99.

---

### Tier 4 — Domain Validation (Arithmetic Invariants)

These are **hard constraints** derivable from accounting identities. Violations indicate extraction errors, not just approximations.

**Invariant Set:**

```
V1 — Contra-Net Check (per asset, per period):
    gross.numeric + contra.numeric = net.numeric
    Tolerance: ±1 千円 (rounding)

    Applied to: buildings, machinery, lease assets, other (all 4 triples × 2 periods = 8 checks)

V2 — Subtotal Summation Check (per period):
    Σ(net values of components) = category subtotal
    Applied to: 流動資産合計, 有形固定資産合計, 無形固定資産合計, 投資その他の資産合計

V3 — Roll-up Check (per period):
    有形固定資産合計 + 無形固定資産合計 + 投資その他の資産合計 = 固定資産合計
    流動資産合計 + 固定資産合計 = 資産合計

V4 — Sign Invariants:
    All line_item_contra.numeric < 0
    All line_item_gross.numeric >= 0
    All subtotal.numeric >= 0 (for assets section)

V5 — Period Arithmetic (YoY change validation, if prior-year filed document available):
    Δvalue = current_period - prior_period
    Cross-validate Δ against disclosed YoY change columns (if present)
```

**Reporting:**
```
Arithmetic Pass Rate (APR):
  APR = (number of satisfied invariant checks) / (total invariant checks)

Weighted APR:
  APR_weighted: V1 checks weighted 2×, V3 (grand total) weighted 3×
  → Grand total error is highest severity
```

**Document-level verdict:**

| APR | Verdict |
|---|---|
| 1.00 | ✅ Arithmetically consistent |
| 0.95–0.99 | ⚠️ Minor discrepancy (rounding or one cell error) |
| < 0.95 | ❌ Structural parsing failure |

**Concrete check for this document (FY2025 column):**

```
V1: 2,121,360 + (−1,923,585) = 197,775  ✓
V1: 4,024,922 + (−3,634,165) = 390,757  ✓
V1: 210,746 + (−147,599)    =  63,147   ← GT shows 63,146 → |diff|=1, within tolerance ✓
V1: 848,814 + (−840,375)    =   8,439   ✓
V2: 197,775 + 390,757 + 768,976 + 63,146 + 0 + 8,439 = 1,429,093 ≈ 1,429,094 (Δ1, rounding) ✓
V3: 1,429,094 + 44,261 + 197,069 = 1,670,424 ≈ 1,670,426 (Δ2, rounding) ✓
V3: 5,863,684 + 1,670,426 = 7,534,110  ✓
```

All ground-truth invariants hold within ±2 千円 tolerance, confirming the GT is internally consistent and can serve as reliable test oracle.

---

### Tier 5 — Cross-Document Consistency

For filings with prior-year comparative columns:
- **Period continuity check**: `prior_year_col_in_current_doc == current_year_col_in_prior_doc`
- **Balance sheet identity**: Assets = Liabilities + Net Assets (when liabilities page also extracted)
- **XBRL cross-reference**: If XBRL inline data is available in the same EDINET filing, compute value-level agreement as a calibration signal for model confidence thresholds

---

### 3.2 Evaluation Pipeline Summary

```
                ┌──────────────────────────────────┐
  PDF Input ──▶ │  Preprocessing                   │
                │  (deskew, resolution normalization│
                │   Ja font normalization)          │
                └────────────────┬─────────────────┘
                                 │
                ┌────────────────▼─────────────────┐
                │  Tier 0: OCR Quality Gate         │
                │  CER < 2%, △-rate > 99%           │
                └────────────────┬─────────────────┘
                                 │
                ┌────────────────▼─────────────────┐
                │  Tier 1: Structure (TEDS-S > .95) │
                └────────────────┬─────────────────┘
                                 │
                ┌────────────────▼─────────────────┐
                │  Tier 2: Cell F1 > 0.97           │
                │  Tier 3: Node-type macro-F1 > .95 │
                └────────────────┬─────────────────┘
                                 │
                ┌────────────────▼─────────────────┐
                │  Tier 4: Arithmetic APR == 1.0    │ ◀── Hard gate
                │  (any V3 failure → reject output) │
                └────────────────┬─────────────────┘
                                 │
                ┌────────────────▼─────────────────┐
                │  Tier 5: Cross-doc consistency    │
                │  (only if prior filing available) │
                └──────────────────────────────────┘
```

Any tier failure routes to a **human review queue** with structured diff showing predicted vs. ground-truth for the failing invariant.

---

## Appendix: Key Design Decisions

| Decision | Rationale |
|---|---|
| `dash` ≠ `0` in CellValue | Preserves audit semantics; prevents false-zero in summation |
| `contra_ref` on contra nodes | Enables pairing without regex heuristics on label text |
| `tolerance: 1` in validation rules | Matches documented J-GAAP rounding (千円 unit, round-half-up) |
| `bbox` on every CellValue | Enables provenance tracing back to PDF coordinate for QA tooling |
| `xbrl_element` annotation | Bridges to EDINET machine-readable layer for cross-validation |
| Separate `sign_source` field | Distinguishes `explicit_triangle` from inherently-negative lines |
| Weighted TEDS for structural errors | Column deletion (weight 2.0) is more catastrophic than node relabeling |
