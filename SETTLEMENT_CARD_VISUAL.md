# Settlement View - Right Side Card Visual Reference

## Page: `admin/company/settlements/view/515`

---

## Right Card Layout (Visual)

```
╔════════════════════════════════════════════════╗
║  ORDER SUMMARY CARD                            ║
║  (Width: 360px)                                ║
╠════════════════════════════════════════════════╣
║                                                ║
║  Sub Total                        ₹986.39     ║
║  Shipping and Handling            ₹0.00       ║
║  Tax                              ₹0.00       ║
║  Discount                         ₹0.00       ║
║  ─────────────────────────────────────────    ║
║  GRAND TOTAL                      ₹986.39     ║
║  ─────────────────────────────────────────    ║
║  Total Paid                       ₹986.39     ║
║  Total Refund                     ₹0.00       ║
║  Admin Commission                 ₹197.28     ║
║  Seller Total                     ₹789.11     ║
║                                                ║
║  ┌─ IF COSTING ENABLED ─────────────────┐    ║
║  │ Buyer App Commission      ₹0.00      │    ║
║  │ Seller App Commission     ₹0.00      │    ║
║  │ TDS                       ₹0.00      │    ║
║  │ TCS                       ₹0.00      │    ║
║  │ Packing Cost              ₹0.00      │    ║
║  │ Domestic Shipping Cost    ₹0.00      │    ║
║  └───────────────────────────────────────┘    ║
║                                                ║
║  Total Due                        ₹0.00       ║
║                                                ║
╚════════════════════════════════════════════════╝
```

---

## Calculation Flow Diagram

```
                    ┌──────────────────┐
                    │  ORDER ITEMS     │
                    │  (4 items)       │
                    └────────┬─────────┘
                             │
                             ↓
         ┌───────────────────────────────────────┐
         │   Item 1: ₹102.04                     │
         │   Item 2: ₹113.38                     │
         │   Item 3: ₹317.46                     │
         │   Item 4: ₹453.51                     │
         └───────────────────┬───────────────────┘
                             │ SUM
                             ↓
                    ┌──────────────────┐
                    │   SUB TOTAL      │
                    │    ₹986.39       │
                    └────────┬─────────┘
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
         ↓                   ↓                   ↓
    ┌─────────┐       ┌──────────┐       ┌──────────┐
    │ + TAX   │       │ - DISC.  │       │ + SHIP.  │
    │  ₹0.00  │       │  ₹0.00   │       │  ₹0.00   │
    └─────────┘       └──────────┘       └──────────┘
         │                   │                   │
         └───────────────────┼───────────────────┘
                             │
                             ↓
                    ┌──────────────────┐
                    │  GRAND TOTAL     │
                    │    ₹986.39       │
                    └────────┬─────────┘
                             │
                    ┌────────┴─────────┐
                    │                  │
                    ↓                  ↓
         ┌─────────────────┐  ┌─────────────────┐
         │ ADMIN COMM.     │  │  COST COMM.     │
         │  20% = ₹197.28  │  │  (if enabled)   │
         └────────┬────────┘  └────────┬────────┘
                  │                    │
                  └──────────┬─────────┘
                             │ SUBTRACT
                             ↓
                    ┌──────────────────┐
                    │  SELLER TOTAL    │
                    │    ₹789.11       │
                    └──────────────────┘
                             │
                    ┌────────┴─────────┐
                    │                  │
                    ↓                  ↓
         ┌─────────────────┐  ┌─────────────────┐
         │  TOTAL PAID     │  │  TOTAL REFUND   │
         │   ₹986.39       │  │    ₹0.00        │
         └─────────────────┘  └─────────────────┘
                    │                  │
                    └──────────┬───────┘
                               │
                               ↓
                    ┌──────────────────┐
                    │   TOTAL DUE      │
                    │     ₹0.00        │
                    └──────────────────┘
```

---

## Commission Calculation Details

### 1. Admin Commission (Always Shown)
```
┌─────────────────────────────────────────────┐
│  Base Amount: ₹986.39 (Sub Total)           │
│  Commission %: 20%                           │
│                                              │
│  Calculation:                                │
│  ₹986.39 × 20 ÷ 100 = ₹197.28               │
│                                              │
│  Result: ₹197.28                             │
└─────────────────────────────────────────────┘
```

### 2. Buyer App Commission (If Enabled)
```
┌─────────────────────────────────────────────┐
│  Base Amount: ₹986.39 (Sub Total)           │
│  Commission %: 2.5% (example)                │
│  GST %: 18%                                  │
│                                              │
│  Step 1: Commission without GST              │
│  ₹986.39 × 2.5 ÷ 100 = ₹24.66               │
│                                              │
│  Step 2: Add GST                             │
│  ₹24.66 + (₹24.66 × 18 ÷ 100) = ₹29.10     │
│                                              │
│  Result: ₹29.10                              │
└─────────────────────────────────────────────┘
```

### 3. TDS Commission (If Enabled)
```
┌─────────────────────────────────────────────┐
│  Base Amount: ₹986.39 (Sub Total)           │
│  TDS %: 1% (example)                         │
│  GST: NOT APPLIED                            │
│                                              │
│  Calculation:                                │
│  ₹986.39 × 1 ÷ 100 = ₹9.86                  │
│                                              │
│  Result: ₹9.86 (no GST)                      │
└─────────────────────────────────────────────┘
```

### 4. Packing Cost (If Enabled)
```
┌─────────────────────────────────────────────┐
│  Fixed Amount: ₹10.00 (example)              │
│  GST %: 18%                                  │
│                                              │
│  Step 1: Base amount (fixed)                 │
│  ₹10.00                                      │
│                                              │
│  Step 2: Add GST                             │
│  ₹10.00 + (₹10.00 × 18 ÷ 100) = ₹11.80     │
│                                              │
│  Result: ₹11.80                              │
└─────────────────────────────────────────────┘
```

---

## Seller Total Calculation

### Without Costing Bifurcation
```
┌──────────────────────────────────────────────┐
│  Grand Total:           ₹986.39              │
│  - Admin Commission:    ₹197.28              │
│  ─────────────────────────────────           │
│  SELLER TOTAL:          ₹789.11              │
└──────────────────────────────────────────────┘
```

### With Full Costing Bifurcation (Example)
```
┌──────────────────────────────────────────────┐
│  Grand Total:                    ₹986.39     │
│  - Admin Commission:             ₹197.28     │
│  - Buyer App Commission:         ₹29.10      │
│  - Seller App Commission:        ₹24.66      │
│  - TDS:                          ₹9.86       │
│  - TCS:                          ₹9.86       │
│  - Packing Cost:                 ₹11.80      │
│  - Domestic Shipping Cost:       ₹23.60      │
│  ───────────────────────────────────────     │
│  SELLER TOTAL:                   ₹680.23     │
└──────────────────────────────────────────────┘
```

---

## GST Application Matrix

| Commission Type            | Base Calculation           | GST Applied? | Formula                                    |
|----------------------------|----------------------------|--------------|---------------------------------------------|
| Admin Commission           | % of Sub Total             | ❌ No        | `(Sub Total × %) / 100`                    |
| Buyer App Commission       | % of Sub Total             | ✅ Yes       | `Base × (1 + GST% / 100)`                  |
| Seller App Commission      | % of Sub Total             | ✅ Yes       | `Base × (1 + GST% / 100)`                  |
| TDS                        | % of Sub Total             | ❌ No        | `(Sub Total × %) / 100`                    |
| TCS                        | % of Sub Total             | ❌ No        | `(Sub Total × %) / 100`                    |
| Packing Cost               | Fixed Amount               | ✅ Yes       | `Fixed Amount × (1 + GST% / 100)`          |
| Domestic Shipping Cost     | Fixed Amount               | ✅ Yes       | `Fixed Amount × (1 + GST% / 100)`          |

---

## Database Field Mapping

```
marketplace_orders table
│
├── SUBTOTAL FIELDS
│   ├── base_sub_total                        → "Sub Total"
│   ├── base_sub_total_incl_tax              → "Sub Total (Incl. Tax)"
│   
├── OTHER COMPONENTS
│   ├── base_shipping_amount                 → "Shipping and Handling"
│   ├── base_tax_amount                      → "Tax"
│   ├── base_discount_amount                 → "Discount"
│   
├── GRAND TOTAL
│   ├── base_grand_total                     → "Grand Total"
│   ├── base_grand_total_invoiced            → "Total Paid"
│   ├── base_grand_total_refunded            → "Total Refund"
│   
├── ADMIN COMMISSION
│   ├── commission_percentage                → Admin %
│   ├── base_commission                      → "Admin Commission"
│   
├── SELLER AMOUNT
│   ├── base_seller_total                    → "Seller Total"
│   ├── base_total_due                       → "Total Due"
│   
├── COST COMMISSIONS (Optional)
│   ├── buyer_app_commission_percentage      → Buyer App %
│   ├── base_buyer_app_commission            → "Buyer App Commission"
│   ├── seller_app_commission_percentage     → Seller App %
│   ├── base_seller_app_commission           → "Seller App Commission"
│   ├── tds_commission_percentage            → TDS %
│   ├── base_tds_commission                  → "TDS"
│   ├── tcs_commission_percentage            → TCS %
│   ├── base_tcs_commission                  → "TCS"
│   ├── packing_cost_commission_percentage   → Packing Cost (fixed)
│   ├── base_packing_cost_commission         → "Packing Cost"
│   ├── domestic_shipping_cost_commission_percentage → Shipping (fixed)
│   └── base_domestic_shipping_cost_commission → "Domestic Shipping Cost"
```

---

## Costing Bifurcation Modes

### Mode 1: Disabled
```
┌──────────────────────────────────────┐
│  Card Shows:                         │
│  ✓ Sub Total                         │
│  ✓ Shipping                          │
│  ✓ Tax                               │
│  ✓ Discount                          │
│  ✓ Grand Total                       │
│  ✓ Total Paid                        │
│  ✓ Total Refund                      │
│  ✓ Admin Commission                  │
│  ✓ Seller Total                      │
│  ✗ Cost Commissions (hidden)         │
│  ✓ Total Due                         │
└──────────────────────────────────────┘
```

### Mode 2: Display Only
```
┌──────────────────────────────────────┐
│  Card Shows:                         │
│  ✓ All standard fields               │
│  ✓ Cost Commissions (calculated)     │
│                                       │
│  Note: Commissions calculated        │
│  on-the-fly from config, not from    │
│  database. Used for display only.    │
└──────────────────────────────────────┘
```

### Mode 3: Full
```
┌──────────────────────────────────────┐
│  Card Shows:                         │
│  ✓ All standard fields               │
│  ✓ Cost Commissions (from DB)        │
│                                       │
│  Note: Commissions retrieved from    │
│  database. Already applied to        │
│  product pricing and order totals.   │
└──────────────────────────────────────┘
```

---

## Data Flow: From Order to Settlement

```
┌──────────────────────────────────────────────────────┐
│  STEP 1: Customer Places Order                       │
│  - Products added to cart                            │
│  - Order created in 'orders' table                   │
└─────────────────────┬────────────────────────────────┘
                      │
                      ↓
┌──────────────────────────────────────────────────────┐
│  STEP 2: Marketplace Order Created                   │
│  - Seller-specific order in 'marketplace_orders'     │
│  - Calculate subtotal, tax, shipping, discount       │
│  - Calculate admin commission                        │
│  - Calculate cost commissions (if enabled)           │
│  - Calculate seller total                            │
└─────────────────────┬────────────────────────────────┘
                      │
                      ↓
┌──────────────────────────────────────────────────────┐
│  STEP 3: Invoice Generated                           │
│  - Invoice created                                   │
│  - Updates 'base_grand_total_invoiced'               │
│  - This becomes "Total Paid"                         │
└─────────────────────┬────────────────────────────────┘
                      │
                      ↓
┌──────────────────────────────────────────────────────┐
│  STEP 4: ONDC Settlement Process                     │
│  - Order confirmation request created                │
│  - Settlement request sent to ONDC                   │
│  - Tracked in 'order_confirm_requests'               │
└─────────────────────┬────────────────────────────────┘
                      │
                      ↓
┌──────────────────────────────────────────────────────┐
│  STEP 5: View Settlement Page                        │
│  - Admin accesses: admin/company/settlements/view/ID │
│  - All calculations displayed in right card          │
│  - Settlement status shown (settled/not settled)     │
└──────────────────────────────────────────────────────┘
```

---

## Configuration Locations

### In Database (core_config table)
```
Config Key: ondc_seller.costing_bifurcation.settings.mode
Values: 
  - 'disabled'
  - 'display_only'
  - 'full'

Config Key: ondc_seller.costing_bifurcation.settings.gst_percentage
Value: 18.00 (example)

Config Key: ondc_seller.costing_bifurcation.settings.buyer_app_commission_percentage
Value: 2.50 (example)

... (similar for all other commission types)
```

### Access Path in Admin Panel
```
Configuration → ONDC Seller → Costing Bifurcation → Settings
```

---

## Real Example Walkthrough (Order #1696)

```
INITIAL VALUES
─────────────────────────────────────────────────
Order Items:
  1. Mesh Fry Strainer:     ₹102.04
  2. Pattern Discs Press:   ₹113.38
  3. Grill Uttapam Tawa:    ₹317.46
  4. Storage Cover:         ₹453.51

STEP 1: CALCULATE SUBTOTAL
─────────────────────────────────────────────────
Sub Total = 102.04 + 113.38 + 317.46 + 453.51
          = ₹986.39

STEP 2: ADD OTHER COMPONENTS
─────────────────────────────────────────────────
Tax                    = ₹0.00
Discount               = ₹0.00
Shipping               = ₹0.00

STEP 3: CALCULATE GRAND TOTAL
─────────────────────────────────────────────────
Grand Total = 986.39 + 0 - 0 + 0
            = ₹986.39

STEP 4: CALCULATE ADMIN COMMISSION
─────────────────────────────────────────────────
Commission % = 20%
Commission = 986.39 × 20 ÷ 100
           = ₹197.28

STEP 5: CALCULATE SELLER TOTAL
─────────────────────────────────────────────────
Seller Total = 986.39 - 197.28
             = ₹789.11

STEP 6: INVOICE & PAYMENT
─────────────────────────────────────────────────
Total Paid = ₹986.39 (fully invoiced)
Total Refund = ₹0.00

STEP 7: CALCULATE TOTAL DUE
─────────────────────────────────────────────────
Total Due = 986.39 - 986.39 - 0
          = ₹0.00 (fully paid)

FINAL SUMMARY
─────────────────────────────────────────────────
Sub Total:          ₹986.39
Shipping:           ₹0.00
Tax:                ₹0.00
Discount:           ₹0.00
──────────────────────────
Grand Total:        ₹986.39
Total Paid:         ₹986.39
Total Refund:       ₹0.00
Admin Commission:   ₹197.28
Seller Total:       ₹789.11
Total Due:          ₹0.00
```

---

## Key Takeaways

1. **Sub Total** = Sum of all item prices (before tax, discount, shipping)

2. **Grand Total** = Sub Total + Tax + Shipping - Discount

3. **Admin Commission** = Grand Total × Commission %

4. **Seller Total** = Grand Total - All Commissions

5. **Total Paid** = Amount invoiced (usually equals Grand Total)

6. **Total Due** = Grand Total - Total Paid - Total Refund

7. **Cost Commissions** are optional and depend on configuration mode

8. **GST** is applied to some commissions but not others

9. **Packing Cost & Domestic Shipping** are fixed amounts, not percentages

10. **Display Only Mode** calculates commissions for viewing without affecting actual pricing

