# Settlement View Page - Complete Breakdown

## Page URL
`https://smartsell.samhita.org/admin/company/settlements/view/515`

## Controller
**File**: `packages/Webkul/ONDCSeller/src/Http/Controllers/Admin/SettlementController.php`
**Method**: `view($id)`

### Data Flow
1. The `$id` parameter (515) refers to the `order_confirm_requests.id`
2. Controller fetches:
   - `$orderRequest` from `order_confirm_requests` table
   - `$order` from `orders` table
   - `$sellerOrder` from `marketplace_orders` table

---

## Right Side Card - Component Breakdown

**View File**: `packages/Webkul/ONDCSeller/src/Resources/views/admin/tenant/settlements/view.blade.php`  
**Location**: Lines 234-424 (Right side card with w-[360px])

---

### 1. **Sub Total** (Line 266-343)

#### Display Modes
The system can display subtotal in three modes based on configuration:

**Configuration**: `sales.taxes.sales.display_subtotal`

**Mode A: Excluding Tax (default)**
```php
Display: "Sub Total"
Value: $sellerOrder->base_sub_total
```

**Mode B: Including Tax**
```php
Display: "Sub Total"
Value: $sellerOrder->base_sub_total  // Still shows excluding tax value
```

**Mode C: Both (Excl + Incl Tax)**
```php
Display Line 1: "Sub Total (Excl. Tax)"
Value 1: $sellerOrder->base_sub_total

Display Line 2: "Sub Total (Incl. Tax)"
Value 2: $sellerOrder->base_sub_total_incl_tax
```

#### Calculation Logic (from OrderRepository.php line 264-271)
```php
// Calculated by summing all order items
$order->sub_total = sum of all items: item->total
$order->base_sub_total = sum of all items: item->base_total

// Including tax
$order->sub_total_incl_tax = sum of all items: (item->tax_amount + item->total)
$order->base_sub_total_incl_tax = sum of all items: (item->base_tax_amount + item->base_total)
```

#### Database Fields
- `marketplace_orders.base_sub_total`
- `marketplace_orders.base_sub_total_incl_tax`

---

### 2. **Shipping and Handling** (Line 271-364)

#### Visibility
Only shown if `$sellerOrder->order->haveStockableItems() == true`

#### Display Modes
**Configuration**: `sales.taxes.sales.display_shipping_amount`

**Mode A: Excluding Tax (default)**
```php
Display: "Shipping and Handling"
Value: $sellerOrder->base_shipping_amount
```

**Mode B: Including Tax**
```php
Display: "Shipping and Handling"
Value: $sellerOrder->base_shipping_amount_incl_tax
```

**Mode C: Both**
```php
Display Line 1: "Shipping and Handling (Excl. Tax)"
Value 1: $sellerOrder->base_shipping_amount

Display Line 2: "Shipping and Handling (Incl. Tax)"
Value 2: $sellerOrder->base_shipping_amount_incl_tax
```

#### Calculation Logic (from OrderRepository.php line 240-254)
```php
// Retrieved from session shipping rates
$shippingCodes = explode('_', $order->order->shipping_method);
$carrier = current($shippingCodes);
$shippingMethod = end($shippingCodes);

$marketplaceShippingRates = session()->get('marketplace_shipping_rates');

if (shipping rate exists for seller) {
    $order->shipping_amount = $sellerShippingRate['amount'];
    $order->base_shipping_amount = $sellerShippingRate['base_amount'];
}
```

#### Database Fields
- `marketplace_orders.base_shipping_amount`
- `marketplace_orders.base_shipping_amount_incl_tax`

---

### 3. **Tax** (Line 367-368)

```php
Display: "Tax"
Value: $sellerOrder->base_tax_amount
```

#### Calculation Logic (from OrderRepository.php line 267-268)
```php
// Sum of all item taxes
$order->tax_amount = sum of all items: item->tax_amount
$order->base_tax_amount = sum of all items: item->base_tax_amount
```

#### Database Field
- `marketplace_orders.base_tax_amount`

---

### 4. **Discount** (Line 371-372)

```php
Display: "Discount"
Value: $sellerOrder->base_discount_amount
```

#### Calculation Logic (from OrderRepository.php line 258-259)
```php
// Sum of all item discounts
$order->discount_amount = sum of all items: item->discount_amount
$order->base_discount_amount = sum of all items: item->base_discount_amount
```

#### Database Field
- `marketplace_orders.base_discount_amount`

---

### 5. **Grand Total** (Line 375-376)

```php
Display: "Grand Total" (Bold, larger font)
Value: $sellerOrder->base_grand_total
```

#### Calculation Logic (from OrderRepository.php line 260-262, 289-292)
```php
// Sum of all items with tax and discount
$order->grand_total = sum of all items: (item->total + item->tax_amount - item->discount_amount)
$order->base_grand_total = sum of all items: (item->base_total + item->base_tax_amount - item->base_discount_amount)

// Add shipping if exists
if ($order->shipping_amount > 0) {
    $order->grand_total += $order->shipping_amount;
    $order->base_grand_total += $order->base_shipping_amount;
}
```

#### Formula
```
Grand Total = Sub Total + Tax - Discount + Shipping
```

#### Database Field
- `marketplace_orders.base_grand_total`

---

### 6. **Total Paid** (Line 379-380)

```php
Display: "Total Paid"
Value: $sellerOrder->base_grand_total_invoiced
```

#### Calculation Logic (from OrderRepository.php line 307-355)
```php
// Sum from all invoices
foreach ($order->invoices as $invoice) {
    $order->sub_total_invoiced += $invoice->sub_total;
    $order->base_sub_total_invoiced += $invoice->base_sub_total;
    $order->shipping_invoiced += $invoice->shipping_amount;
    $order->base_shipping_invoiced += $invoice->base_shipping_amount;
    $order->tax_amount_invoiced += $invoice->tax_amount;
    $order->base_tax_amount_invoiced += $invoice->base_tax_amount;
    $order->discount_amount_invoiced += $invoice->discount_amount;
    $order->base_discount_amount_invoiced += $invoice->base_discount_amount;
}

// Final calculation
$order->grand_total_invoiced = $order->sub_total_invoiced + $order->shipping_invoiced + $order->tax_amount_invoiced - $order->discount_amount_invoiced;
$order->base_grand_total_invoiced = $order->base_sub_total_invoiced + $order->base_shipping_invoiced + $order->base_tax_amount_invoiced - $order->base_discount_amount_invoiced;
```

#### Formula
```
Total Paid = Invoiced Sub Total + Invoiced Shipping + Invoiced Tax - Invoiced Discount
```

#### Database Field
- `marketplace_orders.base_grand_total_invoiced`

---

### 7. **Total Refund** (Line 383-384)

```php
Display: "Total Refund"
Value: $sellerOrder->base_grand_total_refunded
```

#### Calculation Logic
This value is calculated when credit memos are created for refunds.

#### Database Field
- `marketplace_orders.base_grand_total_refunded`

---

### 8. **Admin Commission** (Line 387-388)

```php
Display: "Admin Commission"
Value: $sellerOrder->base_commission
```

#### Calculation Logic (from OrderRepository.php line 110-116, 273-274)
```php
// From each seller order item
if (isset($sellerCommissionPercentage)) {
    $commission = ($total * $sellerCommissionPercentage) / 100;
    $baseCommission = ($baseTotal * $sellerCommissionPercentage) / 100;
}

// Summed at order level
$order->commission = sum of all items: sellerOrderItem->commission
$order->base_commission = sum of all items: sellerOrderItem->base_commission
```

#### Commission Percentage Source
```php
$commissionPercentage = $seller->commission_enable
    ? $seller->commission_percentage
    : core()->getConfigData('marketplace.settings.general.admin_commission_percentage')
```

#### Formula
```
Admin Commission = (Base Amount × Commission Percentage) / 100
```

#### Database Fields
- `marketplace_orders.base_commission`
- `marketplace_orders.commission_percentage`

---

### 9. **Seller Total** (Line 391-392)

```php
Display: "Seller Total"
Value: $sellerOrder->base_seller_total
```

#### Calculation Logic (from OrderRepository.php line 114-115, 281-282)
```php
// From each seller order item
$sellerTotal = $item->total - $commission
$baseSellerTotal = $item->base_total - $baseCommission

// Minus all cost commissions if costing bifurcation is enabled
foreach (Cost::values() as $cost) {
    $baseSellerTotal -= $costCommission
}

// Summed at order level
$order->seller_total = sum of all items: sellerOrderItem->seller_total
$order->base_seller_total = sum of all items: sellerOrderItem->base_seller_total
```

#### Formula
```
Seller Total = Sub Total 
             - Admin Commission 
             - Buyer App Commission (if enabled)
             - Seller App Commission (if enabled)
             - TDS Commission (if enabled)
             - TCS Commission (if enabled)
             - Packing Cost Commission (if enabled)
             - Domestic Shipping Cost Commission (if enabled)
```

#### Database Field
- `marketplace_orders.base_seller_total`

---

### 10. **Cost-Based Commissions** (Lines 315-320, 395-410)

These fields are **conditionally displayed** based on the costing bifurcation mode.

#### Visibility Condition
```php
$orderHelper = app(\Webkul\ONDCSeller\Helpers\Order::class);

if ($orderHelper->shouldDisplayInSettlement()) {
    // Display these commissions
}

// shouldDisplayInSettlement() returns true when:
// - Mode is 'display_only' OR
// - Mode is 'full'
```

#### Configuration
**Config Path**: `ondc_seller.costing_bifurcation.settings.mode`

**Values**:
- `disabled` - Don't show commissions
- `display_only` - Show in settlement but don't affect pricing
- `full` - Show and apply to pricing

---

#### 10.1 **Buyer App Commission**

```php
Display: "Buyer App"
Value: $sellerOrder->base_buyer_app_commission (or calculated on-the-fly)
```

##### Calculation Logic (from Order.php line 55-82)

**If Mode = 'display_only'** (calculated on-the-fly):
```php
$percentage = config('ondc_seller.costing_bifurcation.settings.buyer_app_commission_percentage');
$gstPercentage = config('ondc_seller.costing_bifurcation.settings.gst_percentage');
$baseAmount = $sellerOrder->base_sub_total;

// Calculate commission without GST
$commissionWithoutGst = ($baseAmount × $percentage) / 100

// Add GST to commission (Buyer App includes GST)
$commission = $commissionWithoutGst + ($commissionWithoutGst × $gstPercentage / 100)
```

**If Mode = 'full'** (stored in database):
```php
// Retrieved from database field
$commission = $sellerOrder->base_buyer_app_commission
```

##### Formula
```
Commission Without GST = (Sub Total × Buyer App %) / 100
Commission With GST = Commission Without GST × (1 + GST% / 100)
```

##### Database Fields
- `marketplace_orders.base_buyer_app_commission`
- `marketplace_orders.buyer_app_commission_percentage`

---

#### 10.2 **Seller App Commission**

```php
Display: "Seller App"
Value: $sellerOrder->base_seller_app_commission (or calculated)
```

##### Calculation Logic
Same as Buyer App Commission, but uses:
```php
$percentage = config('ondc_seller.costing_bifurcation.settings.seller_app_commission_percentage');
```

##### Formula
```
Commission Without GST = (Sub Total × Seller App %) / 100
Commission With GST = Commission Without GST × (1 + GST% / 100)
```

##### Database Fields
- `marketplace_orders.base_seller_app_commission`
- `marketplace_orders.seller_app_commission_percentage`

---

#### 10.3 **TDS Commission**

```php
Display: "TDS"
Value: $sellerOrder->base_tds_commission (or calculated)
```

##### Calculation Logic (from Order.php line 72-74)
```php
$percentage = config('ondc_seller.costing_bifurcation.settings.tds_commission_percentage');
$baseAmount = $sellerOrder->base_sub_total;

// TDS does NOT include GST
$commission = ($baseAmount × $percentage) / 100
```

##### Formula
```
TDS Commission = (Sub Total × TDS %) / 100
(No GST applied)
```

##### Database Fields
- `marketplace_orders.base_tds_commission`
- `marketplace_orders.tds_commission_percentage`

---

#### 10.4 **TCS Commission**

```php
Display: "TCS"
Value: $sellerOrder->base_tcs_commission (or calculated)
```

##### Calculation Logic
```php
$percentage = config('ondc_seller.costing_bifurcation.settings.tcs_commission_percentage');
$baseAmount = $sellerOrder->base_sub_total;

// TCS does NOT include GST
$commission = ($baseAmount × $percentage) / 100
```

##### Formula
```
TCS Commission = (Sub Total × TCS %) / 100
(No GST applied)
```

##### Database Fields
- `marketplace_orders.base_tcs_commission`
- `marketplace_orders.tcs_commission_percentage`

---

#### 10.5 **Packing Cost Commission**

```php
Display: "Packing Cost"
Value: $sellerOrder->base_packing_cost_commission (or calculated)
```

##### Calculation Logic (from Order.php line 66-68)
```php
$percentage = config('ondc_seller.costing_bifurcation.settings.packing_cost_commission_percentage');
$gstPercentage = config('ondc_seller.costing_bifurcation.settings.gst_percentage');

// SPECIAL: Packing cost is a fixed amount, not a percentage
$commissionWithoutGst = $percentage  // This is actually the fixed amount

// Add GST (Packing Cost includes GST)
$commission = $commissionWithoutGst + ($commissionWithoutGst × $gstPercentage / 100)
```

##### Formula
```
Commission Without GST = Fixed Packing Cost Amount (from config)
Commission With GST = Fixed Amount × (1 + GST% / 100)
```

##### Database Fields
- `marketplace_orders.base_packing_cost_commission`
- `marketplace_orders.packing_cost_commission_percentage`

---

#### 10.6 **Domestic Shipping Cost Commission**

```php
Display: "Domestic Shipping Cost"
Value: $sellerOrder->base_domestic_shipping_cost_commission (or calculated)
```

##### Calculation Logic
```php
$percentage = config('ondc_seller.costing_bifurcation.settings.domestic_shipping_cost_commission_percentage');
$gstPercentage = config('ondc_seller.costing_bifurcation.settings.gst_percentage');

// SPECIAL: Domestic shipping is a fixed amount, not a percentage
$commissionWithoutGst = $percentage  // This is actually the fixed amount

// Add GST (Domestic Shipping includes GST)
$commission = $commissionWithoutGst + ($commissionWithoutGst × $gstPercentage / 100)
```

##### Formula
```
Commission Without GST = Fixed Domestic Shipping Amount (from config)
Commission With GST = Fixed Amount × (1 + GST% / 100)
```

##### Database Fields
- `marketplace_orders.base_domestic_shipping_cost_commission`
- `marketplace_orders.domestic_shipping_cost_commission_percentage`

---

### 11. **Total Due** (Line 323-325, 412-420)

```php
Display: "Total Due"

// If order is not canceled:
Value: $sellerOrder->base_total_due

// If order is canceled:
Value: 0.00
```

#### Calculation Logic
The `base_total_due` field represents the remaining amount to be paid or collected.

#### Formula (Inferred)
```
Total Due = Grand Total - Total Paid - Total Refunded
```

#### Database Field
- `marketplace_orders.base_total_due`

---

## Database Tables Used

### Primary Table: `marketplace_orders`
Contains all financial calculations for the seller order:
- All subtotal, grand total, and invoice amounts
- All commission fields (admin, buyer app, seller app, TDS, TCS, etc.)
- Seller total and total due
- Tax, discount, and shipping amounts

### Related Tables:
- `orders` - Main order information
- `order_items` - Individual product items
- `marketplace_order_items` - Seller-specific item data
- `invoices` - Invoice records for calculating "Total Paid"
- `order_confirm_requests` - ONDC settlement tracking

---

## Cost Enum Values

**File**: `packages/Webkul/ONDCSeller/src/Enums/Cost.php`

```php
enum Cost: string
{
    case BUYER_APP = 'buyer_app';
    case SELLER_APP = 'seller_app';
    case TDS = 'tds';
    case TCS = 'tcs';
    case PACKING_COST = 'packing_cost';
    case DOMESTIC_SHIPPING_COST = 'domestic_shipping_cost';
}
```

---

## Configuration Paths

### Costing Bifurcation Settings
- `ondc_seller.costing_bifurcation.settings.mode` - Display mode
- `ondc_seller.costing_bifurcation.settings.gst_percentage` - GST %
- `ondc_seller.costing_bifurcation.settings.buyer_app_commission_percentage`
- `ondc_seller.costing_bifurcation.settings.seller_app_commission_percentage`
- `ondc_seller.costing_bifurcation.settings.tds_commission_percentage`
- `ondc_seller.costing_bifurcation.settings.tcs_commission_percentage`
- `ondc_seller.costing_bifurcation.settings.packing_cost_commission_percentage`
- `ondc_seller.costing_bifurcation.settings.domestic_shipping_cost_commission_percentage`

### Tax Display Settings
- `sales.taxes.sales.display_subtotal` - How to show subtotal (excluding/including/both)
- `sales.taxes.sales.display_shipping_amount` - How to show shipping (excluding/including/both)
- `sales.taxes.sales.display_prices` - How to show prices

### Admin Commission
- `marketplace.settings.general.admin_commission_percentage`

---

## Example Calculation (Based on Order #1696)

### Database Values
```sql
base_sub_total = 986.39
base_tax_amount = 0.00
base_discount_amount = 0.00
base_shipping_amount = 0.00
commission_percentage = 20.00
base_commission = 197.28
base_grand_total = 986.39
base_grand_total_invoiced = 986.39
base_grand_total_refunded = 0.00
base_seller_total = 789.11
```

### Calculations

**Grand Total:**
```
986.39 (sub total) + 0 (tax) - 0 (discount) + 0 (shipping) = 986.39
```

**Admin Commission:**
```
986.39 × 20% = 197.28
```

**Seller Total:**
```
986.39 (grand total) - 197.28 (commission) = 789.11
```

**Total Paid:**
```
986.39 (invoiced amount)
```

**Total Due:**
```
986.39 (grand total) - 986.39 (paid) - 0 (refunded) = 0.00
```

---

## Summary Flow

```
┌─────────────────────────────────────────────────┐
│         Order Items Calculation                  │
│  (Sum all items: price, tax, discount)          │
└──────────────────┬──────────────────────────────┘
                   ↓
┌─────────────────────────────────────────────────┐
│              Sub Total                           │
│         (Sum of all item totals)                │
└──────────────────┬──────────────────────────────┘
                   ↓
┌─────────────────────────────────────────────────┐
│         Add Tax, Shipping                        │
│         Subtract Discount                        │
└──────────────────┬──────────────────────────────┘
                   ↓
┌─────────────────────────────────────────────────┐
│            Grand Total                           │
└──────────────────┬──────────────────────────────┘
                   ↓
┌─────────────────────────────────────────────────┐
│         Calculate Commissions:                   │
│  - Admin Commission (% based)                    │
│  - Buyer App Commission (% + GST)               │
│  - Seller App Commission (% + GST)              │
│  - TDS (% only, no GST)                         │
│  - TCS (% only, no GST)                         │
│  - Packing Cost (fixed + GST)                   │
│  - Domestic Shipping (fixed + GST)              │
└──────────────────┬──────────────────────────────┘
                   ↓
┌─────────────────────────────────────────────────┐
│            Seller Total                          │
│  = Grand Total - All Commissions                │
└──────────────────┬──────────────────────────────┘
                   ↓
┌─────────────────────────────────────────────────┐
│         Invoice Created                          │
│         (Total Paid calculated)                  │
└──────────────────┬──────────────────────────────┘
                   ↓
┌─────────────────────────────────────────────────┐
│            Total Due                             │
│  = Grand Total - Total Paid - Refunds           │
└─────────────────────────────────────────────────┘
```

---

## Query to Get Settlement Data

```sql
SELECT 
    mo.id,
    mo.order_id,
    mo.base_sub_total,
    mo.base_sub_total_incl_tax,
    mo.base_shipping_amount,
    mo.base_shipping_amount_incl_tax,
    mo.base_tax_amount,
    mo.base_discount_amount,
    mo.base_grand_total,
    mo.base_grand_total_invoiced,
    mo.base_grand_total_refunded,
    mo.commission_percentage,
    mo.base_commission,
    mo.base_seller_total,
    mo.buyer_app_commission_percentage,
    mo.base_buyer_app_commission,
    mo.seller_app_commission_percentage,
    mo.base_seller_app_commission,
    mo.tds_commission_percentage,
    mo.base_tds_commission,
    mo.tcs_commission_percentage,
    mo.base_tcs_commission,
    mo.packing_cost_commission_percentage,
    mo.base_packing_cost_commission,
    mo.domestic_shipping_cost_commission_percentage,
    mo.base_domestic_shipping_cost_commission,
    mo.base_total_due,
    mo.status,
    o.increment_id as order_number,
    ocr.is_settled
FROM marketplace_orders mo
JOIN orders o ON mo.order_id = o.id
LEFT JOIN order_confirm_requests ocr ON o.id = ocr.order_id
WHERE mo.id = 1680  -- Replace with the marketplace_orders.id
LIMIT 1;
```

---

## Key Helper Classes

### 1. `Webkul\ONDCSeller\Helpers\Order`
- `shouldDisplayInSettlement()` - Check if cost commissions should be shown
- `shouldCalculateOnOrders()` - Check if costs affect pricing
- `calculateCommissionForDisplay()` - Calculate commissions on-the-fly
- `calculateCommission()` - Calculate and store commissions

### 2. `Webkul\ONDCSeller\Repositories\Seller\OrderRepository`
- `collectTotals()` - Main method that calculates all order totals

### 3. `Webkul\ONDCAPI\Helpers\Settlement`
- `calculateTenantAmount()` - Calculate platform's share
- Handles ONDC settlement protocol

---

## Important Notes

1. **Cost Bifurcation Modes:**
   - `disabled`: No cost commissions shown or calculated
   - `display_only`: Commissions shown in settlements but don't affect product pricing
   - `full`: Commissions affect product pricing and are stored in database

2. **GST Application:**
   - Applied to: Buyer App, Seller App, Packing Cost, Domestic Shipping
   - NOT applied to: TDS, TCS

3. **Fixed vs Percentage:**
   - Most commissions are percentage-based on subtotal
   - Packing Cost and Domestic Shipping are fixed amounts (despite field name having "percentage")

4. **Display Only Mode:**
   - Calculates commissions on-the-fly using `calculateCommissionForDisplay()`
   - Uses `base_sub_total` as the base amount for calculation
   - Falls back to stored database values if available

5. **Order Status:**
   - If order is canceled, Total Due shows as 0.00

---

## File References

### View Files
- `packages/Webkul/ONDCSeller/src/Resources/views/admin/tenant/settlements/view.blade.php`

### Controllers
- `packages/Webkul/ONDCSeller/src/Http/Controllers/Admin/SettlementController.php`

### Helpers
- `packages/Webkul/ONDCSeller/src/Helpers/Order.php`
- `packages/Webkul/ONDCAPI/Helpers/Settlement.php`

### Repositories
- `packages/Webkul/ONDCSeller/src/Repositories/Seller/OrderRepository.php`

### Enums
- `packages/Webkul/ONDCSeller/src/Enums/Cost.php`

### Config
- `packages/Webkul/ONDCSeller/src/Config/system.php`

