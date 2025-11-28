## How Does Money Go Back to Customer for ONDC Orders?

This is a fundamental aspect of how ONDC payment settlement works. Here's the key insight:

### **The Seller App (BPP) Does NOT Directly Refund the Customer**

For ONDC orders, the payment is collected **by the Buyer App (BAP)**, not by the Seller App. This is clearly visible in Order 1640's payment details:

```json
{
    "collected_by": "BAP",           // ← Customer paid the BAP
    "payment_status": "PAID",
    "settlement_status": "NOT-PAID"  // ← BAP hasn't settled with Seller yet
}
```

### **ONDC Payment Flow Explained**

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                       ONDC PAYMENT & REFUND ARCHITECTURE                                 │
└─────────────────────────────────────────────────────────────────────────────────────────┘

                      PAYMENT FLOW (Forward)
    ┌────────────┐         ┌────────────┐         ┌────────────┐
    │  Customer  │ ──$──►  │  BAP       │ ──$──►  │  BPP/Seller│
    │            │         │ (MyStore)  │         │  (You)     │
    └────────────┘         └────────────┘         └────────────┘
         │                       │                      │
         │   Pays ₹78.23         │   Settles later     │
         │   via BAP's           │   (after return     │
         │   payment gateway     │   window expires)   │
         └───────────────────────┴──────────────────────┘

                      REFUND FLOW (Reverse)
    ┌────────────┐         ┌────────────┐         ┌────────────┐
    │  Customer  │ ◄──$──  │  BAP       │ ◄─info─ │  BPP/Seller│
    │            │         │ (MyStore)  │         │  (You)     │
    └────────────┘         └────────────┘         └────────────┘
         │                       │                      │
         │   BAP refunds         │   on_update with    │
         │   customer            │   quote_trail       │
         │   (from their wallet) │   (negative value)  │
         └───────────────────────┴──────────────────────┘
```

### **What Happens When You Create a Refund in Admin**

1. **Seller App (Your System)**
   - Creates internal refund record in `refunds` table
   - Updates order status to `closed`
   - Updates marketplace payout status to `refunded`
   - **Does NOT call any external payment gateway**

2. **Communication to BAP (via on_update)**
   When RMA status changes to `Return_Delivered`, the system sends an `on_update` callback to BAP with **`quote_trail`** containing negative values:

   ```json
   {
       "tags": [
           {
               "code": "quote_trail",
               "list": [
                   {"code": "type", "value": "item"},
                   {"code": "id", "value": "ondc3069"},
                   {"code": "currency", "value": "INR"},
                   {"code": "value", "value": "-78.23"}  // ← NEGATIVE = Refund
               ]
           }
       ]
   }
   ```

3. **BAP's Responsibility**
   - BAP receives the `on_update` with negative `quote_trail`
   - BAP initiates the refund to customer's original payment method
   - BAP adjusts the settlement amount owed to BPP (Seller App)

### **For Order 1640 Specifically**

| **Entity** | **Role** | **Action** |
|------------|----------|------------|
| **Customer (Mayuri Deore)** | Paid ₹78.23 | Will receive refund from BAP |
| **BAP (prd.mystore.in)** | Collected payment | Responsible for refunding customer |
| **BPP (Your Seller App)** | Sold the item | Sends `on_update` with refund info |
| **Settlement Status** | `NOT-PAID` | BAP never settled with you (so no money to return) |

### **Why No External API is Called During Refund**

The money is **still with the BAP**! The payment confirmation shows:

```json
"settlement_status": "NOT-PAID"  // BAP hasn't sent money to seller yet
```

Since the BAP hasn't settled with the Seller App yet, there's no money transfer from Seller to BAP needed. The BAP simply:
1. Receives the `on_update` with return/refund information
2. Refunds the customer from the money they still hold
3. Adjusts the settlement amount (won't pay the seller for this order)

### **What Gets Sent to BAP (The "API Call")**

When RMA status becomes `Return_Delivered`, the `on_update` is sent to BAP:

**Endpoint:** `POST https://prd.mystore.in/ondc/1.0/on_update`

**Payload includes:**
```json
{
    "context": {
        "action": "on_update",
        "bap_uri": "https://prd.mystore.in/ondc/1.0/",
        ...
    },
    "message": {
        "order": {
            "id": "68ff48a4c54c9938c2c53ddb",
            "state": "Completed",
            "fulfillments": [
                {
                    "type": "Return",
                    "state": {"descriptor": {"code": "Return_Delivered"}},
                    "tags": [
                        {
                            "code": "quote_trail",
                            "list": [
                                {"code": "value", "value": "-78.23"}  // Negative = refund amount
                            ]
                        }
                    ]
                }
            ]
        }
    }
}
```

### **Summary**

| **Question** | **Answer** |
|--------------|------------|
| **Who refunds the customer?** | The BAP (Buyer App - MyStore in this case) |
| **Why no payment API from Seller?** | Seller never received the money (settlement_status = NOT-PAID) |
| **What does Seller send?** | `on_update` callback with `quote_trail` showing negative values |
| **Where is response stored?** | No response stored locally; BAP acknowledges with ACK |
| **How does BAP know refund amount?** | From `quote_trail` tag with negative value in `on_update` |

The ONDC architecture is designed this way because the BAP collects payment and is responsible for refunds. The Seller App (BPP) only needs to communicate the refund information, not actually move money.
