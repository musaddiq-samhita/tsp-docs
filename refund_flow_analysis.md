# Refund Flow Analysis - SmartSell Application

## Overview
This document provides a comprehensive analysis of how the refund system is implemented in the SmartSell application (based on Bagisto framework with Marketplace extensions).

---

## 1. Refunds List Page Logic

### URL
`https://smartsell.samhita.org/admin/sales/refunds`

### Route Definition
**File:** `packages/Webkul/Admin/src/Routes/sales-routes.php:56`
```php
Route::get('', 'index')->name('admin.sales.refunds.index');
```

### Controller
**File:** `packages/Webkul/Admin/src/Http/Controllers/Sales/RefundController.php`

**Method:** `index()`
```php
public function index()
{
    if (request()->ajax()) {
        return datagrid(OrderRefundDataGrid::class)->process();
    }
    
    return view('admin::sales.refunds.index');
}
```

### Data Grid Implementation

**File:** `packages/Webkul/Admin/src/DataGrids/Sales/OrderRefundDataGrid.php`

#### Query Builder Logic
The data grid populates the refunds list using the following SQL query structure:

```sql
SELECT 
    refunds.id,
    orders.increment_id,
    refunds.state,
    refunds.base_grand_total,
    refunds.created_at,
    CONCAT(order_address_billing.first_name, " ", order_address_billing.last_name) as billed_to
FROM refunds
LEFT JOIN orders ON refunds.order_id = orders.id
LEFT JOIN marketplace_orders ON orders.id = marketplace_orders.order_id
LEFT JOIN marketplace_sellers ON marketplace_orders.marketplace_seller_id = marketplace_sellers.id
LEFT JOIN addresses as order_address_billing 
    ON order_address_billing.order_id = orders.id 
    AND order_address_billing.address_type = 'billing'
WHERE marketplace_sellers.id IN (allowed_seller_ids) -- Seller filter applied if configured
```

#### Columns Displayed
1. **ID** - Refund ID
2. **Order ID** - Original order increment ID
3. **Refunded Amount** - Base grand total (formatted)
4. **Billed To** - Customer name from billing address
5. **Refund Date** - Created timestamp

#### Seller Filtering
The trait `SellerFilter` is applied via `applySellerFilter()` method which filters refunds based on:
- Allowed seller IDs from `BagistoCustom::getAllowedSellerIds()`
- If `null`, shows all refunds
- Otherwise filters by `marketplace_sellers.id`

---

## 2. Database Schema

### Core Tables

#### `refunds` Table
Main refund table storing refund transactions:

| Column | Type | Description |
|--------|------|-------------|
| id | int unsigned | Primary key |
| company_id | int unsigned | Company/Tenant ID |
| order_id | int unsigned | Reference to orders table |
| increment_id | varchar(191) | Human-readable refund ID (nullable) |
| state | varchar(191) | Refund state (e.g., 'refunded') |
| email_sent | tinyint(1) | Email notification status |
| total_qty | int | Total quantity refunded |
| base_currency_code | varchar(191) | Base currency |
| channel_currency_code | varchar(191) | Channel currency |
| order_currency_code | varchar(191) | Order currency |
| adjustment_refund | decimal(12,4) | Additional refund adjustment |
| base_adjustment_refund | decimal(12,4) | Base currency adjustment |
| adjustment_fee | decimal(12,4) | Adjustment fee deducted |
| base_adjustment_fee | decimal(12,4) | Base currency fee |
| sub_total | decimal(12,4) | Subtotal amount |
| base_sub_total | decimal(12,4) | Base subtotal |
| grand_total | decimal(12,4) | Total refund amount |
| base_grand_total | decimal(12,4) | Base grand total |
| shipping_amount | decimal(12,4) | Shipping refund |
| base_shipping_amount | decimal(12,4) | Base shipping refund |
| tax_amount | decimal(12,4) | Tax refunded |
| base_tax_amount | decimal(12,4) | Base tax refunded |
| discount_amount | decimal(12,4) | Discount amount |
| base_discount_amount | decimal(12,4) | Base discount |
| *_incl_tax fields | decimal(12,4) | Amounts including tax |

#### `refund_items` Table
Individual items within each refund:

| Column | Type | Description |
|--------|------|-------------|
| id | int unsigned | Primary key |
| refund_id | int unsigned | FK to refunds table |
| parent_id | int unsigned | Parent refund item (for composite products) |
| order_item_id | int unsigned | FK to order_items table |
| product_id | int unsigned | Product ID |
| product_type | varchar(191) | Product type class |
| name | varchar(191) | Product name |
| sku | varchar(191) | Product SKU |
| qty | int | Quantity refunded |
| price | decimal(12,4) | Unit price |
| base_price | decimal(12,4) | Base unit price |
| total | decimal(12,4) | Total item amount |
| base_total | decimal(12,4) | Base total |
| tax_amount | decimal(12,4) | Item tax |
| discount_amount | decimal(12,4) | Item discount |
| additional | json | Additional item data |

#### `marketplace_refunds` Table
Marketplace-specific refund tracking:

| Column | Type | Description |
|--------|------|-------------|
| id | int unsigned | Primary key |
| refund_id | int unsigned | FK to core refunds table |
| marketplace_order_id | int unsigned | FK to marketplace_orders |
| company_id | int unsigned | Company ID |
| state | varchar(191) | Refund state |
| total_qty | int | Total quantity |
| sub_total, grand_total, etc. | decimal(12,4) | Financial amounts per seller |

#### `marketplace_refund_items` Table
Links marketplace refunds to refund items:

| Column | Type | Description |
|--------|------|-------------|
| id | int unsigned | Primary key |
| marketplace_refund_id | int unsigned | FK to marketplace_refunds |
| refund_item_id | int unsigned | FK to refund_items |
| company_id | int unsigned | Company ID |

---

## 3. Refund Creation Flow

### Entry Point: Order View Page

**File:** `resources/admin-themes/default/views/sales/orders/view.blade.php`

Refund button is displayed when:
```php
@if (
    $order->canRefund()
    && bouncer()->hasPermission('sales.refunds.create')
)
    @include('admin::sales.refunds.create')
@endif
```

### Order Can Refund Check

**File:** `packages/Webkul/Sales/src/Models/Order.php:366`

```php
public function canRefund(): bool
{
    foreach ($this->items as $item) {
        if (
            $item->qty_to_refund > 0
            && ! in_array($item->order->status, [
                self::STATUS_CLOSED,
                self::STATUS_FRAUD,
            ])
        ) {
            return true;
        }
    }
    return false;
}
```

**Conditions:**
- At least one order item has `qty_to_refund > 0`
- Order status is NOT 'closed' or 'fraud'
- `qty_to_refund` is calculated as: `qty_invoiced - qty_refunded`

### Refund Creation Form

**File:** `packages/Webkul/Admin/src/Resources/views/sales/refunds/create.blade.php`

**Vue Component:** `v-create-refund`

**Features:**
1. Lists all order items with `qty_to_refund > 0`
2. For each item displays:
   - Product image
   - Product name
   - SKU
   - Unit price
   - Order quantities (ordered, invoiced, shipped, refunded, canceled)
   - Input field for quantity to refund
3. Shipping refund amount
4. Adjustment refund (additional amount to refund)
5. Adjustment fee (amount to deduct from refund)
6. Real-time totals calculation via AJAX

**Form Submission:**
- **Route:** `admin.sales.refunds.store`
- **Method:** POST
- **URL:** `/admin/sales/refunds/create/{order_id}`

### Update Totals (AJAX)

**Route:** `admin.sales.refunds.update_totals`
**Method:** POST
**Controller Method:** `RefundController::updateTotals()`

Calculates refund totals without creating the refund:
```php
public function updateTotals(int $orderId)
{
    try {
        $data = $this->refundRepository->getOrderItemsRefundSummary(
            request()->input(), 
            $orderId
        );
    } catch (\Exception $e) {
        return response()->json(['message' => $e->getMessage()], 400);
    }
    
    return response()->json($data);
}
```

Returns:
```json
{
    "subtotal": {"price": 100.00, "formatted_price": "₹100.00"},
    "discount": {"price": 0.00, "formatted_price": "₹0.00"},
    "tax": {"price": 18.00, "formatted_price": "₹18.00"},
    "shipping": {"price": 20.00, "formatted_price": "₹20.00"},
    "grand_total": {"price": 138.00, "formatted_price": "₹138.00"}
}
```

---

## 4. Refund Processing Logic

### Controller Store Method

**File:** `packages/Webkul/Admin/src/Http/Controllers/Sales/RefundController.php:55`

```php
public function store(int $orderId)
{
    $order = $this->orderRepository->findOrFail($orderId);
    
    // Validation: Can order be refunded?
    if (!$order->canRefund()) {
        session()->flash('error', trans('admin::app.sales.refunds.create.creation-error'));
        return redirect()->back();
    }
    
    // Validate input
    $this->validate(request(), [
        'refund.items'   => 'array',
        'refund.items.*' => 'required|numeric|min:0',
    ]);
    
    $data = request()->all();
    
    // Set default shipping to 0 if not provided
    if (!isset($data['refund']['shipping'])) {
        $data['refund']['shipping'] = 0;
    }
    
    // Calculate totals
    $totals = $this->refundRepository->getOrderItemsRefundSummary(
        $data['refund'], 
        $orderId
    );
    
    if (!$totals) {
        session()->flash('error', trans('admin::app.sales.refunds.create.invalid-qty'));
        return redirect()->route('admin.sales.refunds.index');
    }
    
    // Validate refund amount doesn't exceed maximum allowed
    $maxRefundAmount = $totals['grand_total']['price'] 
        - $order->refunds()->sum('base_adjustment_refund');
    
    $refundAmount = $totals['grand_total']['price'] 
        - $totals['shipping']['price'] 
        + $data['refund']['shipping'] 
        + $data['refund']['adjustment_refund'] 
        - $data['refund']['adjustment_fee'];
    
    if (!$refundAmount) {
        session()->flash('error', trans('admin::app.sales.refunds.create.invalid-refund-amount-error'));
        return redirect()->back();
    }
    
    if ($refundAmount > $maxRefundAmount) {
        session()->flash('error', trans('admin::app.sales.refunds.create.refund-limit-error', [
            'amount' => core()->formatBasePrice($maxRefundAmount),
        ]));
        return redirect()->back();
    }
    
    // Create refund
    $this->refundRepository->create(array_merge($data, ['order_id' => $orderId]));
    
    session()->flash('success', trans('admin::app.sales.refunds.create.create-success'));
    return redirect()->route('admin.sales.refunds.index');
}
```

### Repository Create Method

**File:** `packages/Webkul/Sales/src/Repositories/RefundRepository.php:38`

**Transaction Flow:**

```php
public function create(array $data)
{
    DB::beginTransaction();
    
    try {
        // 1. Dispatch before event
        Event::dispatch('sales.refund.save.before', $data);
        
        // 2. Get order
        $order = $this->orderRepository->find($data['order_id']);
        
        // 3. Calculate total quantity
        $totalQty = array_sum($data['refund']['items'] ?? []) ?? 0;
        
        // 4. Create main refund record
        $refund = parent::create([
            'order_id'               => $order->id,
            'total_qty'              => $totalQty,
            'state'                  => 'refunded',
            'base_currency_code'     => $order->base_currency_code,
            'channel_currency_code'  => $order->channel_currency_code,
            'order_currency_code'    => $order->order_currency_code,
            'adjustment_refund'      => core()->convertPrice($data['refund']['adjustment_refund'], $order->order_currency_code),
            'base_adjustment_refund' => $data['refund']['adjustment_refund'],
            'adjustment_fee'         => core()->convertPrice($data['refund']['adjustment_fee'], $order->order_currency_code),
            'base_adjustment_fee'    => $data['refund']['adjustment_fee'],
            'shipping_amount'        => core()->convertPrice($data['refund']['shipping'], $order->order_currency_code),
            'base_shipping_amount'   => $data['refund']['shipping'],
        ]);
        
        // 5. Create refund items
        foreach ($data['refund']['items'] ?? [] as $itemId => $qty) {
            if (!$qty) continue;
            
            $orderItem = $this->orderItemRepository->find($itemId);
            
            // Validate qty doesn't exceed qty_to_refund
            if ($qty > $orderItem->qty_to_refund) {
                $qty = $orderItem->qty_to_refund;
            }
            
            // Calculate pro-rated tax
            $taxAmount = (($orderItem->tax_amount / $orderItem->qty_ordered) * $qty);
            $baseTaxAmount = (($orderItem->base_tax_amount / $orderItem->qty_ordered) * $qty);
            
            // Create refund item
            $refundItem = $this->refundItemRepository->create([
                'refund_id'            => $refund->id,
                'order_item_id'        => $orderItem->id,
                'name'                 => $orderItem->name,
                'sku'                  => $orderItem->sku,
                'qty'                  => $qty,
                'price'                => $orderItem->price,
                'price_incl_tax'       => $orderItem->price_incl_tax,
                'base_price'           => $orderItem->base_price,
                'base_price_incl_tax'  => $orderItem->base_price_incl_tax,
                'total'                => $orderItem->price * $qty,
                'total_incl_tax'       => ($orderItem->price * $qty) + $taxAmount,
                'base_total'           => $orderItem->base_price * $qty,
                'base_total_incl_tax'  => ($orderItem->base_price * $qty) + $baseTaxAmount,
                'tax_amount'           => $taxAmount,
                'base_tax_amount'      => $baseTaxAmount,
                'discount_amount'      => (($orderItem->discount_amount / $orderItem->qty_ordered) * $qty),
                'base_discount_amount' => (($orderItem->base_discount_amount / $orderItem->qty_ordered) * $qty),
                'product_id'           => $orderItem->product_id,
                'product_type'         => $orderItem->product_type,
                'additional'           => $orderItem->additional,
            ]);
            
            // Handle composite products (bundles, configurables)
            if ($orderItem->getTypeInstance()->isComposite()) {
                foreach ($orderItem->children as $childOrderItem) {
                    $finalQty = $childOrderItem->qty_ordered
                        ? ($childOrderItem->qty_ordered / $orderItem->qty_ordered) * $qty
                        : $orderItem->qty_ordered;
                    
                    // Create child refund item
                    $refundItem->child = $this->refundItemRepository->create([
                        'refund_id'            => $refund->id,
                        'order_item_id'        => $childOrderItem->id,
                        'parent_id'            => $refundItem->id,
                        'name'                 => $childOrderItem->name,
                        'sku'                  => $childOrderItem->sku,
                        'qty'                  => $finalQty,
                        'price'                => $childOrderItem->price,
                        'base_price'           => $childOrderItem->base_price,
                        'total'                => $childOrderItem->price * $finalQty,
                        'base_total'           => $childOrderItem->base_price * $finalQty,
                        'tax_amount'           => 0,
                        'base_tax_amount'      => 0,
                        'discount_amount'      => 0,
                        'base_discount_amount' => 0,
                        'product_id'           => $childOrderItem->product_id,
                        'product_type'         => $childOrderItem->product_type,
                        'additional'           => $childOrderItem->additional,
                    ]);
                    
                    // Return inventory for child items
                    if ($childOrderItem->getTypeInstance()->isStockable() 
                        || $childOrderItem->getTypeInstance()->showQuantityBox()) {
                        $this->refundItemRepository->returnQtyToProductInventory(
                            $childOrderItem, 
                            $finalQty
                        );
                    }
                    
                    // Update child order item totals
                    $this->orderItemRepository->collectTotals($childOrderItem);
                }
            } else {
                // Return inventory for simple products
                if ($orderItem->getTypeInstance()->isStockable() 
                    || $orderItem->getTypeInstance()->showQuantityBox()) {
                    $this->refundItemRepository->returnQtyToProductInventory(
                        $orderItem, 
                        $qty
                    );
                }
            }
            
            // Update order item totals
            $this->orderItemRepository->collectTotals($orderItem);
            
            // Expire downloadable links if fully refunded
            if ($orderItem->qty_ordered == $orderItem->qty_refunded + $orderItem->qty_canceled) {
                $this->downloadableLinkPurchasedRepository->updateStatus($orderItem, 'expired');
            }
        }
        
        // 6. Collect refund totals
        $this->collectTotals($refund);
        
        // 7. Update order totals
        $this->orderRepository->collectTotals($order);
        
        // 8. Update order status
        $this->orderRepository->updateOrderStatus($order);
        
        // 9. Dispatch after event
        Event::dispatch('sales.refund.save.after', $refund);
        
        DB::commit();
    } catch (\Exception $e) {
        DB::rollBack();
        throw $e;
    }
    
    return $refund;
}
```

### Collect Totals Method

**File:** `packages/Webkul/Sales/src/Repositories/RefundRepository.php:178`

```php
public function collectTotals($refund)
{
    // Initialize totals
    $refund->sub_total = $refund->base_sub_total = 0;
    $refund->sub_total_incl_tax = $refund->base_sub_total_incl_tax = 0;
    $refund->tax_amount = $refund->base_tax_amount = 0;
    $refund->shipping_tax_amount = $refund->base_shipping_tax_amount = 0;
    $refund->discount_amount = $refund->base_discount_amount = 0;
    
    // Sum item totals
    foreach ($refund->items as $item) {
        $refund->tax_amount += $item->tax_amount;
        $refund->base_tax_amount += $item->base_tax_amount;
        $refund->discount_amount += $item->discount_amount;
        $refund->base_discount_amount += $item->base_discount_amount;
        $refund->sub_total += $item->total;
        $refund->base_sub_total += $item->base_total;
        $refund->sub_total_incl_tax += $item->total_incl_tax;
        $refund->base_sub_total_incl_tax += $item->base_total_incl_tax;
    }
    
    // Calculate pro-rated shipping tax
    if ((float) $refund->order->shipping_invoiced) {
        $refund->shipping_tax_amount = 
            ($refund->order->shipping_tax_amount / $refund->order->shipping_invoiced) 
            * $refund->shipping_amount;
        $refund->base_shipping_tax_amount = 
            ($refund->order->base_shipping_tax_amount / $refund->order->base_shipping_invoiced) 
            * $refund->base_shipping_amount;
    }
    
    // Calculate shipping with tax
    $refund->shipping_amount_incl_tax = 
        $refund->shipping_amount + $refund->shipping_tax_amount;
    $refund->base_shipping_amount_incl_tax = 
        $refund->base_shipping_amount + $refund->base_shipping_tax_amount;
    
    // Add shipping tax to total tax
    $refund->tax_amount += $refund->shipping_tax_amount;
    $refund->base_tax_amount += $refund->base_shipping_tax_amount;
    
    // Calculate grand total
    $refund->grand_total = $refund->sub_total 
        + $refund->tax_amount 
        + $refund->shipping_amount 
        + $refund->adjustment_refund 
        - $refund->adjustment_fee 
        - $refund->discount_amount;
    
    $refund->base_grand_total = $refund->base_sub_total 
        + $refund->base_tax_amount 
        + $refund->base_shipping_amount 
        + $refund->base_adjustment_refund 
        - $refund->base_adjustment_fee 
        - $refund->base_discount_amount;
    
    $refund->save();
    
    return $refund;
}
```

---

## 5. Event-Driven Architecture

### Events Dispatched

**1. Before Refund Creation**
```php
Event::dispatch('sales.refund.save.before', $data);
```

**2. After Refund Creation**
```php
Event::dispatch('sales.refund.save.after', $refund);
```

### Event Listeners

#### Admin Event Listeners

**File:** `packages/Webkul/Admin/src/Providers/EventServiceProvider.php`

```php
'sales.refund.save.after' => [
    'Webkul\Admin\Listeners\Refund@afterCreated',
],
```

**Listener:** `Webkul\Admin\Listeners\Refund::afterCreated()`

**Actions:**
1. **PayPal Refund Processing** (if applicable)
   - Checks if payment method is `paypal_smart_button`
   - Gets PayPal order ID from payment additional data
   - Retrieves capture ID
   - Processes refund via PayPal API
   
2. **Email Notification**
   - Checks if refund email notifications are enabled
   - Sends `RefundedNotification` email to customer
   - Email contains refund details

#### Marketplace Event Listeners

**File:** `packages/Webkul/Marketplace/src/Providers/EventServiceProvider.php`

```php
'sales.refund.save.after' => [
    'Webkul\Marketplace\Listeners\Refund@afterRefund',
],
```

**Listener:** `Webkul\Marketplace\Listeners\Refund::afterRefund()`

**Actions:**
1. Creates marketplace-specific refunds in `marketplace_refunds` table
2. For each refund item:
   - Identifies the seller (from item's seller_info or product)
   - Finds the marketplace order
   - Creates/updates marketplace refund record
   - Links refund items via `marketplace_refund_items`
   - Calculates seller-specific totals
   - Updates marketplace order totals
   - Updates marketplace order status
   - Sets seller payout status to 'refunded'
3. Dispatches `marketplace.sales.refund.save.after` event

**File:** `packages/Webkul/Marketplace/src/Repositories/RefundRepository.php:35`

**Key Logic:**
```php
public function create($refund)
{
    Event::dispatch('marketplace.sales.refund.save.before', $refund);
    
    $sellerRefunds = [];
    
    // Process each refund item
    foreach ($refund->items()->get() as $item) {
        // Get seller from item's seller_info or product
        if (isset($item->additional['seller_info']) 
            && !$item->additional['seller_info']['is_owner']) {
            $seller = $this->sellerRepository->find(
                $item->additional['seller_info']['seller_id']
            );
        } else {
            $seller = $this->productRepository->getSellerByProductId($item->product_id);
        }
        
        if (!$seller) continue;
        
        // Find marketplace order for this seller
        $sellerOrder = $this->orderRepository->findOneWhere([
            'order_id'              => $refund->order->id,
            'marketplace_seller_id' => $seller->id,
        ]);
        
        if (!$sellerOrder) continue;
        
        // Find marketplace order item
        $sellerOrderItem = $this->orderItemRepository->findOneWhere([
            'marketplace_order_id' => $sellerOrder->id,
            'order_item_id'        => $item->order_item->id,
        ]);
        
        if (!$sellerOrderItem) continue;
        
        // Create or update marketplace refund
        $sellerRefund = $this->findOneWhere([
            'refund_id'            => $refund->id,
            'marketplace_order_id' => $sellerOrder->id,
        ]);
        
        if (!$sellerRefund) {
            $sellerRefunds[] = $sellerRefund = parent::create([
                'total_qty'              => $item->qty,
                'state'                  => 'refunded',
                'refund_id'              => $refund->id,
                'marketplace_order_id'   => $sellerOrder->id,
                'adjustment_refund'      => core()->convertPrice($refund['adjustment_refund'], $sellerOrder->order_currency_code),
                'base_adjustment_refund' => $refund['adjustment_refund'],
                'adjustment_fee'         => core()->convertPrice($refund['adjustment_fee'], $sellerOrder->order_currency_code),
                'base_adjustment_fee'    => $refund['adjustment_fee'],
                'shipping_amount'        => core()->convertPrice($refund['shipping_amount'], $sellerOrder->order_currency_code),
                'base_shipping_amount'   => $refund['shipping_amount'],
            ]);
        } else {
            $sellerRefund->total_qty += $item->qty;
            $sellerRefund->save();
        }
        
        // Link refund item to marketplace refund
        $this->refundItemRepository->create([
            'marketplace_refund_id' => $sellerRefund->id,
            'refund_item_id'        => $item->id,
        ]);
        
        // Update seller order item totals
        $this->orderItemRepository->collectTotals($sellerOrderItem);
    }
    
    // Collect totals for all seller refunds
    foreach ($sellerRefunds as $sellerRefund) {
        $this->collectTotals($sellerRefund);
    }
    
    // Update seller orders
    foreach ($sellers ?? [] as $seller) {
        $orders = $this->orderRepository->findWhere([
            'order_id'              => $refund->order->id,
            'marketplace_seller_id' => $seller->id,
        ]);
        
        foreach ($orders as $order) {
            $this->orderRepository->collectTotals($order);
            $this->orderRepository->updateOrderStatus($order);
            $this->orderRepository->update(['seller_payout_status' => 'refunded'], $order->id);
        }
    }
    
    // Dispatch after events
    foreach ($sellerRefunds as $sellerRefund) {
        Event::dispatch('marketplace.sales.refund.save.after', $sellerRefund);
    }
}
```

---

## 6. Refund Calculation Logic

### Total Calculation Formula

```
Grand Total = Sub Total 
            + Tax Amount 
            + Shipping Amount 
            + Adjustment Refund 
            - Adjustment Fee 
            - Discount Amount
```

### Per-Item Calculations

**Item Tax:**
```
Item Tax = (Order Item Tax / Order Item Qty) × Refund Qty
```

**Item Discount:**
```
Item Discount = (Order Item Discount / Order Item Qty) × Refund Qty
```

**Shipping Tax:**
```
Shipping Tax = (Order Shipping Tax / Order Shipping Invoiced) × Refund Shipping
```

### Validations

1. **Quantity Validation:**
   - Refund quantity cannot exceed `qty_to_refund`
   - `qty_to_refund = qty_invoiced - qty_refunded`

2. **Amount Validation:**
   - Total refund amount must be > 0
   - Refund amount cannot exceed maximum refundable amount
   - Maximum refundable = Grand total - Sum of previous adjustment refunds

3. **Order Status Validation:**
   - Order status must NOT be 'closed' or 'fraud'

---

## 7. Inventory Management

### Inventory Return Logic

**File:** `packages/Webkul/Sales/src/Repositories/RefundItemRepository.php`

When a refund is created, inventory is returned to stock for:
- Simple products (if stockable)
- Child products of composite products (bundles, configurables)
- Products with quantity tracking enabled

**Method:** `returnQtyToProductInventory()`

**Process:**
1. Find product inventory record
2. Increase qty by refunded amount
3. Save inventory record

---

## 8. Database Queries - Sample Data

### Current Refunds in System

**Total Refunds:** 12

**Recent Refunds Sample:**

| ID | Order ID | State | Grand Total | Created Date |
|----|----------|-------|-------------|--------------|
| 12 | 1700 | refunded | ₹79.37 | 2025-11-13 |
| 11 | 1682 | refunded | ₹68.03 | 2025-11-12 |
| 10 | 1631 | refunded | ₹963.72 | 2025-11-11 |
| 9 | 1683 | refunded | ₹204.08 | 2025-11-06 |
| 8 | 1598 | refunded | ₹970.00 | 2025-11-04 |

### Sample Refund Data Structure

```json
{
  "id": 12,
  "order_id": 1700,
  "state": "refunded",
  "email_sent": 1,
  "total_qty": 1,
  "base_currency_code": "INR",
  "adjustment_refund": "0.0000",
  "adjustment_fee": "0.0000",
  "sub_total": "79.3651",
  "grand_total": "79.3651",
  "shipping_amount": "0.0000",
  "tax_amount": "0.0000",
  "discount_amount": "0.0000",
  "created_at": "2025-11-13T15:56:43.000Z"
}
```

---

## 9. Key Features & Observations

### Multi-Currency Support
- Supports base, channel, and order currencies
- Automatic currency conversion using `core()->convertPrice()`

### Multi-Seller/Marketplace Support
- Core refunds are mirrored in marketplace refunds
- Each seller gets their own refund record
- Commission and payout tracking per seller
- Seller payout status updated to 'refunded'

### Composite Product Handling
- Parent and child items tracked separately
- Inventory returned for child items
- Pro-rated quantities for child items

### Adjustment Fields
- **Adjustment Refund:** Additional amount to refund to customer
- **Adjustment Fee:** Amount to deduct from refund (e.g., restocking fee)

### Email Notifications
- Configurable via `emails.general.notifications.emails.general.notifications.new_refund`
- Uses `RefundedNotification` mailable class
- Email sent flag tracked in database

### Payment Gateway Integration
- PayPal Smart Button refunds processed automatically
- Other payment methods may require manual processing

### Seller Filtering
- Admins can be restricted to view only specific sellers' refunds
- Filter applied via `SellerFilter` trait
- Uses `BagistoCustom::getAllowedSellerIds()` for permission checking

---

## 10. API Endpoints Summary

| Method | URL | Controller Method | Purpose |
|--------|-----|-------------------|---------|
| GET | `/admin/sales/refunds` | `RefundController::index()` | List all refunds (DataGrid) |
| GET | `/admin/sales/refunds/view/{id}` | `RefundController::view()` | View refund details |
| POST | `/admin/sales/refunds/create/{order_id}` | `RefundController::store()` | Create new refund |
| POST | `/admin/sales/refunds/update-totals/{order_id}` | `RefundController::updateTotals()` | AJAX - Calculate refund totals |

---

## 11. Models & Relationships

### Refund Model

**File:** `packages/Webkul/Sales/src/Models/Refund.php`

**Relationships:**
- `order()` - BelongsTo Order
- `items()` - HasMany RefundItem (excluding child items)
- `customer()` - MorphTo Customer
- `channel()` - MorphTo Channel
- `address()` - BelongsTo OrderAddress

### RefundItem Model

**File:** `packages/Webkul/Sales/src/Models/RefundItem.php`

**Relationships:**
- `refund()` - BelongsTo Refund
- `orderItem()` - BelongsTo OrderItem
- `parent()` - BelongsTo RefundItem (for child items)
- `child()` - HasOne RefundItem (for parent items)

### Marketplace Refund Model

**File:** `packages/Webkul/Marketplace/src/Models/Refund.php`

**Relationships:**
- `refund()` - BelongsTo Core Refund
- `order()` - BelongsTo Marketplace Order
- `items()` - HasMany Marketplace RefundItem

---

## 12. Security & Permissions

### ACL Configuration

**File:** `packages/Webkul/Admin/src/Config/acl.php`

```php
[
    'key'   => 'sales.refunds',
    'name'  => 'admin::app.acl.refunds',
    'route' => 'admin.sales.refunds.index',
    'sort'  => 4,
], [
    'key'   => 'sales.refunds.view',
    'name'  => 'admin::app.acl.view',
    'route' => 'admin.sales.refunds.view',
    'sort'  => 1,
], [
    'key'   => 'sales.refunds.create',
    'name'  => 'admin::app.acl.create',
    'route' => 'admin.sales.refunds.store',
    'sort'  => 2,
]
```

### Permission Checks

```php
bouncer()->hasPermission('sales.refunds.view')
bouncer()->hasPermission('sales.refunds.create')
```

---

## Conclusion

The refund system is a comprehensive, event-driven architecture that:

1. **Integrates with Orders:** Only allows refunds for eligible orders with invoiced items
2. **Handles Complexity:** Supports composite products, multi-currency, multi-seller scenarios
3. **Maintains Data Integrity:** Uses database transactions, validates amounts and quantities
4. **Extensible:** Event system allows plugins to hook into refund lifecycle
5. **Marketplace-Aware:** Automatically creates seller-specific refund records
6. **Inventory Management:** Returns refunded items to stock automatically
7. **Payment Integration:** Processes refunds via payment gateways (PayPal)
8. **Audit Trail:** Tracks all refund activities with timestamps and email notifications

The list page at `https://smartsell.samhita.org/admin/sales/refunds` is populated by the `OrderRefundDataGrid` which joins refunds with orders, marketplace orders, and sellers to display a filtered, sortable list with pagination support.

