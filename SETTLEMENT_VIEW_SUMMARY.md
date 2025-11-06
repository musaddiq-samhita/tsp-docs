# Settlement View - Executive Summary

## 📄 Document Overview

This document provides a complete breakdown of the Settlement View page at:
**`https://smartsell.samhita.org/admin/company/settlements/view/515`**

Specifically focusing on the **right side card** that displays order financial calculations.

---

## 📚 Related Documentation

1. **SETTLEMENT_VIEW_BREAKDOWN.md** - Complete technical breakdown with calculation logic
2. **SETTLEMENT_CARD_VISUAL.md** - Visual diagrams and examples
3. **SETTLEMENT_SQL_QUERIES.md** - Database queries for analysis
4. **This file** - Executive summary and quick reference

---

## 🎯 Quick Answer: What Does Each Field Mean?

| Field | What It Shows | Formula |
|-------|--------------|---------|
| **Sub Total** | Sum of all item prices | Sum of item totals |
| **Shipping and Handling** | Shipping cost | From shipping rates |
| **Tax** | Total tax on items | Sum of item taxes |
| **Discount** | Total discount applied | Sum of item discounts |
| **Grand Total** | Final order amount | Sub Total + Tax + Shipping - Discount |
| **Total Paid** | Amount invoiced | Sum from invoices |
| **Total Refund** | Amount refunded | From credit memos |
| **Admin Commission** | Platform commission | Grand Total × Commission % |
| **Seller Total** | Amount seller receives | Grand Total - All Commissions |
| **Buyer App Commission** | ONDC buyer app fee | (Sub Total × %) × (1 + GST%) |
| **Seller App Commission** | ONDC seller app fee | (Sub Total × %) × (1 + GST%) |
| **TDS** | Tax Deducted at Source | Sub Total × % (no GST) |
| **TCS** | Tax Collected at Source | Sub Total × % (no GST) |
| **Packing Cost** | Packing charge | Fixed Amount × (1 + GST%) |
| **Domestic Shipping Cost** | Shipping charge | Fixed Amount × (1 + GST%) |
| **Total Due** | Remaining amount | Grand Total - Paid - Refund |

---

## 🔢 Real Example (Order #1696)

```
ITEMS IN ORDER:
─────────────────────────────────────────
1. Mesh Fry Strainer         ₹102.04
2. Pattern Discs Press        ₹113.38
3. Grill Uttapam Tawa         ₹317.46
4. Storage Cover              ₹453.51

CALCULATIONS:
─────────────────────────────────────────
Sub Total              ₹986.39   (sum of items)
+ Shipping             ₹0.00
+ Tax                  ₹0.00
- Discount             ₹0.00
─────────────────────────────────────────
= GRAND TOTAL          ₹986.39

- Admin Commission     ₹197.28   (20% of 986.39)
─────────────────────────────────────────
= SELLER TOTAL         ₹789.11

PAYMENT STATUS:
─────────────────────────────────────────
Total Paid             ₹986.39   (invoiced)
Total Refund           ₹0.00
─────────────────────────────────────────
Total Due              ₹0.00     (fully paid)
```

---

## 💾 Database Location

**Primary Table:** `marketplace_orders`

**Key Fields:**
```
marketplace_orders.id = 1680
marketplace_orders.order_id = 1696
marketplace_orders.base_sub_total = 986.39
marketplace_orders.base_grand_total = 986.39
marketplace_orders.base_commission = 197.28
marketplace_orders.base_seller_total = 789.11
```

**Related Tables:**
- `orders` - Main order data
- `order_items` - Individual products
- `marketplace_order_items` - Seller item data
- `order_confirm_requests` - ONDC settlement tracking
- `invoices` - Payment records

---

## 🔧 How It Works (Step by Step)

### Step 1: Customer Places Order
- Customer adds products to cart
- Order is created in system
- Record in `orders` table

### Step 2: Order Split by Seller
- System identifies which items belong to which seller
- Creates `marketplace_orders` record for each seller
- Calculates all totals and commissions

### Step 3: Invoice Generated
- Invoice created when order is processed
- `base_grand_total_invoiced` updated
- This becomes "Total Paid"

### Step 4: Settlement Process
- ONDC settlement request created
- Tracked in `order_confirm_requests`
- Settlement status updated when completed

### Step 5: View Page
- Admin accesses settlement view
- Controller loads data from database
- Right card displays all calculations

---

## 🧮 Commission Types Explained

### 1. Admin Commission (Always Present)
```
Purpose: Platform's commission from seller
Calculation: Percentage of sub total
GST: Not applied separately
Example: 20% of ₹986.39 = ₹197.28
```

### 2. Cost Commissions (Optional - Based on Configuration)

**When shown?** Only if costing bifurcation mode is enabled

#### A. Buyer App Commission
```
Purpose: Fee for buyer-side ONDC app
Calculation: Percentage of sub total + GST
GST: Yes (18%)
Example: 2.5% of ₹986.39 = ₹24.66 + GST = ₹29.10
```

#### B. Seller App Commission
```
Purpose: Fee for seller-side ONDC app
Calculation: Percentage of sub total + GST
GST: Yes (18%)
Example: 2% of ₹986.39 = ₹19.73 + GST = ₹23.28
```

#### C. TDS (Tax Deducted at Source)
```
Purpose: Income tax deduction
Calculation: Percentage of sub total
GST: No
Example: 1% of ₹986.39 = ₹9.86
```

#### D. TCS (Tax Collected at Source)
```
Purpose: Sales tax collection
Calculation: Percentage of sub total
GST: No
Example: 1% of ₹986.39 = ₹9.86
```

#### E. Packing Cost
```
Purpose: Fixed packing charge
Calculation: Fixed amount + GST
GST: Yes (18%)
Example: ₹10.00 + GST = ₹11.80
Note: This is NOT a percentage, it's a fixed amount
```

#### F. Domestic Shipping Cost
```
Purpose: Fixed shipping charge
Calculation: Fixed amount + GST
GST: Yes (18%)
Example: ₹20.00 + GST = ₹23.60
Note: This is NOT a percentage, it's a fixed amount
```

---

## ⚙️ Configuration Modes

### Mode 1: Disabled (Default)
```
✓ Shows standard fields only
✗ No cost commissions shown
✓ Simple commission structure
```

### Mode 2: Display Only
```
✓ Shows all fields including cost commissions
✓ Commissions calculated on-the-fly
✗ Doesn't affect product pricing
✓ For reporting purposes only
```

### Mode 3: Full
```
✓ Shows all fields including cost commissions
✓ Commissions stored in database
✓ Affects product pricing
✓ Applied throughout system
```

**To change mode:** Go to Configuration → ONDC Seller → Costing Bifurcation → Settings → Mode

---

## 📊 Where Values Come From

| Field | Source | Calculation Point |
|-------|--------|-------------------|
| Sub Total | Database | When order is placed |
| Shipping | Database | When shipping method selected |
| Tax | Database | When order items are totaled |
| Discount | Database | When promotions applied |
| Grand Total | Database | When order is finalized |
| Total Paid | Database | When invoice is created |
| Total Refund | Database | When credit memo is created |
| Admin Commission | Database | When order is placed |
| Seller Total | Database | When order is placed |
| Cost Commissions | Database OR Calculated | Depends on mode |
| Total Due | Database | Continuously updated |

---

## 🔍 Common Questions

### Q1: Why is Total Paid different from Grand Total?
**A:** This happens when:
- Order is partially invoiced
- Multiple invoices were created
- Refunds were issued

### Q2: What if cost commissions show ₹0.00?
**A:** This means:
- Costing bifurcation mode is "Disabled"
- OR configuration percentages are set to 0
- OR display mode is "Display Only" with 0% rates

### Q3: How is Seller Total calculated?
**A:** 
```
Seller Total = Grand Total 
             - Admin Commission 
             - All Cost Commissions (if enabled)
```

### Q4: What does "Total Due" represent?
**A:**
```
Total Due = Grand Total 
          - Total Paid 
          - Total Refund
```
This shows if there's any outstanding amount.

### Q5: Can I recalculate these values?
**A:** Yes, use the SQL queries in `SETTLEMENT_SQL_QUERIES.md` file, specifically Query #3 (Calculate and Verify Totals).

---

## 🎨 Visual Card Structure

```
┌──────────────────────────────────────┐
│  PRICING SECTION                     │
│  • Sub Total                         │
│  • Shipping                          │
│  • Tax                               │
│  • Discount                          │
├──────────────────────────────────────┤
│  TOTAL SECTION                       │
│  • Grand Total (bold)                │
├──────────────────────────────────────┤
│  PAYMENT SECTION                     │
│  • Total Paid                        │
│  • Total Refund                      │
├──────────────────────────────────────┤
│  COMMISSION SECTION                  │
│  • Admin Commission                  │
│  • Seller Total                      │
├──────────────────────────────────────┤
│  COST COMMISSION SECTION (optional)  │
│  • Buyer App                         │
│  • Seller App                        │
│  • TDS                               │
│  • TCS                               │
│  • Packing Cost                      │
│  • Domestic Shipping Cost            │
├──────────────────────────────────────┤
│  FINAL SECTION                       │
│  • Total Due                         │
└──────────────────────────────────────┘
```

---

## 🚀 Quick Start Guide

### To View Settlement Details:
1. Go to Admin Panel
2. Navigate to: Company → Settlements
3. Click on any order row
4. View opens showing order details and right side card

### To Get Data from Database:
```sql
-- Run this query (replace 515 with your ID)
SELECT * FROM marketplace_orders mo
JOIN orders o ON mo.order_id = o.id
LEFT JOIN order_confirm_requests ocr ON o.id = ocr.order_id
WHERE ocr.id = 515;
```

### To Configure Costing:
1. Admin Panel → Configuration
2. ONDC Seller → Costing Bifurcation
3. Set Mode (Disabled/Display Only/Full)
4. Set percentages and amounts
5. Save configuration

---

## 📁 File References

### View Files
```
packages/Webkul/ONDCSeller/src/Resources/views/admin/tenant/settlements/view.blade.php
- Line 234-424: Right side card
- Line 255-422: Individual field rendering
```

### Controller
```
packages/Webkul/ONDCSeller/src/Http/Controllers/Admin/SettlementController.php
- Method: view($id)
- Loads order data and passes to view
```

### Calculation Logic
```
packages/Webkul/ONDCSeller/src/Repositories/Seller/OrderRepository.php
- Method: collectTotals()
- Calculates all order totals

packages/Webkul/ONDCSeller/src/Helpers/Order.php
- Method: calculateCommissionForDisplay()
- Calculates cost commissions
```

---

## 🔐 Access Control

**Required Permission:** `company.settlements`

**Bouncer Check:**
```php
@if (bouncer()->hasPermission('company.settlements'))
    // Show settlement view
@endif
```

**To grant access:**
1. Admin Panel → Roles
2. Edit role
3. Enable "Company Settlements" permission

---

## 💡 Key Insights

### 1. Two Calculation Modes
- **Stored Mode**: Values saved in database when order is placed
- **Display Mode**: Values calculated on-the-fly for viewing only

### 2. GST Application Rules
- **Applied to**: Buyer App, Seller App, Packing, Shipping
- **NOT applied to**: TDS, TCS, Admin Commission

### 3. Fixed vs Percentage
- **Percentage-based**: Most commissions (Buyer App, Seller App, TDS, TCS, Admin)
- **Fixed Amount**: Packing Cost, Domestic Shipping Cost

### 4. Seller Total Formula
```
Seller gets what's left after all deductions:
Grand Total - All Commissions = Seller Total
```

### 5. Settlement Status
- **Not Settled**: Payment pending to seller
- **Settled**: Payment completed to seller
- Tracked via `order_confirm_requests.is_settled`

---

## 📈 Impact on Business

### Platform Revenue
```
Platform Revenue = Admin Commission 
                 + Buyer App Commission
                 + Seller App Commission
                 + TDS + TCS
                 + Packing Cost
                 + Domestic Shipping Cost
```

### Seller Revenue
```
Seller Revenue = Grand Total - Platform Revenue
```

### Commission Percentage
```
Typical breakdown:
- Admin Commission: 15-25%
- Buyer App: 0-3%
- Seller App: 0-3%
- TDS: 0-2%
- TCS: 0-2%
- Packing: Fixed ₹5-15
- Shipping: Fixed ₹10-30
```

---

## 🛠️ Troubleshooting

### Issue: Numbers don't add up
**Solution:**
1. Run verification query (#3 from SQL queries)
2. Check for rounding differences
3. Verify configuration settings

### Issue: Cost commissions not showing
**Solution:**
1. Check costing mode: Config → ONDC Seller → Costing Bifurcation
2. Ensure mode is "Display Only" or "Full"
3. Check percentage values are > 0

### Issue: Seller total seems incorrect
**Solution:**
1. Calculate manually: Grand Total - All Commissions
2. Verify all commission fields in database
3. Check if any commissions are missing from calculation

### Issue: Can't find order in settlements
**Solution:**
1. Verify order has `order_confirm_requests` record
2. Check company_id matches
3. Ensure order is in correct status

---

## 📞 Support Information

### For Calculation Issues
- Refer to: `SETTLEMENT_VIEW_BREAKDOWN.md`
- Section: "Calculation Logic"

### For Database Queries
- Refer to: `SETTLEMENT_SQL_QUERIES.md`
- Use Query #3 for verification

### For Visual Reference
- Refer to: `SETTLEMENT_CARD_VISUAL.md`
- See diagrams and examples

---

## 📝 Notes

1. **All amounts are in base currency** (usually INR)
2. **Fields prefixed with `base_`** indicate base currency amounts
3. **Order ID vs Marketplace Order ID**: 
   - Order ID = Main order
   - Marketplace Order ID = Seller-specific order
4. **Invoice amounts might differ** if order is partially invoiced
5. **Total Due can be negative** if refund > grand total

---

## 🎓 Learning Path

### Beginner Level
1. Read this summary
2. View `SETTLEMENT_CARD_VISUAL.md` for diagrams
3. Understand basic fields (Sub Total, Grand Total, etc.)

### Intermediate Level
4. Read `SETTLEMENT_VIEW_BREAKDOWN.md`
5. Understand commission calculations
6. Learn about costing modes

### Advanced Level
7. Study `SETTLEMENT_SQL_QUERIES.md`
8. Analyze database structure
9. Create custom reports
10. Modify calculation logic

---

## 🔄 Quick Reference Formulas

```javascript
// Basic Calculations
subTotal = sum(itemPrices)
grandTotal = subTotal + tax + shipping - discount

// Admin Commission
adminCommission = grandTotal × commissionPercentage / 100

// Cost Commissions (with GST)
buyerAppBase = subTotal × buyerAppPercentage / 100
buyerAppWithGST = buyerAppBase × (1 + gstPercentage / 100)

// Cost Commissions (without GST)
tds = subTotal × tdsPercentage / 100
tcs = subTotal × tcsPercentage / 100

// Fixed Costs (with GST)
packingCost = fixedAmount × (1 + gstPercentage / 100)
shippingCost = fixedAmount × (1 + gstPercentage / 100)

// Final Calculations
allCommissions = adminCommission + buyerApp + sellerApp 
                + tds + tcs + packing + shipping
                
sellerTotal = grandTotal - allCommissions

totalDue = grandTotal - totalPaid - totalRefund
```

---

## 📅 Document Information

**Last Updated:** November 6, 2025  
**Version:** 1.0  
**Based on Order:** #1696 (marketplace_orders.id = 1680)  
**Company ID:** 22  
**Database:** laravel_db

---

## ✅ Summary Checklist

- [x] Understand what each field represents
- [x] Know where data comes from (database vs calculated)
- [x] Learn commission calculation formulas
- [x] Understand GST application rules
- [x] Know the difference between fixed and percentage commissions
- [x] Understand costing bifurcation modes
- [x] Know how to query database for verification
- [x] Understand seller total calculation
- [x] Know how settlement process works

---

**For detailed technical information, refer to the accompanying documentation files.**

