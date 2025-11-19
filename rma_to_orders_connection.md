## Connection Path

**`rma_requests.ondc_order_id` → `orders.ondc_order_id` → `orders.id`**

The `orders` table has the `ondc_order_id` field that matches with `rma_requests.ondc_order_id`.

Let me show you more detailed queries to get complete order information:

[3 tools called]

Excellent! Here's the complete connection details:

## Summary

### ✅ Connection Confirmed

```
rma_requests.ondc_order_id = orders.ondc_order_id
```

**Match Rate:** 100% (21 RMA requests → 19 unique orders, 0 unmatched)

### Key Findings

1. **All RMA requests successfully link to orders** via `ondc_order_id`
2. **Product matching:** Use `CONCAT('ondc', order_items.product_id) = rma_requests.ondc_item_id`
3. **19 unique ONDC orders** have RMA requests (some orders have multiple returns)

### RMA Status Distribution

| RMA Status | Count | Associated Order Statuses |
|------------|-------|---------------------------|
| Return_Delivered | 16 | canceled, completed |
| Return_Initiated | 2 | completed |
| Return_Approved | 2 | completed |
| Liquidated | 1 | completed |

## Useful Query Templates

### 1. **Complete RMA to Order Join**

```sql
SELECT 
    rma.id as rma_id,
    rma.ondc_order_id,
    rma.status as rma_status,
    rma.reason_desc,
    -- Order info
    o.id as order_id,
    o.increment_id as order_number,
    o.status as order_status,
    o.grand_total,
    o.customer_email,
    -- Item info
    oi.name as product_name,
    oi.qty_ordered,
    rma.qty as return_qty,
    oi.price,
    (oi.price * rma.qty) as refund_amount
FROM rma_requests rma
LEFT JOIN orders o ON rma.ondc_order_id = o.ondc_order_id
LEFT JOIN order_items oi ON o.id = oi.order_id 
    AND CONCAT('ondc', oi.product_id) = rma.ondc_item_id
```

### 2. **Orders with RMA Requests (for Refund Tracking)**

```sql
SELECT 
    o.id as order_id,
    o.increment_id,
    o.ondc_order_id,
    o.grand_total,
    o.status as order_status,
    COUNT(rma.id) as rma_count,
    GROUP_CONCAT(rma.status ORDER BY rma.created_at) as rma_statuses,
    SUM(CAST(rma.qty AS UNSIGNED) * oi.base_price) as total_return_value
FROM orders o
INNER JOIN rma_requests rma ON o.ondc_order_id = rma.ondc_order_id
LEFT JOIN order_items oi ON o.id = oi.order_id 
    AND CONCAT('ondc', oi.product_id) = rma.ondc_item_id
WHERE o.ondc_order_id IS NOT NULL
GROUP BY o.id
```

### 3. **RMA Requests Ready for Refund (Liquidated Status)**

```sql
SELECT 
    rma.id as rma_id,
    rma.ondc_order_id,
    rma.status,
    o.id as order_id,
    o.increment_id as order_number,
    oi.name as product_name,
    oi.base_price * CAST(rma.qty AS UNSIGNED) as refund_amount_due,
    rma.created_at as return_date,
    rma.updated_at as status_updated_at
FROM rma_requests rma
INNER JOIN orders o ON rma.ondc_order_id = o.ondc_order_id
LEFT JOIN order_items oi ON o.id = oi.order_id 
    AND CONCAT('ondc', oi.product_id) = rma.ondc_item_id
WHERE rma.status = 'Liquidated'
  AND NOT EXISTS (
      SELECT 1 FROM refunds r WHERE r.order_id = o.id
  )
```

## Database Relationship Diagram

```
rma_requests
├── ondc_order_id (varchar) ──→ orders.ondc_order_id (varchar)
├── ondc_item_id (varchar)  ──→ CONCAT('ondc', order_items.product_id)
└── order_update_request_id ──→ order_update_requests.id

orders
├── id (PK)
├── ondc_order_id (unique identifier from ONDC)
├── increment_id (order number shown to users)
└── Related tables:
    ├── order_items (order_id FK)
    ├── refunds (order_id FK)
    ├── invoices (order_id FK)
    ├── shipments (order_id FK)
    └── marketplace_orders (order_id FK)
```

## Key Insights

1. **Product ID Mapping:** ONDC uses prefixed IDs (e.g., "ondc109"), while internal system uses numeric IDs (109)

2. **No Refund Records:** Despite 21 RMA requests (16 completed as "Return_Delivered"), **ZERO entries exist in `refunds` table** for these orders

3. **Return Values:** The sample shows returns ranging from ₹5.90 to ₹23.60 in product value, plus shipping amounts

4. **Customer:** Majority of returns are from the same test customer (Nirdosh Chauhan)

Would you like me to:
1. Create a query to generate refund records for completed RMA requests?
2. Build a view that combines RMA and refund data?
3. Analyze which RMA requests should have triggered refunds but didn't?
