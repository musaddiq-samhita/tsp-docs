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

---

## ADDENDUM: ONDC Refund & Return Flow Analysis

### Database Analysis - BAP (Buyer App) Distribution

Based on the database inspection, here is the breakdown of orders by BAP (Buyer Application Platform):

**Total Orders:** 696
- **ONDC Orders:** 608 (87.4%)
- **Non-ONDC Orders:** 88 (12.6%)

**Refunds by Channel:**
- **All 12 refunds** are from "Samhita Marketplace" channel (non-ONDC orders)
- **Zero refunds** exist for ONDC orders

**BAP Distribution for ONDC Orders:**

| BAP ID | Order Count | Percentage |
|--------|-------------|------------|
| `prd.mystore.in` | 147 | 24.2% |
| `pramaan.ondc.org/beta/preprod/mock/buyer` | 132 | 21.7% |
| `shop.samhitabazaar.com` | 115 | 18.9% |
| `prod.nirmitbap.ondc.org` | 90 | 14.8% |
| `buyer-app-preprod-v2.ondc.org` | 67 | 11.0% |
| `buyer.easypay.co.in` | 45 | 7.4% |
| `buysimplynow-preprod.eazehub.com` | 25 | 4.1% |
| `cloud-adaptor.proteantech.in/adaptor-v2/csc-grameen` | 2 | 0.3% |

### ONDC Return & Refund Mechanism

ONDC (Open Network for Digital Commerce) handles returns and refunds differently from regular marketplace orders. The system uses two separate mechanisms:

#### 1. RMA (Return Merchandise Authorization) Flow

**Database Table:** `rma_requests`

**Current Status:**
- **Total RMA Requests:** 21
- **All requests status:** `Return_Delivered` (completed returns)

**RMA Request Schema:**

| Field | Type | Description |
|-------|------|-------------|
| id | bigint | Primary key |
| status | enum | Return lifecycle status |
| ondc_order_id | varchar(191) | ONDC order identifier |
| ondc_item_id | varchar(191) | ONDC item identifier (e.g., "ondc109") |
| qty | varchar(191) | Quantity to return |
| return_fulfillment_id | varchar(191) | Unique fulfillment ID for return |
| return_pickup_start_time_range | json | Pickup window start time |
| return_pickup_end_time_range | json | Pickup window end time |
| reason_id | varchar(191) | Return reason code |
| reason_desc | varchar(191) | Human-readable return reason |
| images | json | Supporting images for return |
| ttl_approval | varchar(191) | Time to live for approval (e.g., "PT24H") |
| ttl_reverseqc | varchar(191) | Time for reverse QC (e.g., "P3D") |
| order_update_request_id | bigint | FK to order_update_requests |
| return_order_id | varchar(191) | Shipper return order ID |
| return_shipment_id | varchar(191) | Shipper shipment ID |
| return_status | varchar(191) | Shipper return status |
| return_courier_id | varchar(191) | Courier partner ID |
| return_awb_code | varchar(191) | Air waybill code |

**RMA Status Values:**
- `Return_Initiated` - Return request created
- `Return_Approved` - Seller approved return
- `Return_Rejected` - Seller rejected return
- `Return_Picked` - Item picked up from customer
- `Return_Pick_Failed` - Pickup failed
- `Return_Failed` - Return shipment failed
- `Return_Delivered` - Item returned to seller
- `Liquidated` - Return completed and refund processed

**Sample RMA Request Data:**

```json
{
  "id": 16,
  "status": "Return_Delivered",
  "ondc_order_id": "SS1743999738194",
  "ondc_item_id": "ondc109",
  "qty": "1",
  "return_fulfillment_id": "5060fc40-ce13-444e-a6bf-d415fc007b8b",
  "reason_desc": "some detailed reason for return",
  "images": ["https://some-images-for-item.com"],
  "ttl_approval": "PT24H",
  "ttl_reverseqc": "P3D"
}
```

#### 2. IGM (Issue & Grievance Management) Flow

**Database Table:** `issue_requests`

**Current Status:**
- **Total Issue Requests:** 3
- **Sample Status:** `CLOSED`

**Issue Request Schema:**

| Field | Type | Description |
|-------|------|-------------|
| id | bigint | Primary key |
| status | enum | Issue processing status |
| transaction_id | varchar(191) | ONDC transaction ID |
| ondc_issue_id | varchar(191) | Unique issue identifier |
| ondc_order_id | varchar(191) | Related ONDC order |
| provider_id | varchar(191) | Seller/Provider ID |
| fulfillment_id | varchar(191) | Fulfillment ID |
| items | json | Array of affected item IDs |
| context | json | ONDC context object |
| message | json | Complete issue message payload |

**Issue Status Values:**
- `PROCESSING` - Issue received and being processed
- `INFO_REQUESTED` - Seller requests more information
- `INFO_PROVIDED` - Buyer provided requested info
- `RESOLUTION_PROPOSED` - Seller proposes resolution options
- `RESOLUTION_ACCEPTED` - Buyer accepts resolution
- `RESOLVED` - Issue resolved
- `CLOSED` - Issue closed by buyer

**Issue Resolution Types:**
1. **REFUND** - Provide full/partial refund to customer
2. **REPLACEMENT** - Replace the defective item
3. **CANCEL** - Cancel the order
4. **NO_ACTION** - No action required

**Sample Issue Flow:**

```plaintext
1. OPEN → Buyer creates complaint
2. PROCESSING → Seller acknowledges issue
3. INFO_REQUESTED → Seller asks for more details/images
4. INFO_PROVIDED → Buyer provides additional information
5. PROCESSING → Seller reviews information
6. RESOLUTION_PROPOSED → Seller offers: REFUND, REPLACEMENT, CANCEL
7. RESOLUTION_ACCEPTED → Buyer accepts REFUND
8. RESOLVED → Seller processes refund
9. CLOSED → Buyer closes issue with rating (THUMBS_UP/THUMBS_DOWN)
```

### ONDC API Implementation

#### RET12/RET16 Order Update Services

**Files:**
- `packages/Webkul/ONDCAPI/src/Services/RET12/RET12OrderUpdateService.php`
- `packages/Webkul/ONDCAPI/src/Services/RET16/RET16OrderUpdateService.php`
- `packages/Webkul/ONDCAPI/src/Services/RET13/RET13OrderStatusService.php`

**Purpose:** Handle order updates including returns via ONDC protocol

**RET Codes:**
- **RET10** - F&B (Food & Beverage)
- **RET12** - Fashion & Apparel
- **RET13** - Beauty & Personal Care
- **RET16** - Home & Kitchen

**Key Methods:**

**1. `getItems()` - Line 127**
- Retrieves order items
- Includes canceled items with special fulfillment ID
- Fetches previous returned items from RMA repository
- Calculates remaining quantity: `qty_ordered - qty_canceled`

```php
private function getItems($order, $orderRequest, $fulfillment): array
{
    $items = [];
    
    // Get previous returned items
    $previousReturnedItems = app(RmaRequestRepository::class)->findWhere([
        'ondc_order_id' => $order->ondc_order_id,
        'ondc_item_id'  => $this->getData($orderRequest, 'item_id'),
    ]);
    
    // Include canceled items
    foreach ($order->items as $item) {
        if ($item->qty_canceled > 0) {
            $items[] = [
                'id'             => "ondc{$item->product_id}",
                'fulfillment_id' => Utility::getCancelledFulfillment($fulfillment),
                'quantity'       => ['count' => $item->qty_canceled],
            ];
        }
    }
    
    // Include previous returns
    if ($previousReturnedItems->isNotEmpty()) {
        foreach ($previousReturnedItems as $item) {
            $items[] = [
                'id'             => $item->ondc_item_id,
                'fulfillment_id' => $item->return_fulfillment_id,
                'quantity'       => ['count' => (int) $item->qty],
            ];
        }
    }
    
    // Include remaining items
    foreach ($order->items as $item) {
        $items[] = [
            'id'             => "ondc{$item->product_id}",
            'fulfillment_id' => $fulfillment->fulfillment_id,
            'quantity'       => ['count' => $item->qty_ordered - $item->qty_canceled],
        ];
    }
    
    return $items;
}
```

**2. `getFulfillments()` - Line 299**
- Returns array of fulfillment objects
- Includes return fulfillment with status `Return_Initiated`
- Includes `quote_trail` tags showing refund amount calculation

```php
private function getFulfillments(...): array
{
    return [
        // Main fulfillment
        [...],
        
        // Cancel fulfillments
        ...$this->getCancelFulfillment(...),
        
        // Previous return fulfillments
        ...$this->getPreviousReturnFulfillment(...),
        
        // Current return fulfillment
        [
            'id'    => $id,
            'type'  => $fulfillmentRequest['type'],
            'state' => [
                'descriptor' => ['code' => 'Return_Initiated'],
            ],
            'tags' => [
                [
                    'code' => 'return_request',
                    'list' => [...],
                ],
                // Quote trail showing negative value (refund amount)
                [
                    'code' => 'quote_trail',
                    'list' => [
                        ['code' => 'type', 'value' => 'item'],
                        ['code' => 'id', 'value' => "ondc{$item->product_id}"],
                        ['code' => 'currency', 'value' => 'INR'],
                        ['code' => 'value', 'value' => -($item->price * $item->qty_canceled)],
                    ],
                ],
            ],
        ],
    ];
}
```

**3. `getUpdatedQuoteSummary()` - Line 180**
- Calculates order total after returns/cancellations
- Applies negative values for returned items
- Updates shipping and convenience fees

```php
protected function getUpdatedQuoteSummary($order, $orderRequest, $fulfillment): array
{
    $isCancelledOrRefunded = false;
    
    if ($order->status == Order::STATUS_CANCELED || 
        $order->status == Order::STATUS_CLOSED) {
        $isCancelledOrRefunded = true;
    }
    
    if ($isCancelledOrRefunded) {
        return $this->getQuoteSummary($order, $fulfillment);
    }
    
    // Include previous returns in qty_canceled
    $previousReturnedItems = app(RmaRequestRepository::class)->findWhere([
        'ondc_order_id' => $order->ondc_order_id,
        'ondc_item_id'  => $this->getData($orderRequest, 'item_id'),
    ]);
    
    if ($previousReturnedItems->isNotEmpty()) {
        foreach ($previousReturnedItems as $item) {
            $order->items
                ->where('product_id', Str::replace('ondc', '', $item->ondc_item_id))
                ->first()->qty_canceled += $item->qty;
        }
    }
    
    // Calculate subtotal with returns
    $subTotal = 0;
    $itemsBreakup = $order->items->map(function ($item) use (&$subTotal) {
        $qty = $item->qty_ordered - $item->qty_canceled;
        $price = $item->total / $item->qty_ordered;
        $totalPrice = $price * $qty;
        $subTotal += $totalPrice;
        
        return [
            'title' => $item->product->name,
            'price' => ['currency' => 'INR', 'value' => $totalPrice],
            '@ondc/org/item_id' => "ondc{$item->product_id}",
            '@ondc/org/title_type' => 'item',
            '@ondc/org/item_quantity' => ['count' => (int) $qty],
            'item' => ['price' => ['currency' => 'INR', 'value' => $price]],
        ];
    })->toArray();
    
    return [
        'price' => [
            'currency' => 'INR',
            'value' => $order->shipping_amount + $convenienceFee + $subTotal,
        ],
        'breakup' => array_merge($itemsBreakup, $additionalBreakup),
        'ttl' => 'PT6H',
    ];
}
```

#### Issue Management Service

**File:** `packages/Webkul/ONDCAPI/src/Helpers/Issue.php`

**Key Methods:**

**1. `prepareIssueData()` - Line 32**
- Extracts issue details from ONDC issue message
- Identifies consumer and interfacing NP details
- Parses proposed resolutions

**2. `takeAction()` - Line 154**
- Sends response to buyer app (BAP)
- Updates issue status
- Handles INFO_REQUESTED or RESOLUTION_PROPOSED actions

**3. `prepareResolutions()` - Line 287**
- Creates resolution options for buyer to choose from
- Options include: REFUND, REPLACEMENT, CANCEL, NO_ACTION

```php
private function getResolutionOptions(Onboarding $onboarding, object $item, array $data): array
{
    $options = [
        'REFUND' => [
            'id'   => 'R1',
            'desc' => 'Providing refund for the item',
            'extend_tags' => fn () => tap($getCommonTags(), function (&$tags) use ($item) {
                $tags[0]['list'][] = [
                    'descriptor' => ['code' => 'REFUND_AMOUNT'],
                    'value' => $item->total,
                ];
            }),
        ],
        'REPLACEMENT' => [
            'id'   => 'R2',
            'desc' => 'Providing replacement of the item',
        ],
        'CANCEL' => [
            'id'   => 'R3',
            'desc' => 'Providing cancellation of the item',
        ],
        'NO_ACTION' => [
            'id'   => 'R4',
            'desc' => 'No action required',
        ],
    ];
    
    return collect($data['resolutions'])
        ->filter(fn ($type) => isset($options[$type]))
        ->map(function ($type) use ($options, $onboarding) {
            $option = $options[$type];
            return [
                'id'         => $option['id'],
                'ref_id'     => 'R_PARENT',
                'ref_type'   => 'PARENT',
                'descriptor' => [
                    'code'       => $type,
                    'short_desc' => $option['desc'],
                ],
                'updated_at'  => Utility::getFormattedDateTime(),
                'proposed_by' => $onboarding->subscriber_id,
                'tags'        => ($option['extend_tags'])(),
            ];
        })
        ->values()
        ->all();
}
```

### ONDC vs Regular Refund Flow Comparison

| Aspect | Regular Refunds | ONDC Returns | ONDC Issues |
|--------|----------------|--------------|-------------|
| **Trigger** | Admin creates from order view | Buyer initiates via BAP | Buyer raises complaint via BAP |
| **Database Table** | `refunds` | `rma_requests` | `issue_requests` |
| **Status Flow** | `refunded` (single state) | 8 states (Initiated → Delivered) | 7 states (Processing → Closed) |
| **Inventory Return** | Automatic | Via RMA status updates | After resolution accepted |
| **Refund Processing** | Immediate on creation | After `Liquidated` status | After `RESOLVED` status |
| **Communication** | Email notification | ONDC protocol webhooks | ONDC IGM protocol |
| **Marketplace Integration** | Via event listeners | Order update flow | Issue flow |
| **Count in System** | 12 (all non-ONDC) | 21 (all ONDC) | 3 (all ONDC) |

### Key Differences in ONDC Refund Logic

1. **No Direct Refund Creation:**
   - ONDC orders do not use the standard `RefundController::store()` method
   - Returns are initiated by buyers through their buyer apps (BAPs)
   - Seller responds via ONDC protocol, not direct refund creation

2. **Two-Step Process:**
   - **Step 1: Return (RMA)** - Physical return of goods
   - **Step 2: Refund (Settlement)** - Financial refund via ONDC settlement

3. **Quote Trail System:**
   - Refunds are represented as negative values in `quote_trail` tags
   - Formula: `-($item->price * $item->qty_canceled)`
   - Included in order update responses

4. **Status Tracking:**
   - RMA status tracks physical return journey
   - Issue status tracks dispute resolution
   - Order status updated based on return completion

5. **No Email Notifications:**
   - Uses ONDC protocol webhooks instead
   - Buyer apps handle customer notifications
   - Seller receives updates via `on_update` callbacks

### Integration with Core Refund System

Currently, there is **no integration** between ONDC returns/issues and the core refund system:

**Evidence:**
- Zero refunds exist for ONDC orders in `refunds` table
- All 608 ONDC orders have `NULL` refund relationships
- RMA requests and Issue requests are tracked separately
- No event listeners connect ONDC flow to core refund repository

**Potential Integration Points:**

1. **After RMA `Liquidated` Status:**
   - Create refund record in `refunds` table
   - Link to ONDC order via `order_id`
   - Populate amounts from RMA quote trail

2. **After Issue `RESOLVED` with REFUND:**
   - Create refund record when resolution is REFUND type
   - Extract refund amount from resolution tags
   - Mark as ONDC-originated refund

3. **Settlement Integration:**
   - When ONDC settlement confirms refund
   - Update refund record with settlement details
   - Track refund completion

**Recommended Implementation:**

```php
// In RET12OrderUpdateService or RET16OrderUpdateService
public function syncOrderUpdate(array $request)
{
    // After processing RMA or Issue resolution
    if ($rmaRequest->status === 'Liquidated') {
        // Create refund record
        $refundRepository = app(RefundRepository::class);
        $refundRepository->create([
            'order_id' => $order->id,
            'refund' => [
                'items' => $this->mapRmaItemsToRefundItems($rmaRequest),
                'shipping' => 0,
                'adjustment_refund' => 0,
                'adjustment_fee' => 0,
            ],
        ]);
    }
}
```

### Conclusion - ONDC Refund Status

**Current State:**
- ✅ ONDC returns are tracked via RMA system (21 requests)
- ✅ ONDC issues/disputes handled via IGM (3 requests)
- ✅ Physical return flow is complete and functional
- ❌ **No ONDC orders have entries in `refunds` table**
- ❌ **Financial refunds for ONDC orders are not tracked in core system**
- ❌ **Admin refund list does not show ONDC refunds**

**Recommendation:**
Implement a bridge between ONDC's RMA/IGM system and the core refund tracking system to:
1. Show ONDC refunds in admin panel
2. Track refund amounts for ONDC orders
3. Generate consistent refund reports across all channels
4. Integrate with marketplace commission adjustments

---

## End of Analysis

