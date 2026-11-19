# Midtrans Sandbox Integration

Documentation for integrating **Midtrans Snap in Sandbox mode only**.

> **Environment policy:** This project is restricted to Midtrans Sandbox. Do not use Production credentials, endpoints, SDK settings, or Snap.js URLs.

## Project Status

This repository is currently empty. The sections below describe the intended integration contract; they do not claim that frontend or backend payment features have already been implemented.

| Component | Status |
|---|---|
| Midtrans Sandbox configuration | Required |
| Backend: create Snap transaction | Planned |
| Frontend: display Snap payment page | Planned |
| Backend: handle payment notifications | Planned |
| Backend: verify transaction status | Planned |

## Integration Flow

1. The customer proceeds to checkout in the application.
2. The backend creates a unique order and calculates the amount from trusted order data.
3. The backend requests a Snap transaction token from Midtrans Sandbox using the **Server Key**.
4. The frontend receives the token and opens the Midtrans Snap payment interface using the **Client Key**.
5. Midtrans sends a payment notification to the backend.
6. The backend verifies the notification and, when necessary, retrieves the transaction status from Midtrans.
7. The application updates the order only after the payment has been verified.

The frontend must not create a Snap token or receive the Server Key.

## Sandbox Configuration

Get the **Sandbox** Client Key and Server Key from the Midtrans Merchant Administration Portal (MAP) under **Settings → Access Keys**.

Configure these values in the appropriate environment-specific configuration:

| Variable | Used by | Description |
|---|---|---|
| `MIDTRANS_CLIENT_KEY` | Frontend | Sandbox Client Key |
| `MIDTRANS_SERVER_KEY` | Backend only | Sandbox Server Key |

Keep the Server Key secret. Never commit credentials to the repository or expose the Server Key in browser code.

### Sandbox Endpoints

| Purpose | Sandbox URL |
|---|---|
| Snap API: create transaction | `https://app.sandbox.midtrans.com/snap/v1/transactions` |
| Snap.js | `https://app.sandbox.midtrans.com/snap/snap.js` |
| Core API: get transaction status | `https://api.sandbox.midtrans.com/v2/{order_id}/status` |

Use only these Sandbox URLs. Do not remove the `.sandbox` subdomain.

## Backend Requirements

### Create a Snap transaction

- Generate a unique `order_id` for each transaction.
- Calculate `gross_amount` on the backend using trusted order data.
- Send the request to the Sandbox Snap API.
- Authenticate with HTTP Basic Authentication: Base64-encode `MIDTRANS_SERVER_KEY:` (Server Key as username and an empty password).
- Return the Snap transaction token to the frontend; do not return the Server Key.

Minimum request structure:

```json
{
  "transaction_details": {
    "order_id": "ORDER-UNIQUE-ID",
    "gross_amount": 10000
  }
}
```

### Handle payment notifications

The notification endpoint must:

1. Accept Midtrans HTTP POST notifications.
2. Verify `signature_key` using SHA-512 of the concatenated values:

   ```
   order_id + status_code + gross_amount + MIDTRANS_SERVER_KEY
   ```

3. Match the notification to the corresponding order in the application.
4. Verify the amount and transaction status against the stored order and the verified Midtrans transaction data.
5. Treat a payment as successful only when the applicable conditions are satisfied:
   - `status_code` is `200`.
   - `transaction_status` is `settlement` or `capture`.
   - If `fraud_status` is present, it must be `accept`.
6. Handle repeated notifications idempotently. Do not create duplicate payments or apply the same state transition more than once.

Never trust the browser redirect or frontend callback as proof of payment.

### Verify transaction status

Use the Sandbox Get Transaction Status API when reconciling a payment or when a notification is delayed or needs verification. Confirm that the returned order identifier and amount match the application's stored order before updating it.

## Frontend Requirements

Load Snap.js with the Sandbox Client Key:

```html
<script
  src="https://app.sandbox.midtrans.com/snap/snap.js"
  data-client-key="YOUR_SANDBOX_CLIENT_KEY">
</script>
```

Use the transaction token returned by the backend to open the Snap payment interface. Frontend callbacks may be used for user experience, but the backend notification and status verification remain authoritative.

## Local Testing

1. Configure Sandbox Client Key and Server Key.
2. Run the application and expose the backend notification endpoint through a publicly reachable HTTPS URL.
3. Configure the notification URL in Midtrans Sandbox settings.
4. Create a test order and complete a payment using Midtrans Sandbox test credentials.
5. Confirm that the backend verifies the notification and updates the matching order exactly once.
6. Test pending, successful, expired, cancelled, and rejected/denied payment outcomes as applicable.

Use Midtrans' official Sandbox testing documentation for payment-method test credentials and scenarios.

## Security Rules

- Sandbox only: Production mode is out of scope.
- Keep the Server Key on the backend only.
- Never accept the order amount or payment success state solely from the frontend.
- Verify notification signatures and reconcile transaction identity and amount.
- Make notification processing idempotent.
- Do not mark an order as paid based only on a redirect, client callback, or `order_id` supplied by the browser.

## Official Documentation

- [Midtrans Snap Integration Guide](https://docs.midtrans.com/docs/snap-snap-integration-guide)
- [Midtrans Access Keys](https://docs.midtrans.com/docs/access-keys)
- [Backend Integration — Create Snap Transaction](https://docs.midtrans.com/reference/backend-integration)
- [HTTP(S) Notifications / Webhooks](https://docs.midtrans.com/docs/https-notification-webhooks)
- [Handle Notifications](https://docs.midtrans.com/reference/handle-notifications)
- [Get Transaction Status](https://docs.midtrans.com/reference/get-transaction-status)
- [Testing Payments on Sandbox](https://docs.midtrans.com/docs/testing-payment-on-sandbox)

---

**Environment:** Midtrans Sandbox only  
**Integration method:** Midtrans Snap  
**Production support:** Not supported
