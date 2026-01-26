# Frictionless SPEI Deposits

## Overview

Frictionless SPEI is a payment feature that enables automatic processing of SPEI bank transfers even when they don't perfectly match an existing pending transaction.

**How it works:** Tonder (as your Payment Service Provider) assigns a unique CLABE account for each of your customers. By combining CLABE-based identification with flexible tracking through the `external_id` field, the system can intelligently reconcile deposits that would otherwise require manual intervention.

This document explains how the feature works, when it activates, and how to implement it correctly.

### Quick Reference: The Two-Layer System

```
Incoming SPEI Deposit
        │
        ├─► Layer 1: CLABE Number (Primary Key)
        │   • Unique account assigned by Tonder for each merchant customer
        │   • 18-digit unique bank account number
        │   • Used for deposit identification and matching
        │
        └─► Layer 2: external_id (Flexible Identifier)
            • Links CLABE to your internal records
            • Common uses:
            •   - Customer/Player ID (most common)
            •   - Transaction UUID from your system
            •   - Order reference number
            • Facilitates reconciliation and tracking
```

**Your responsibility:** Send `external_id` with every payment to enable reliable matching and reconciliation.

---

## The Challenge: When Standard Matching Fails

Traditional SPEI deposit processing relies on matching incoming transfers to pending transactions using:

- Transaction amount
- Payment reference
- Timing windows

However, real-world scenarios frequently break these assumptions:

**Scenario 1: Amount Mismatch**
A customer initiates a 100 MXN deposit but actually transfers 600 MXN from their bank. The system can't match by amount alone.

**Scenario 2: No Pending Transaction**
A customer makes a direct bank transfer using their saved CLABE without going through your checkout flow first. There's no pending transaction to match against.

In both cases, without proper user identification, these deposits would either fail or require manual reconciliation.

---

## The Solution: Enhanced Reconciliation with Flexible Identifiers

Frictionless SPEI solves these challenges by combining two identification mechanisms:

1. **CLABE number** - Unique account assigned by Tonder for each merchant customer (primary key)
2. **Flexible identifier (`external_id`)** - Your internal reference that links to the CLABE

This dual-layer approach enables reliable transaction matching even when traditional metadata fails.

### Understanding Tonder's CLABE Assignment Model

**Tonder assigns CLABEs for your customers:**

- When a customer initiates their first SPEI deposit, Tonder generates a unique CLABE
- This CLABE is specific to: **Your merchant account + That specific customer**
- The CLABE remains associated with that customer for future deposits
- Each customer gets their own dedicated CLABE for deposits

**The reconciliation flow:**

```
Your Customer → Unique CLABE (assigned by Tonder) → Incoming Deposit → Match via external_id
```

### The `external_id` Field

`external_id` is a **flexible linking mechanism** that works alongside CLABE to facilitate transaction matching, tracking, and reconciliation. You choose what identifier makes sense for your business model.

**Common use cases:**

1. **Customer/User/Player ID (Most Common)**
   - Gaming: `external_id: "PLAYER-12345"`
   - E-commerce: `external_id: "CUSTOMER-98765"`
   - Services: `external_id: "ACCOUNT-45678"`

2. **Transaction UUID**
   - Some merchants prefer transaction-level tracking
   - `external_id: "TXN-uuid-1234-5678-90ab"`
   - Useful for order-specific reconciliation

3. **Order/Session Reference**
   - `external_id: "ORDER-2024-001234"`
   - Links deposit to specific purchase or session

**What you send:**

- Your chosen internal identifier (customer ID, transaction UUID, order reference, etc.)

**What Tonder does:**

- Stores this identifier alongside the CLABE assignment
- Uses it to strengthen deposit matching when CLABE alone isn't sufficient
- Returns it in webhooks for your reconciliation process
- Enables historical lookback to recover the identifier from CLABE history

**Key principle:**

```
CLABE = Tonder-assigned deposit account (primary identifier)
external_id = Your internal reference (facilitates your reconciliation)
Combined = Robust matching and tracking
```

This mapping must be consistent with your chosen strategy (customer-based, transaction-based, or order-based).

---

## Use Case 1: Mismatched Amount Deposits

### Status

✅ **Live in Staging/Sandbox**

### How It Works

1. **Customer initiates deposit:** Creates a checkout for 100 MXN with their user ID
2. **Pending transaction created:** System stores the user's `external_id`
3. **Customer transfers different amount:** Sends 600 MXN via SPEI
4. **System receives deposit:** Amount doesn't match, but CLABE and `external_id` do
5. **Automatic reconciliation:** Transaction is marked as "Success" with mismatched_deposit flag

### Technical Flow

**Initial Payment Request:**

```json
{
  "amount": 100,
  "currency": "MXN",
  "payment_method": "spei",
  "metadata": {
    "external_id": "USER-12345",
    "order_id": "ORDER-98765"
  }
}
```

**System Response:**

- Generates or assigns a CLABE for this deposit
- Stores the CLABE → external_id mapping
- Creates pending transaction

**Incoming SPEI Transfer:**

- **Primary identifier:** CLABE matches the pending transaction ✓
- Amount: 600 MXN (different from expected 100 MXN)
- Reference: Valid

**System Decision:**

- ✅ **CLABE matches** → Identifies the correct deposit account (primary key)
- ✅ **external_id available** → Links to user: "USER-12345" (supplementary data)
- ❌ Amount differs → Flag as mismatched
- ✅ **Process automatically** using CLABE + external_id context

**Resulting Webhook:**

```json
{
  "event": "confirmed",
  "data": {
    "transaction_status": "Success",
    "amount": 600.0,
    "metadata": {
      "external_id": "USER-12345",
      "mismatched_deposit": "True",
      "original_expected_amount": "100"
    }
  }
}
```

### What This Enables

- **CLABE-based matching:** Primary deposit identification works reliably
- **User context enrichment:** external_id provides immediate user linkage without additional lookups
- **No manual intervention:** Deposit processes automatically using both identifiers
- **Clear flagging:** You know the amount differed and can decide how to handle it
- **Direct reconciliation:** Webhook contains both CLABE reference and user identifier
- **Audit trail:** Full history of what was expected vs. what was received

---

## Use Case 2: Direct SPEI Transfers (No Checkout)

### Status

⚠️ **In Internal Testing**

### How It Works

1. **No checkout flow:** Customer transfers money directly from their bank
2. **System receives deposit:** No pending transaction exists
3. **CLABE lookup:** System finds previous successful transaction with same CLABE
4. **Identity extraction:** Retrieves `external_id` from historical transaction
5. **Auto-creation:** New transaction created with inherited user identity

### Technical Flow

**No Active Checkout:**

- Customer uses their existing CLABE from a previous deposit
- Transfers 700 MXN directly via their banking app
- System has no pending transaction to match

**System Response (CLABE-based lookup):**

1. Receives SPEI notification with CLABE: `710969000000565332`
2. **Queries transaction history using CLABE** (primary key)
3. Finds most recent successful transaction for this CLABE
4. **Extracts `external_id: "USER-12345"`** from that historical transaction
5. Creates new transaction inheriting the CLABE → user mapping

**The Key Mechanism:**

- **CLABE is the lookup key:** System searches "show me the last successful transaction for this CLABE"
- **external_id is recovered:** Once the historical transaction is found, external_id is extracted
- **User identity restored:** New deposit is automatically linked to the correct user

**Resulting Webhook:**

```json
{
  "event": "confirmed",
  "data": {
    "transaction_status": "Success",
    "amount": 700.0,
    "reference": "FLESS-a971fc1c95e0",
    "metadata": {
      "external_id": "USER-12345",
      "approval_type": "amount",
      "concept": "Frictionless deposit - auto-created"
    }
  }
}
```

### What This Enables

- **CLABE recognition:** System identifies returning customers by their CLABE
- **User identity recovery:** external_id is automatically retrieved from transaction history
- **True frictionless deposits:** Customers can transfer directly without checkout
- **One-time setup:** Once a CLABE is used successfully with external_id, it's "registered"
- **Reduced friction:** No need to go through your UI for every deposit
- **Automatic crediting:** You receive webhook with both CLABE reference and user identifier

---

## How CLABE and external_id Work Together

### The Two-Layer Identification System

Frictionless SPEI relies on a complementary relationship between Tonder-assigned CLABEs and your internal identifiers:

```
┌─────────────────────────────────────────────────┐
│  Incoming SPEI Deposit                          │
│  CLABE: 710969000000565332                      │
│  Amount: 600 MXN                                │
└─────────────────────────────────────────────────┘
                    │
                    ↓
        ┌───────────────────────┐
        │   Layer 1: CLABE       │  ← Tonder-assigned
        │   Identifies deposit   │     (Which account?)
        │   account              │
        └───────────────────────┘
                    │
                    ↓
        ┌───────────────────────┐
        │   Layer 2: external_id │  ← Your identifier
        │   Links to your        │     (Which customer/transaction?)
        │   internal records     │
        └───────────────────────┘
                    │
                    ↓
        ┌───────────────────────┐
        │   Result:              │
        │   - Customer: USER-123 │
        │   OR Transaction: TXN  │
        │   Credit 600 MXN       │
        └───────────────────────┘
```

### Tonder's CLABE Management (What Tonder Does)

**Tonder handles CLABE assignment and lifecycle:**

1. **First deposit request from a customer:**
   - Tonder generates unique CLABE for this merchant + customer combination
   - CLABE is returned in payment response
   - This CLABE is now dedicated to this customer

2. **Subsequent deposits:**
   - Same customer can reuse their assigned CLABE
   - CLABE remains linked to the customer across sessions
   - Direct transfers to the CLABE are recognized

3. **CLABE storage:**
   - Tonder maintains the CLABE → Merchant relationship
   - Tonder tracks CLABE creation and usage history
   - Tonder provides CLABE in all webhook notifications

### Your Responsibility: Linking with external_id

**You control what external_id represents and must track it:**

**Option 1: Customer-Based Tracking (Most Common)**

When a customer creates a payment:

```javascript
// Your customer initiates deposit
const payment = await tonder.createPayment({
  amount: 500,
  metadata: {
    external_id: "CUSTOMER-12345", // Your customer ID
  },
});

// Tonder assigns CLABE: 710969000000565332
// You should optionally store: CLABE → CUSTOMER-12345 mapping
await db.savePaymentMethod({
  customer_id: "CUSTOMER-12345",
  clabe: payment.clabe, // Store for reference
  first_used: new Date(),
});
```

**Option 2: Transaction-Based Tracking**

Some merchants prefer transaction-level granularity:

```javascript
// Your system creates unique transaction ID
const transactionId = generateUUID(); // "TXN-abc-123"

const payment = await tonder.createPayment({
  amount: 500,
  metadata: {
    external_id: transactionId, // Your transaction UUID
  },
});

// Store transaction details
await db.saveTransaction({
  transaction_id: transactionId,
  customer_id: "CUSTOMER-12345",
  clabe: payment.clabe,
  amount: 500,
});
```

**Note on storage:** While Tonder manages CLABE assignment and persistence, you may want to store the CLABE for your own reference, auditing, or customer service purposes. However, this is optional - Tonder always includes the CLABE in webhook notifications.

### Why Both Are Needed

**CLABE alone (what Tonder knows):**

- Identifies which Tonder-assigned account received funds
- Links deposit to the correct merchant
- Enables Tonder to route the webhook to your system

**external_id adds your context:**

- Tells you which customer, transaction, or order this relates to
- Enables your business logic (credit customer, complete order, etc.)
- Provides flexibility in your reconciliation strategy
- Travels with transaction for your convenience

**Together they enable:**

- Tonder matches incoming SPEI to the right merchant (via CLABE)
- You match the deposit to the right customer/transaction (via external_id)
- Frictionless flow works through Tonder's CLABE history + your external_id

### Integration Patterns by Strategy

**Pattern 1: Customer-Centric (Recommended for most merchants)**

```javascript
// Payment creation
async function createCustomerDeposit(customerId, amount) {
  const payment = await tonder.createPayment({
    amount: amount,
    currency: "MXN",
    payment_method: "spei",
    metadata: {
      external_id: customerId, // Link to customer
    },
  });

  // Optional: Store CLABE for your reference
  await db.updateCustomer(customerId, {
    assigned_clabe: payment.clabe,
    last_deposit_at: new Date(),
  });

  return payment;
}

// Webhook processing
async function processWebhook(webhook) {
  const customerId = webhook.data.metadata.external_id;
  const amount = webhook.data.amount;

  // Credit customer directly
  await creditCustomerAccount(customerId, amount);
}
```

**Pattern 2: Transaction-Centric**

```javascript
// Payment creation
async function createOrderDeposit(orderId, customerId, amount) {
  const txnId = `TXN-${orderId}-${Date.now()}`;

  const payment = await tonder.createPayment({
    amount: amount,
    currency: "MXN",
    payment_method: "spei",
    metadata: {
      external_id: txnId, // Link to specific transaction
      order_id: orderId, // Additional context
    },
  });

  await db.saveTransaction({
    transaction_id: txnId,
    order_id: orderId,
    customer_id: customerId,
    amount: amount,
    clabe: payment.clabe,
  });

  return payment;
}

// Webhook processing
async function processWebhook(webhook) {
  const txnId = webhook.data.metadata.external_id;

  // Lookup transaction to find customer and order
  const txn = await db.getTransaction(txnId);

  // Complete the order
  await completeOrder(txn.order_id, txn.customer_id, webhook.data.amount);
}
```

---

## Implementation Guide

### 1. Choose Your external_id Strategy

Decide what `external_id` will represent in your system:

**Option A: Customer/User ID (Recommended)**

- Use when you want to track deposits by customer
- Simplest reconciliation: direct customer crediting
- Example: `external_id: "CUSTOMER-12345"`

**Option B: Transaction UUID**

- Use when you need transaction-level granularity
- Useful for order-specific tracking
- Example: `external_id: "TXN-uuid-abc-123"`

**Option C: Order/Session Reference**

- Use when deposits tie to specific orders
- Example: `external_id: "ORDER-2024-001234"`

### 2. Always Send `external_id`

Include your chosen identifier in every payment request:

```json
{
  "amount": 500,
  "currency": "MXN",
  "payment_method": "spei",
  "metadata": {
    "external_id": "<YOUR_IDENTIFIER>",
    "order_id": "<OPTIONAL_ADDITIONAL_CONTEXT>"
  }
}
```

### 3. Optionally Store CLABE References

While Tonder manages CLABE assignment and persistence, you may want to store CLABEs for your own purposes:

```javascript
// Optional: Store CLABE for reference
const payment = await tonder.createPayment({
  metadata: { external_id: "CUSTOMER-12345" },
});

// Tonder assigns CLABE, you optionally store it
await db.updateCustomer("CUSTOMER-12345", {
  assigned_clabe: payment.clabe, // For your reference
  last_deposit_initiated: new Date(),
});
```

**When to store CLABEs:**

- Customer service needs (showing customers their deposit account)
- Audit and compliance requirements
- Additional validation in webhook processing
- Analytics and reporting

**When NOT to store CLABEs:**

- Simple use cases where external_id alone is sufficient
- Transaction-based tracking (CLABE not reused)
- When minimizing data storage is priority

### 4. Read from Canonical Location

In webhook payloads, always read from:

```
$.data.metadata.external_id
```

**✅ Correct:**

```javascript
const identifier = webhookPayload.data.metadata.external_id;
```

**❌ Incorrect:**

```javascript
// Don't rely on nested request metadata
const identifier =
  webhookPayload.data.metadata._incoming_request.metadata.external_id;
```

### 5. Implement Your Reconciliation Logic

Based on your chosen strategy:

**For Customer-Based Tracking:**

```javascript
async function processDeposit(webhook) {
  const customerId = webhook.data.metadata.external_id;
  const amount = webhook.data.amount;

  // Direct crediting
  await creditCustomerAccount(customerId, amount);
}
```

**For Transaction-Based Tracking:**

```javascript
async function processDeposit(webhook) {
  const txnId = webhook.data.metadata.external_id;
  const amount = webhook.data.amount;

  // Lookup transaction details
  const txn = await db.getTransaction(txnId);

  // Complete order and credit customer
  await completeOrder(txn.order_id, txn.customer_id, amount);
}
```

**With Optional CLABE Verification (if you store CLABEs):**

```javascript
async function processDeposit(webhook) {
  const clabe = webhook.data.clabe;
  const customerId = webhook.data.metadata.external_id;

  // Optional: Verify CLABE matches your records
  const stored = await db.getCustomerClabe(customerId);

  if (stored && stored.clabe !== clabe) {
    // Unexpected CLABE - flag for review
    await alertSecurityTeam(webhook);
    return;
  }

  // Proceed with crediting
  await creditCustomer(customerId, webhook.data.amount);
}
```

### 6. Handle Special Flags

For mismatched deposits, check for:

```javascript
const isMismatched = webhookPayload.data.metadata.mismatched_deposit === "True";
const originalAmount = webhookPayload.data.metadata.original_expected_amount;
const actualAmount = webhookPayload.data.amount;

if (isMismatched) {
  // Decide how to handle the difference
  // - Credit the actual amount?
  // - Refund the difference?
  // - Contact the customer?
}
```

### 7. Ensure Consistency

Use the same `external_id` strategy consistently:

**If using customer/player IDs:**

- ✅ Always send the same customer ID format
- ✅ Use across all payment methods (not just SPEI)
- ✅ Include in subsequent deposits from same customer
- ✅ Apply to refunds and related operations
- ❌ Don't mix formats ("CUSTOMER-123" vs "CUST-123" vs "123")

**If using transaction UUIDs:**

- ✅ Generate unique ID per transaction
- ✅ Store mapping to customer/order in your system
- ✅ Ensure UUIDs are truly unique
- ❌ Don't reuse transaction IDs

**Optional CLABE storage (if you choose to store):**

- ✅ Store when first received from Tonder
- ✅ Update last_used timestamps for analytics
- ✅ Use for customer service lookups
- ❌ Don't modify or reassign CLABEs (they're Tonder-managed)

---

## Data Flow Examples

### Complete Flow: Mismatched Deposit

```
1. Customer initiates checkout
   POST /payment
   {
     "amount": 100,
     "metadata": { "external_id": "CUSTOMER-12345" }
   }

2. Tonder creates Pending transaction
   - Assigns CLABE: 710969000000565332 (for this merchant + customer)
   - Stores: CLABE → external_id → merchant mapping
   - Returns CLABE to merchant in payment response

3. Merchant optionally stores CLABE
   - Database: CUSTOMER-12345 → CLABE 710969000000565332
   - (This is optional - Tonder maintains the authoritative mapping)

4. Customer transfers 600 MXN (not 100!)
   - Bank sends SPEI notification to Tonder
   - Transfer to CLABE: 710969000000565332

5. Tonder's reconciliation
   - **CLABE matches ✓** (identifies merchant + customer account)
   - Amount differs ✗ (600 vs expected 100)
   - external_id available ✓ (provides merchant's context)
   → Mark as Success with mismatched flag

6. Webhook sent to merchant
   {
     "event": "confirmed",
     "data": {
       "clabe": "710969000000565332",
       "amount": 600.0,
       "metadata": {
         "external_id": "CUSTOMER-12345",
         "mismatched_deposit": "True",
         "original_expected_amount": "100"
       }
     }
   }

7. Merchant processes webhook
   - Reads external_id: CUSTOMER-12345
   - (Optional: Verifies CLABE matches stored value)
   - Credits 600 MXN to customer account
   - Logs amount discrepancy for review
```

### Complete Flow: Direct Transfer

```
1. No checkout - customer transfers directly
   - Customer uses their existing CLABE: 710969000000565332
   - (This CLABE was assigned by Tonder in a previous deposit)
   - Transfers 700 MXN from their bank app

2. Tonder receives SPEI notification
   - Transfer to CLABE: 710969000000565332
   - No pending transaction found
   - Triggers frictionless flow

3. Tonder performs CLABE-based lookback
   - Query Tonder's records: "Find last successful transaction for CLABE 710969000000565332"
   - Result: Transaction from 2 days ago found
   - **Extract: external_id = "CUSTOMER-12345"** from Tonder's historical record
   - Identifies merchant from CLABE ownership

4. Tonder auto-creates transaction
   - Create new payment_id
   - Inherit CLABE → merchant → customer mapping
   - Apply external_id from history: "CUSTOMER-12345"
   - Mark as Success immediately

5. Webhook sent to merchant
   {
     "event": "confirmed",
     "data": {
       "clabe": "710969000000565332",
       "amount": 700.0,
       "metadata": {
         "external_id": "CUSTOMER-12345",
         "concept": "Frictionless deposit - auto-created"
       }
     }
   }

6. Merchant processes webhook
   - Reads external_id: CUSTOMER-12345
   - (Optional: Verifies CLABE matches stored records)
   - Credits 700 MXN to customer account
   - No manual intervention needed
```

---

## Benefits Summary

### For Your Operations

- **Reduced manual work:** Automatic reconciliation of mismatched deposits
- **Faster processing:** No need to queue deposits for review
- **Better UX:** Customers see credits faster
- **Lower support costs:** Fewer "where's my money?" tickets

### For Your Customers

- **Flexibility:** Can deposit any amount, not locked to exact quote
- **Convenience:** Can use saved CLABE for direct transfers
- **Speed:** Deposits process automatically, no delays
- **Trust:** Money arrives in their account reliably

### For Your Development Team

- **Simpler integration:** One field (`external_id`) handles complex scenarios
- **Future-proof:** Foundation for merchant-specific field aliases
- **Reliable webhooks:** Always contains user context
- **Clear audit trail:** Full visibility into matching logic

---

## Best Practices

### 1. **Choose the Right external_id Strategy**

Select an identifier strategy that matches your business model:

✅ **Customer-based (Recommended for most):**

- One identifier per customer/player
- Simplest reconciliation
- Example: `"CUSTOMER-12345"`, `"PLAYER-98765"`

✅ **Transaction-based:**

- Unique ID per transaction
- Better for order-specific tracking
- Example: `"TXN-uuid-abc-123"`

✅ **Hybrid approach:**

- Include both in metadata
- `external_id: "CUSTOMER-123"` + `order_id: "ORDER-456"`

❌ **Avoid:**

- Inconsistent formats
- Non-unique identifiers
- Personally identifiable information (use internal IDs, not emails)

### 2. **Understand Tonder's CLABE Management**

Remember that Tonder handles CLABE lifecycle:

✅ **What Tonder does:**

- Assigns unique CLABE per merchant + customer
- Maintains CLABE → merchant → customer mapping
- Performs CLABE lookbacks for frictionless deposits
- Includes CLABE in all webhook notifications

✅ **What you should do:**

- Trust Tonder's CLABE assignments
- Optionally store CLABEs for customer service
- Always read CLABE from webhooks (don't assume)
- Use external_id for your reconciliation logic

❌ **Don't:**

- Try to generate or manage CLABEs yourself
- Assume CLABE patterns or structure
- Modify or reassign CLABEs

### 3. **Implement Appropriate Validation**

Validate based on your chosen strategy:

**Always verify:**

- ✅ Webhook signature/authentication
- ✅ external_id exists and is valid in your system
- ✅ No duplicate processing (idempotency)
- ✅ Amount is within expected/reasonable range

**Optional CLABE verification (if you store CLABEs):**

```javascript
// Only if you've chosen to store CLABEs for additional security
const storedClabe = await getStoredClabe(customerId);
if (storedClabe && storedClabe !== webhook.data.clabe) {
  await flagForReview(webhook);
}
```

**For transaction-based tracking:**

- Verify transaction UUID is valid and not reused
- Check transaction maps to correct customer/order

### 4. **Handle Edge Cases**

Plan for:

- Very large mismatches (500% of expected amount)
- Multiple deposits in short timeframe
- Refund scenarios
- Disputed transactions
- Unknown/invalid external_id values
- Transaction UUID collisions (if using transaction-based)

### 5. **Log Everything**

For frictionless deposits, log:

- CLABE received in webhook
- external_id and what it represents (customer/transaction/order)
- Original vs. actual amounts (for mismatched deposits)
- Auto-creation events (Use Case 2)
- Reconciliation results
- Customer/order crediting actions

### 6. **Communicate with Customers**

When processing mismatched deposits:

- Notify customers of the discrepancy
- Explain how you handled it
- Provide clear transaction history
- Make refunds easy if needed

---

## Forward Compatibility

### Current Implementation

```json
{
  "metadata": {
    "external_id": "USER-12345"
  }
}
```

### Future: Semantic Aliases

Upcoming platform versions will support merchant-specific field names:

```json
{
  "metadata": {
    "player_id": "USER-12345", // Gaming merchants
    "customer_id": "USER-12345", // E-commerce merchants
    "account_id": "USER-12345" // Service merchants
  }
}
```

These will automatically map to `external_id` internally.

**Your integration won't break:** The `external_id` approach is backward compatible and will continue working unchanged.

---

## Technical Reference

### Required Fields

| Field          | Location             | Type   | Required | Description                                                                     |
| -------------- | -------------------- | ------ | -------- | ------------------------------------------------------------------------------- |
| external_id    | metadata.external_id | string | Yes      | Your internal identifier (customer ID, transaction UUID, order reference, etc.) |
| amount         | root                 | number | Yes      | Transaction amount                                                              |
| currency       | root                 | string | Yes      | Currency code (e.g., "MXN")                                                     |
| payment_method | root                 | string | Yes      | Must be "spei" for this feature                                                 |

### Webhook Fields

| Field                    | Location                               | Type   | Description                                                       |
| ------------------------ | -------------------------------------- | ------ | ----------------------------------------------------------------- |
| clabe                    | data.clabe                             | string | Tonder-assigned CLABE for this deposit (18 digits)                |
| external_id              | data.metadata.external_id              | string | Your identifier (canonical location - customer/transaction/order) |
| mismatched_deposit       | data.metadata.mismatched_deposit       | string | "True" if amount differed from expected                           |
| original_expected_amount | data.metadata.original_expected_amount | string | Original amount from checkout (if mismatched)                     |
| concept                  | data.concept                           | string | Describes the deposit type                                        |
| transaction_status       | data.transaction_status                | string | "Success" for completed deposits                                  |

### Event Types

| Event     | Type    | Description                                       |
| --------- | ------- | ------------------------------------------------- |
| confirmed | deposit | Standard SPEI deposit confirmed                   |
| confirmed | deposit | Mismatched amount deposit (check metadata)        |
| confirmed | deposit | Frictionless auto-created deposit (check concept) |

---

## Troubleshooting

### "Why wasn't my deposit matched?"

**Check:**

1. Was `external_id` sent in original payment request?
2. Did the CLABE match a previous transaction?
3. Is the CLABE associated with an active business account?
4. Are webhooks configured correctly?
5. **Does your database have the CLABE → user mapping stored?**

### "external_id is missing from webhook"

**Possible causes:**

1. Not sent in original payment request
2. Using incorrect field name
3. Nested in wrong metadata location
4. Reading from non-canonical location

**Solution:**  
Always send `metadata.external_id` in requests and read from `data.metadata.external_id` in webhooks.

### "CLABE in webhook doesn't match my stored records"

**Cause:**  
You've optionally stored a CLABE for a customer, but the webhook shows a different CLABE.

**Possible reasons:**

1. Customer used a different CLABE (Tonder may assign multiple CLABEs per customer in certain scenarios)
2. Your storage logic had a bug during initial payment creation
3. Manual intervention changed your records
4. Edge case: Customer initiated payment from different device/session

**Solution:**

- Trust Tonder's CLABE in the webhook (it's authoritative)
- Use external_id as primary reconciliation mechanism
- If you store CLABEs, treat them as reference data, not source of truth
- Consider updating your stored CLABE to match latest webhook
- If pattern persists, verify your CLABE storage logic

**Remember:** Tonder manages CLABE assignment and persistence. Your CLABE storage is optional and for reference purposes only.

### "Multiple deposits matched to wrong user"

**Cause:**  
Different transactions sharing the same CLABE but different `external_id` values.

**Solution:**  
Ensure one CLABE = one user mapping. If users share CLABEs, frictionless flow may not work reliably.

---

## Summary for Developers

| Aspect                      | Implementation                                         |
| --------------------------- | ------------------------------------------------------ |
| **Tonder's role**           | Assigns and manages CLABEs (1 per merchant + customer) |
| **Your role**               | Send external_id for reconciliation                    |
| **external_id flexibility** | Customer ID, transaction UUID, or order reference      |
| **What to send**            | Your chosen identifier in `metadata.external_id`       |
| **CLABE storage**           | Optional - for your reference/customer service         |
| **Where to read**           | `data.metadata.external_id` in webhooks                |
| **CLABE source of truth**   | Always trust CLABE from Tonder's webhook               |
| **Critical validation**     | Verify external_id is valid in your system             |
| **Use Case 1**              | Live on Staging (mismatched amounts)                   |
| **Use Case 2**              | In testing (direct transfers without checkout)         |
| **Breaking changes**        | None planned - future aliases will be additive         |

**Key principle:**

- Tonder assigns CLABEs and manages the CLABE → merchant → customer relationship
- You send external_id to facilitate your own reconciliation (customer/transaction/order)
- Frictionless works through Tonder's CLABE lookback + your external_id

---

## Next Steps

1. **Choose your external_id strategy:** Decide between customer-based, transaction-based, or hybrid approach
2. **Review current implementation:** Ensure `external_id` is being sent in all SPEI payments
3. **Decide on CLABE storage:** Determine if you need to store CLABEs for customer service/analytics
4. **Update webhook handlers:** Read from `data.metadata.external_id` and implement your reconciliation logic
5. **Test mismatched deposits:** Use staging environment to verify Use Case 1 behavior
6. **Plan for Use Case 2:** Prepare for frictionless direct transfers (coming soon)
7. **Monitor and optimize:** Track reconciliation success rates and external_id validation

---

## Support

For questions about implementing Frictionless SPEI:

- Check platform documentation
- Review webhook examples in this guide
- Test thoroughly in staging environment
- Contact technical support for integration assistance

---

_Document Version: 1.0_  
_Last Updated: January 2026_  
_Feature Status: Use Case 1 (Staging), Use Case 2 (Internal Testing)_
