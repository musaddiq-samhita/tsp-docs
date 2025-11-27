# ONDC Return and Refund Flow - Comprehensive Analysis

## 1. Database Schema Overview

### Core Tables for ONDC Returns and Refunds

| Table | Purpose | Key Fields |
|-------|---------|------------|
| [`orders`](packages/Webkul/Sales/src/Models/Order.php) | Main order table | `ondc_order_id`, `transaction_id`, `status` |
| [`rma_requests`](packages/Webkul/ONDCAPI/src/Models/RmaRequest.php) | ONDC return requests | `status`, `ondc_order_id`, `ondc_item_id`, `qty`, `return_fulfillment_id`, `return_order_id`, `return_shipment_id`, `return_awb_code`, `shipping_method` |
| [`order_update_requests`](packages/Webkul/ONDCAPI/src/Repositories/OrderUpdateRequestRepository.php) | ONDC update requests | `type` (item/payment/fulfillment), `ondc_order_id`, `context`, `message` |
| [`settlement_requests`](packages/Webkul/ONDCAPI/src/Repositories/SettlementRequestRepository.php) | Settlement tracking | `action` (on_settle/on_report/on_recon), `transaction_id`, `type` |
| [`refunds`](database/migrations/) | Refund records | `order_id`, `state`, `grand_total` |
| [`order_fulfillment_statuses`](packages/Webkul/ONDCAPI/src/Repositories/OrderFulfillmentStatusRepository.php) | Fulfillment tracking | `fulfillment_status`, `order_status`, `shipment_status_id` |

---

## 2. Return Status State Machine

### RMA Status Transitions (defined in [`RmaRequest.php`](packages/Webkul/ONDCAPI/src/Models/RmaRequest.php:15-27))

```
Return_Initiated → Return_Rejected
                 → Return_Approved → (Manual status updates via shipping provider webhooks)
                 → Liquidated (if previous RMA exists for same order)

Return_Approved → Return_Pick_Failed → Return_Picked
               → Return_Picked → Return_Failed → Return_Delivered
                              → Return_Delivered
```

### Status Enum Values ([`UpdateStatusType.php`](packages/Webkul/ONDCAPI/src/Enums/UpdateStatusType.php:5-14))
- `Return_Initiated` - Initial state when buyer requests return
- `Return_Approved` - Seller approves the return
- `Liquidated` - Item liquidated (for subsequent returns on same order)
- `Return_Rejected` - Seller rejects the return
- `Return_Picked` - Return item picked up by courier
- `Return_Pick_Failed` - Pickup attempt failed
- `Return_Failed` - Return delivery failed
- `Return_Delivered` - Return successfully delivered to seller

---

## 3. End-to-End Return Flow

### Phase 1: Return Initiation (Buyer App → TSP)

1. **ONDC `update` API Call** - Buyer app sends return request
   - Endpoint: `POST /v1/update`
   - Routed via [`APIController`](packages/Webkul/ONDCAPI/src/Http/Controllers/APIController.php:45) or [`RET12APIController`](packages/Webkul/ONDCAPI/src/Http/Controllers/RET12APIController.php:45)
   - Service: [`OrderUpdateService`](packages/Webkul/ONDCAPI/src/Services/OrderUpdateService.php) / [`RET12OrderUpdateService`](packages/Webkul/ONDCAPI/src/Services/RET12/RET12OrderUpdateService.php)

2. **Request Processing** ([`OrderUpdateRequestRepository::syncOrderUpdate()`](packages/Webkul/ONDCAPI/src/Repositories/OrderUpdateRequestRepository.php:24-64))
   ```php
   // Creates order_update_request record
   $orderUpdateRequest = $this->model->create([
       'type' => $request['message']['update_target'], // 'item' for returns
       'ondc_order_id' => $request['message']['order']['id'],
       'context' => $request['context'],
       'message' => $request['message'],
   ]);
   
   // Creates RMA record if update_target is 'item'
   if ($request['message']['update_target'] == UpdateType::ITEM->value) {
       $orderUpdateRequest->rma()->create([
           'ondc_order_id' => $request['message']['order']['id'],
           'ondc_item_id' => $this->getData($request, 'item_id'),
           'qty' => $this->getData($request, 'item_quantity'),
           'return_fulfillment_id' => $this->getData($request, 'id'),
           'reason_desc' => $this->getData($request, 'reason_desc'),
           // ... other fields
       ]);
   }
   ```

3. **Response Preparation** ([`RET12OrderUpdateService::prepareMessage()`](packages/Webkul/ONDCAPI/src/Services/RET12/RET12OrderUpdateService.php:55-122))
   - Fetches order and seller details
   - Prepares fulfillment data with `Return_Initiated` status
   - Calculates updated quote summary

4. **Async Callback** ([`ProcessRequest`](packages/Webkul/ONDCAPI/src/Jobs/ProcessRequest.php) job)
   - Sends `on_update` response to buyer app's BAP URI
   - Includes signed authorization header

### Phase 2: Seller Action (Admin Panel)

1. **RMA Management UI** - Admin views RMA at `/admin/company/rma/view/{id}`
   - Controller: [`RMAController::view()`](packages/Webkul/ONDCSeller/src/Http/Controllers/Admin/RMAController.php:72-112)

2. **Status Update Actions** ([`RMAController::handleAction()`](packages/Webkul/ONDCSeller/src/Http/Controllers/Admin/RMAController.php:138-189))
   ```php
   private array $rmaMethods = [
       'Return_Approved' => 'prepareApprovalRequest',
       'Liquidated' => 'prepareLiquidatedRequest',
       'Return_Rejected' => 'prepareRejectedRequest',
       'Return_Picked' => 'preparePickedRequest',
       'Return_Pick_Failed' => 'preparePickFailedRequest',
       'Return_Failed' => 'prepareReturnFailedRequest',
       'Return_Delivered' => 'prepareReturnDeliveredRequest',
   ];
   ```

3. **ONDC Callback** - Sends `on_update` to buyer app
   ```php
   $postURL = "{$context['bap_uri']}/{$context['action']}"; // on_update
   $response = Http::withHeaders(['Content-Type' => 'application/json'])
       ->post($postURL, ['context' => $context, 'message' => $data]);
   ```

### Phase 3: Return Shipping (Shiprocket/Effimove Integration)

#### Shiprocket Flow:
1. **Create Return Order** ([`RMAController::createReturnOrder()`](packages/Webkul/ONDCSeller/src/Http/Controllers/Admin/RMAController.php:211-281))
   - Prepares return order data with pickup (customer) and delivery (seller) addresses
   - Calls Shiprocket API to create return order
   - Updates RMA with `return_order_id`, `return_shipment_id`

2. **Assign Courier** ([`RMAController::assignCourierToReturn()`](packages/Webkul/ONDCSeller/src/Http/Controllers/Admin/RMAController.php:286-392))
   - Gets available couriers via serviceability check
   - Assigns courier and gets AWB code
   - Updates RMA with `return_awb_code`, `return_courier_id`

3. **Webhook Status Updates** ([`OrderFulfillmentStatusRepository::syncReturnOrderStatus()`](packages/Webkul/ONDCAPI/src/Repositories/OrderFulfillmentStatusRepository.php:648-691))
   - Receives Shiprocket webhook with shipment status
   - Updates RMA `return_status` field
   - Triggers ONDC `on_status` callback

#### Effimove Flow:
1. **Create Effimove Return** ([`RMAController::createEffimoveReturn()`](packages/Webkul/ONDCSeller/src/Http/Controllers/Admin/RMAController.php:450-589))
   - Prepares payload with pickup/drop locations
   - Creates `effimove_returns` record
   - Updates RMA with shipping details

2. **Webhook Status Updates** ([`OrderFulfillmentStatusRepository::syncEffimoveReturnStatus()`](packages/Webkul/ONDCAPI/src/Repositories/OrderFulfillmentStatusRepository.php:733-924))
   - Maps Effimove status to RMA status
   - Triggers ONDC callbacks

---

## 4. Refund Calculation Logic

### Quote Recalculation ([`RET12OrderUpdateService::getUpdatedQuoteSummary()`](packages/Webkul/ONDCAPI/src/Services/RET12/RET12OrderUpdateService.php:180-273))

```php
// Calculate remaining items after return
$subTotal = 0;
$itemsBreakup = $order->items->map(function ($item) use (&$subTotal) {
    $qty = $item->qty_ordered - $item->qty_canceled;
    $price = $item->total / $item->qty_ordered;
    $totalPrice = $price * $qty;
    $subTotal += $totalPrice;
    // ... return breakup item
});

// Add convenience fee and delivery charges
$convenienceFee = $this->searchRequestRepository->calculateConvenienceFee($this->bapId, $order->sub_total);

return [
    'price' => [
        'currency' => 'INR',
        'value' => $order->shipping_amount + $convenienceFee + $subTotal,
    ],
    'breakup' => array_merge($itemsBreakup, $additionalBreakup),
];
```

### Quote Trail Tags ([`RMA::preparePickedFulfillments()`](packages/Webkul/ONDCAPI/src/Helpers/RMA.php:332-354))
```php
$quoteTrailTags = $order->items->map(function ($item) {
    return [
        'code' => 'quote_trail',
        'list' => [
            ['code' => 'type', 'value' => 'item'],
            ['code' => 'id', 'value' => "ondc{$item->product_id}"],
            ['code' => 'currency', 'value' => 'INR'],
            ['code' => 'value', 'value' => -($item->price * $item->qty_canceled)], // Negative for refund
        ],
    ];
});
```

---

## 5. Settlement Flow

### Settlement Actions ([`SettlementAction.php`](packages/Webkul/ONDCAPI/src/Enums/SettlementAction.php:5-9))
- `on_settle` - Settlement notification from RSP
- `on_report` - Settlement report
- `on_recon` - Reconciliation data

### Settlement Processing ([`SettlementService::processSettlement()`](packages/Webkul/ONDCAPI/src/Services/SettlementService.php:35-58))
```php
private function processSettlement(array $order): void
{
    $parts = ['inter_participant', 'self', 'provider'];
    $allSettled = true;

    foreach ($parts as $part) {
        $status = $order[$part]['status'] ?? null;
        if ($status != 'SETTLED') {
            $allSettled = false;
            break;
        }
    }

    $confirmRequest->is_settled = (int) $allSettled;
    $confirmRequest->save();
}
```

---

## 6. Key Service Classes and Components

### Controllers
| Controller | Path | Purpose |
|------------|------|---------|
| [`APIController`](packages/Webkul/ONDCAPI/src/Http/Controllers/APIController.php) | ONDCAPI | Main ONDC API handler |
| [`RET12APIController`](packages/Webkul/ONDCAPI/src/Http/Controllers/RET12APIController.php) | ONDCAPI | RET12 domain handler |
| [`RMAController`](packages/Webkul/ONDCSeller/src/Http/Controllers/Admin/RMAController.php) | ONDCSeller | Admin RMA management |

### Services
| Service | Path | Purpose |
|---------|------|---------|
| [`OrderUpdateService`](packages/Webkul/ONDCAPI/src/Services/OrderUpdateService.php) | ONDCAPI | Processes update requests |
| [`RET12OrderUpdateService`](packages/Webkul/ONDCAPI/src/Services/RET12/RET12OrderUpdateService.php) | ONDCAPI | RET12 update handling |
| [`SettlementService`](packages/Webkul/ONDCAPI/src/Services/SettlementService.php) | ONDCAPI | Settlement processing |

### Repositories
| Repository | Path | Purpose |
|------------|------|---------|
| [`RmaRequestRepository`](packages/Webkul/ONDCAPI/src/Repositories/RmaRequestRepository.php) | ONDCAPI | RMA data access |
| [`OrderUpdateRequestRepository`](packages/Webkul/ONDCAPI/src/Repositories/OrderUpdateRequestRepository.php) | ONDCAPI | Update request handling |
| [`OrderFulfillmentStatusRepository`](packages/Webkul/ONDCAPI/src/Repositories/OrderFulfillmentStatusRepository.php) | ONDCAPI | Fulfillment status sync |
| [`SettlementRequestRepository`](packages/Webkul/ONDCAPI/src/Repositories/SettlementRequestRepository.php) | ONDCAPI | Settlement data |

### Helpers
| Helper | Path | Purpose |
|--------|------|---------|
| [`RMA`](packages/Webkul/ONDCAPI/src/Helpers/RMA.php) | ONDCAPI | Prepares ONDC protocol payloads |

### Jobs
| Job | Path | Purpose |
|-----|------|---------|
| [`ProcessRequest`](packages/Webkul/ONDCAPI/src/Jobs/ProcessRequest.php) | ONDCAPI | Async ONDC callbacks |

---

## 7. API Endpoints

### ONDC Protocol Endpoints (Incoming)
| Endpoint | Action | Service |
|----------|--------|---------|
| `POST /v1/update` | Return request | `OrderUpdateService` |
| `POST /v1/on_settle` | Settlement notification | `SettlementService` |
| `POST /v1/on_report` | Settlement report | `ReportService` |
| `POST /v1/on_recon` | Reconciliation | `ReconciliationService` |

### Admin RMA Endpoints
| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/admin/company/rma` | GET | List RMAs |
| `/admin/company/rma/view/{id}` | GET | View RMA details |
| `/admin/company/rma/action` | POST | Update RMA status |
| `/admin/company/rma/create-return-order` | POST | Create Shiprocket return |
| `/admin/company/rma/assign-courier-to-return` | POST | Assign courier |
| `/admin/company/rma/effimove/create-return/{id}` | POST | Create Effimove return |

---

## 8. Production Data Patterns

### RMA Status Distribution (from production)
| Status | Count |
|--------|-------|
| Return_Delivered | 16 |
| Return_Initiated | 2 |
| Return_Approved | 2 |
| Liquidated | 1 |

### Settlement Types
- `NP-NP` - Network Participant to Network Participant
- `MISC` - Miscellaneous settlements

---

## 9. Error Handling and Retry Mechanisms

### Acknowledgment Handling ([`APIController::prepareAcknowledgment()`](packages/Webkul/ONDCAPI/src/Http/Controllers/APIController.php:103-177))
- Returns `ACK` for successful processing
- Returns `NACK` with error details for failures
- Special handling for cancel and track actions

### Database Transactions ([`OrderUpdateRequestRepository::syncOrderUpdate()`](packages/Webkul/ONDCAPI/src/Repositories/OrderUpdateRequestRepository.php:26-61))
```php
DB::beginTransaction();
try {
    // Create records
    DB::commit();
} catch (\Exception $e) {
    DB::rollBack();
}
```

### Logging
- ONDC transactions logged via `Utility::pushTransactionLogToAnalytic()`
- Shiprocket operations logged to `shiprocket` channel
- Effimove operations logged to `effimove` channel

---

## 10. Flow Diagram Summary

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        ONDC RETURN & REFUND FLOW                            │
└─────────────────────────────────────────────────────────────────────────────┘

BUYER APP                    TSP (Samhita)                    SELLER/LOGISTICS
    │                              │                                │
    │  1. update (return request)  │                                │
    │─────────────────────────────>│                                │
    │                              │  Create order_update_request   │
    │                              │  Create rma_request            │
    │                              │  Status: Return_Initiated      │
    │                              │                                │
    │  2. on_update (ACK)          │                                │
    │<─────────────────────────────│                                │
    │                              │                                │
    │                              │  3. Admin reviews RMA          │
    │                              │─────────────────────────────────>
    │                              │                                │
    │                              │  4. Approve/Reject             │
    │                              │<─────────────────────────────────
    │                              │  Status: Return_Approved       │
    │                              │                                │
    │  5. on_update (status)       │                                │
    │<─────────────────────────────│                                │
    │                              │                                │
    │                              │  6. Create return shipment     │
    │                              │─────────────────────────────────>
    │                              │  (Shiprocket/Effimove)         │
    │                              │                                │
    │                              │  7. Webhook: status updates    │
    │                              │<─────────────────────────────────
    │                              │  Status: Return_Picked →       │
    │                              │          Return_Delivered      │
    │                              │                                │
    │  8. on_update (delivered)    │                                │
    │<─────────────────────────────│                                │
    │                              │                                │
    │  9. update (payment)         │                                │
    │─────────────────────────────>│  Record settlement details     │
    │                              │                                │
    │  10. on_settle               │                                │
    │<─────────────────────────────│  Mark order as settled         │
    │                              │                                │
```

This comprehensive analysis covers the complete ONDC return and refund flow from initiation through settlement, including all code paths, database operations, external API integrations, and state machine transitions.
