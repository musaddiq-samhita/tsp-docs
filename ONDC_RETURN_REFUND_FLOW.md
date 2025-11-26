# ONDC Return and Refund Flow - Current Implementation

**Document Date:** November 26, 2025  
**System:** Samhita TSP - ONDC Seller Platform

---

## Executive Summary

The ONDC return management and refund processing are **two separate, non-integrated systems**. Returns are handled automatically through the ONDC protocol and shipping partners, but refunds must be manually created by administrators. There is no automatic refund generation when returns are completed.

---

## 1. Return Management Flow (RMA System)

### Process Overview

When a customer initiates a return through their buyer app (Paytm, PhonePe, etc.), the following workflow occurs:

### **Step 1: Return Request Received**
- Customer selects items and reason for return in their buyer app
- Request arrives in the system as an "Order Update Request"
- System creates an RMA (Return Merchandise Authorization) record
- Request appears in **Admin Panel → Company → RMA** section

### **Step 2: Return Approval**
**Admin Action Required:**
- Navigate to **Admin → Company → RMA**
- Review return request details (items, quantities, reason)
- Click **"Approve"** button to accept the return

**System Actions:**
- Sets pickup time window (next 30 minutes)
- Notifies buyer app of approval via ONDC protocol
- Creates return order with shipping partner (ShipRocket or Effimove)
- Status changes to: **"Return_Approved"**

### **Step 3: Pickup Scheduled**
**Admin Action Required:**
- Click **"Create Return Order"** button
- Select shipping provider (ShipRocket or Effimove)
- Confirm pickup details

**System Actions:**
- Generates AWB (Airway Bill) number for return shipment
- Schedules pickup with courier partner
- Shares tracking details with buyer via ONDC

### **Step 4: Item Pickup**
**Admin Action Required:**
- Click **"Picked"** button once courier collects the item

**System Actions:**
- Updates tracking information
- Notifies buyer app that item is in transit
- Status changes to: **"Return_Picked"**

### **Step 5: Return Delivery**
**Admin Action Required:**
- Once item physically arrives at seller warehouse
- Verify item condition
- Click **"Return Delivered"** button

**System Actions:**
- Updates return status to "Delivered"
- Sends delivery confirmation to buyer app via ONDC
- Updates settlement details in payment tracking
- **Does NOT create refund automatically** ❌

### **Step 6: Liquidation (Final Settlement)**
**Admin Action Optional:**
- Click **"Liquidated"** button to mark return as complete

**System Actions:**
- Marks return as financially settled in ONDC protocol
- Sends final status update to buyer app
- **Does NOT create refund automatically** ❌

### Alternative Failure Paths

**If Pickup Fails:**
- Admin clicks **"Pick Failed"** button
- System notifies buyer app
- Can retry pickup or reject return

**If Return is Rejected:**
- Admin clicks **"Reject"** button at approval stage
- System notifies buyer with rejection reason
- No shipment created

**If Return Fails in Transit:**
- Admin clicks **"Return Failed"** button
- System marks return as unsuccessful
- May initiate re-delivery to customer

---

## 2. Settlement System (ONDC Financial Tracking)

### Purpose
The settlement system tracks money movement between participants in the ONDC network (buyer app, seller app, and platform). It integrates with NPCI's NOCS (National Open Commerce Settlement) platform.

### How It Works

**During Order Placement:**
- System records payment transaction details
- Bank account information stored for settlement
- Settlement status marked as "NOT-PAID"

**During Settlement Process:**
- Platform sends settlement request to NPCI NOCS
- Request includes payment splits for all parties:
  - **Seller amount** (product cost + shipping)
  - **Platform commission** (marketplace fees)
  - **Buyer app commission** (convenience fees)

**Settlement Tracking:**
- Navigate to **Admin → Company → Settlements**
- View settlement status: PENDING / SETTLED / NOT_SETTLED
- Check reconciliation reports

### For Returns

When return is delivered:
- System updates settlement details in ONDC protocol
- Shows negative payment adjustment (refund amount)
- Settlement request **may** be sent to NOCS for reverse payment
- **Does NOT create local refund record** ❌

### Key Point
Settlement system handles external banking/payment tracking between ONDC participants. It does **NOT** update your local database records for customer refunds.

---

## 3. Refund Management System

### Current Process: **100% Manual**

Refunds must be manually created by administrators for every return, even after the return is delivered and accepted.

### How to Create a Refund

**Step 1: Navigate to Refunds**
- Go to **Admin Panel → Sales → Refunds**
- Click **"Create Refund"** button

**Step 2: Select Order**
- Search for the order number
- Click on order to open refund creation form

**Step 3: Specify Refund Details**
- Select items to refund (checkboxes)
- Enter quantity for each item
- Set shipping refund amount (if applicable)
- Add adjustment refund (additional amount to refund)
- Add adjustment fee (amount to deduct from refund)

**Step 4: Calculate Totals**
- Click **"Update Totals"** button
- System calculates:
  - Subtotal (item costs)
  - Tax amount
  - Shipping amount
  - Discount adjustments
  - Grand total refund amount

**Step 5: Submit Refund**
- Review calculated total
- Click **"Refund"** button

### What Happens After Refund Creation

**Database Updates:**
- Creates refund record with all financial details
- Updates order's "Total Refunded" amount
- Creates marketplace refund for seller commission adjustments

**Inventory Management:**
- Returns refunded quantities back to product stock
- Updates inventory availability

**Seller Settlements:**
- Adjusts seller payout status to "refunded"
- Deducts refund amount from seller's pending payments
- Updates commission calculations

**Customer Notifications:**
- Sends refund confirmation email to customer
- Email includes refund amount and item details

---

## 4. The Integration Gap

### Critical Issue: No Automatic Refund Creation

**What Happens:**
1. Customer initiates return via buyer app → ✅ Works
2. Admin approves return → ✅ Works
3. Courier picks up item → ✅ Works
4. Item delivered to seller → ✅ Works
5. Admin marks "Return Delivered" → ✅ Works
6. ONDC protocol updated → ✅ Works
7. Settlement details adjusted → ✅ Works
8. **Refund created in database** → ❌ **NEVER HAPPENS**
9. **Customer receives money** → ❌ **NEVER HAPPENS**
10. **Inventory returned to stock** → ❌ **NEVER HAPPENS**

### Required Manual Intervention

After completing return delivery:
1. Note the order number from RMA screen
2. Navigate separately to **Sales → Refunds → Create**
3. Search for the order
4. Manually enter all refund details (items, quantities, amounts)
5. Submit refund

**Time Gap:** Between return delivery and manual refund creation, customer sees:
- No refund in their buyer app
- No email confirmation
- No money credited to account

---

## 5. Real-World Example: Order #1640

### Timeline

**November 7, 2025**
- Customer initiates return via buyer app
- RMA request #36 created in system
- Admin approves return

**November 8-11, 2025**
- ShipRocket return order created (AWB: 19041829614806)
- Courier picks up item from customer
- Item in transit to seller warehouse

**November 12, 2025**
- Item delivered to seller warehouse
- Admin clicks "Return Delivered" button
- RMA status updated to "Delivered"
- ONDC protocol notified
- Settlement details updated with refund amount (₹78.23)

**November 26, 2025 (Today - 14 days later)**
- ❌ No refund record exists in database
- ❌ Order still shows ₹0.00 refunded (should show ₹78.23)
- ❌ Customer never received refund confirmation
- ❌ Customer money not returned
- ❌ Inventory not returned to stock

### Current Status
- **RMA Status:** Return_Approved
- **Return Status:** Delivered
- **Refund Created:** NO
- **Customer Impact:** Waiting 14 days for ₹78.23 refund with no communication

---

## 6. System Navigation Reference

### For Return Management

**View All Returns:**
- Path: **Admin Panel → Company → RMA**
- Shows: List of all return requests with status

**View Return Details:**
- Click on RMA ID from list
- Shows: Customer info, items, reason, timeline, tracking

**Take Actions:**
- Available buttons based on status:
  - Approve / Reject (initial stage)
  - Create Return Order (after approval)
  - Picked (after courier pickup)
  - Return Delivered (after warehouse receipt)
  - Liquidated (final settlement)
  - Pick Failed / Return Failed (error cases)

### For Refund Management

**View All Refunds:**
- Path: **Admin Panel → Sales → Refunds**
- Shows: List of all refunds processed

**Create New Refund:**
- Path: **Admin Panel → Sales → Refunds → Create Refund**
- Or: **Admin Panel → Sales → Orders → [Order] → Refund button**

**View Refund Details:**
- Click on refund ID from list
- Shows: Refund breakdown, items, amounts, status

### For Settlement Tracking

**View Settlements:**
- Path: **Admin Panel → Company → Settlements**
- Shows: ONDC settlement requests and status

**View Settlement Details:**
- Click on settlement record
- Shows: Payment splits, bank details, reconciliation status

---

## 7. Key Operational Points

### What Works Automatically
✅ Return request reception from ONDC network  
✅ RMA record creation  
✅ ONDC protocol communication (all status updates)  
✅ Shipping partner integration (ShipRocket/Effimove)  
✅ Return tracking and status updates  
✅ Settlement detail updates in ONDC messages  

### What Requires Manual Action
⚠️ Return approval/rejection decision  
⚠️ Return order creation with shipping partner  
⚠️ Status updates (Picked, Delivered, Liquidated)  
❌ **Refund creation - ALWAYS manual, no automation**  
❌ **Refund amount calculation - manual entry**  
❌ **Customer refund notification - only after manual refund**  

### Business Process Gap

**Expected Flow:**
Return Delivered → System Auto-Creates Refund → Customer Receives Money

**Actual Flow:**
Return Delivered → Admin Must Remember → Manually Create Refund → Customer Receives Money

**Risk:**
- Forgotten refunds (like Order #1640)
- Delayed customer payments
- Customer dissatisfaction
- Compliance issues with ONDC policies

---

## 8. Recommendations for Operations

### Immediate Workaround

**Create Manual Checklist:**
1. Daily review of RMA list for "Delivered" status
2. Cross-check against refunds created
3. Create missing refunds same day
4. Maintain tracking spreadsheet if needed

### For Order #1640 Specifically

**Action Required Today:**
1. Go to **Admin → Sales → Refunds → Create**
2. Search for Order #1640
3. Select all returned items
4. Enter quantities and shipping refund
5. Submit to process ₹78.23 refund
6. Follow up with customer regarding 14-day delay

### Long-Term Solution Needed

**System Enhancement Required:**
- Automatic refund creation when "Return Delivered" is clicked
- Or: Automated reminder/notification when return delivered but refund pending
- Or: Workflow integration requiring refund creation before status change to "Liquidated"

---

## Document Control

**Version:** 1.0  
**Last Updated:** November 26, 2025  
**Author:** System Analysis  
**Review Status:** Current Implementation Documentation

**Note:** This document describes the current "as-is" implementation. System enhancements are required to automate the refund creation process for ONDC returns.
