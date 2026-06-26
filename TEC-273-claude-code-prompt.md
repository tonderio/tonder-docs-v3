# Claude Code Task: Fix Webhook Docs — TEC-273

## What to do

Apply a documentation fix to one file, then create and push a PR to GitHub.

---

## Context

**Repo:** `tonder-docs-v3` — a Mintlify docs site using MDX.  
**Live site:** https://docs.tonder.io  
**Linear issue:** TEC-273 (Urgent) — assigned to Arturo Torres  
**Branch name to use:** `arturot/tec-273-update-documentation-webhooks-api-direct`

**Problem:** The page `direct-integration/webhooks/how-webhooks-works.mdx` documents the legacy webhook payload format (nested under a `data` key). API Direct actually sends a **flat payload** with a completely different field structure. A merchant got confused and thought their integration was broken.

---

## File to edit

`direct-integration/webhooks/how-webhooks-works.mdx`

Make **three targeted changes** to this file:

---

### Change 1 — Replace the "Webhook Payload Structure" section

Find and replace the entire section starting at `## Webhook Payload Structure` through the closing ` ``` ` of the JSON example.

**Replace with:**

```mdx
## Webhook Payload Structure

API Direct webhooks use a **flat payload** — all fields are at the top level with no nested `data` wrapper. Here are the key fields:

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique identifier for this webhook event |
| `operation_type` | string | Type of operation (e.g., `payment`) |
| `amount` | string | Transaction amount as a string |
| `currency` | string | ISO 4217 currency code (e.g., `MXN`) |
| `client_reference` | string | Your own reference ID for the transaction |
| `status` | string | Current transaction status (e.g., `Success`, `Pending`, `Failed`) |
| `provider` | string | Payment provider that processed the transaction |
| `transaction_id` | string | Tonder's internal transaction identifier |
| `payment_method_type` | string | Payment method used (e.g., `SPEI`, `CARD`, `OXXO`) |
| `created` | string | ISO 8601 timestamp of when the event was created |
| `metadata` | object | Custom key-value pairs passed when the payment was created |
| `event_type` | string | Specific event that triggered this notification (e.g., `payment_Success`, `payment_Pending`) |
| `action` | string | Action associated with the event (e.g., `MODIFY`) |

Here are examples of the two most common event types:

<CodeGroup>

```json payment_Success
{
  "id": "fc38522e-3e5d-45b8-ba6a-ece72caee71f",
  "operation_type": "payment",
  "amount": "70",
  "currency": "MXN",
  "client_reference": "f6d16280-7bff-4bb7-b6f1-967f9721248b",
  "status": "Success",
  "provider": "tonder",
  "transaction_id": "e9340a04-6d68-4afc-86c5-79f8b7c87de4",
  "payment_method_type": "SPEI",
  "created": "2026-05-21T19:15:32.029134Z",
  "metadata": {
    "order_id": "f6d16280-7bff-4bb7-b6f1-967f9721248b",
    "transaction_type": "deposit"
  },
  "event_type": "payment_Success",
  "action": "MODIFY"
}
```

```json payment_Pending
{
  "id": "b47faf2d-844e-4f14-bedd-f998a016cacd",
  "operation_type": "payment",
  "amount": "200",
  "currency": "MXN",
  "client_reference": "a38f5292-4c5e-42cd-aa73-15763d2d4862",
  "status": "Pending",
  "provider": "tonder",
  "transaction_id": "aa15fe61-a680-40b9-8546-4a98c204dc9d",
  "payment_method_type": "SPEI",
  "created": "2026-05-21T21:27:56.220201Z",
  "metadata": {
    "order_id": "a38f5292-4c5e-42cd-aa73-15763d2d4862",
    "transaction_type": "deposit"
  },
  "event_type": "payment_Pending",
  "action": "MODIFY"
}
```

</CodeGroup>
```

---

### Change 2 — Replace the "Real-time Payment Status Updates" accordion

Find the `<Accordion title="Real-time Payment Status Updates">` block and replace its content with:

```mdx
<Accordion title="Real-time Payment Status Updates">

Handle payment status changes to automatically fulfill orders when payments complete or fail. Note that the API Direct payload is flat — all fields are at the top level.

```python
@app.route('/webhook', methods=['POST'])
def handle_payment_webhook():
    payload = request.get_json()
    
    if payload['event_type'] == 'payment_Success':
        # Payment completed successfully
        fulfill_order(payload['client_reference'])
        
    elif payload['event_type'] == 'payment_Failed':
        # Payment failed
        cancel_order(payload['client_reference'])

    elif payload['event_type'] == 'payment_Pending':
        # Payment is awaiting confirmation (e.g., SPEI transfer in progress)
        mark_order_pending(payload['client_reference'])
    
    return jsonify({'status': 'received'}), 200
```

</Accordion>
```

---

### Change 3 — Replace the "3DS Authentication Handling" accordion

Find the `<Accordion title="3DS Authentication Handling">` block and replace its content with:

```mdx
<Accordion title="3DS Authentication Handling">

Process 3D Secure authentication completions to finalize payments that required additional customer verification.

```python
@app.route('/webhook', methods=['POST'])
def handle_3ds_webhook():
    payload = request.get_json()
    
    if payload['event_type'] == 'payment_Success':
        # 3DS authentication successful, payment completed
        complete_order(payload['client_reference'])
    elif payload['event_type'] == 'payment_Failed':
        # 3DS authentication failed
        cancel_order(payload['client_reference'])
    
    return jsonify({'status': 'received'}), 200
```

</Accordion>
```

---

## Git instructions

1. Make sure you are on `main` and it is up to date:
   ```bash
   git checkout main && git pull origin main
   ```

2. Create the branch:
   ```bash
   git checkout -b arturot/tec-273-update-documentation-webhooks-api-direct
   ```

3. Apply the three changes above to `direct-integration/webhooks/how-webhooks-works.mdx`.

4. Verify the diff looks correct — only the payload structure section and two accordion code blocks should have changed.

5. Commit:
   ```bash
   git add direct-integration/webhooks/how-webhooks-works.mdx
   git commit -m "docs(webhooks): fix API Direct payload format in how-webhooks-works

   The payload structure section was showing the legacy nested format
   (data wrapper) instead of the flat API Direct format. This caused
   merchant confusion when implementing webhook handling.

   Changes:
   - Replace payload field table with correct flat field set
   - Replace JSON example with real payment_Success and payment_Pending
     examples in a CodeGroup
   - Update Python code examples to reference top-level fields and
     correct event_type values (payment_Success, payment_Failed,
     payment_Pending)

   Fixes: TEC-273"
   ```

6. Push and open PR:
   ```bash
   git push origin arturot/tec-273-update-documentation-webhooks-api-direct
   gh pr create \
     --title "docs(webhooks): fix API Direct payload format [TEC-273]" \
     --body "## What
   The 'How Webhooks Work' page was documenting the legacy nested payload format (\`data\` wrapper) instead of the actual API Direct flat payload. A merchant reported integration issues as a direct result.

   ## Changes
   - **Payload field table** — replaced 6 legacy fields with the 13 correct flat fields
   - **JSON examples** — replaced the single legacy example with real \`payment_Success\` and \`payment_Pending\` payloads in a \`<CodeGroup>\`
   - **Python code snippets** — updated to reference top-level fields and correct \`event_type\` values

   ## File changed
   \`direct-integration/webhooks/how-webhooks-works.mdx\`

   Fixes **TEC-273** · Priority: Urgent" \
     --base main
   ```

7. After the PR is merged to `main`, Mintlify will auto-deploy. Verify the fix is live at:
   https://docs.tonder.io/direct-integration/webhooks/how-webhooks-works

---

## What NOT to change

- `direct-integration/webhooks/setup-and-managing.mdx` — no payload examples, no changes needed
- `direct-integration/webhooks/delivery-and-retry.mdx` — no changes needed
- `direct-integration/webhooks/best-practices.mdx` — no changes needed
- Any other file in the repo
