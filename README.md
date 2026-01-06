# @blazium/js-sdk

Official JavaScript SDK for BlaziumPay - Production-ready crypto payment infrastructure.

## Features

- 🔐 **Secure**: HMAC-SHA256 webhook verification with timing-safe comparison
- 💰 **Multi-Chain**: TON, Solana, Bitcoin support
- 🎯 **Reward Locking**: Lock rewards at payment creation (400 coins = exactly 400 coins)
- 🔄 **Idempotency**: Built-in duplicate payment prevention
- ⚡ **Real-time**: Long polling and webhook support
- 📊 **Balance Management**: Track earnings and withdrawals
- 🚀 **Pure JavaScript**: No TypeScript compilation needed
- 🚫 **No Database Required**: Works with any backend stack

## Installation

```bash
npm install @blazium/js-sdk
```

## Quick Start

```javascript
import { BlaziumPayClient } from '@blazium/js-sdk';

const client = new BlaziumPayClient({
  apiKey: process.env.BLAZIUM_API_KEY,
  webhookSecret: process.env.BLAZIUM_WEBHOOK_SECRET,
  environment: 'production'
});

// Create payment with locked reward
const payment = await client.createPayment(
  {
    amount: 10.00,
    currency: 'USD',
    description: 'Premium Pack',
    rewardAmount: 400,
    rewardCurrency: 'coins'
  },
  {
    idempotencyKey: 'order_12345'
  }
);

console.log('Checkout URL:', payment.checkoutUrl);
console.log('Reward (LOCKED):', payment.rewardAmount); // Always 400
```

## Core Concepts

### Reward Metadata (Developer Controlled)

The `rewardAmount` and `rewardCurrency` fields are **optional metadata** that you can use to track what you plan to give users. 
**BlaziumPay does NOT automatically grant rewards** - you must implement your own logic in webhook handlers.

```javascript
// Set reward metadata when creating payment
const payment = await client.createPayment({
  amount: 3.00,
  currency: 'USD',
  rewardAmount: 1000, // Metadata: 1000 gold to grant
  rewardCurrency: 'gold' // Metadata: Gold currency
});

// The rewardAmount is stored in the payment object
// You must implement your own logic to grant rewards when payment confirms
```

### Payment Lifecycle

```
CREATE → PENDING → CONFIRMED → [Your Custom Logic]
         ↓
       EXPIRED / FAILED / CANCELLED
```

**Important:** When a payment reaches `CONFIRMED` status, **you must implement your own webhook handler** to grant 
premium features, add currency, unlock content, or perform any other actions. BlaziumPay handles payment processing; 
you handle the business logic.

### Webhook Verification (Critical)

**Important:** Only verified webhooks receive events from BlaziumPay. You must verify your webhook endpoint in the dashboard before it will receive any events. Unverified webhooks will not receive any webhook events.

```javascript
import express from 'express';
import { BlaziumPayClient } from '@blazium/js-sdk';

const app = express();

// CRITICAL: Use raw body parser for signature verification
app.use(express.json({
  verify: (req, res, buf) => {
    req.rawBody = buf.toString('utf8');
  }
}));

const client = new BlaziumPayClient({
  apiKey: process.env.BLAZIUM_API_KEY,
  webhookSecret: process.env.BLAZIUM_WEBHOOK_SECRET,
  environment: 'production'
});

app.post('/webhooks/blazium', async (req, res) => {
  const signature = req.headers['x-blazium-signature'];
  
  // CRITICAL: Verify signature
  // Note: If webhookSecret is not configured, this will throw a ValidationError
  // with a helpful message explaining that webhooks must be verified first
  if (!client.verifyWebhookSignature(req.rawBody, signature)) {
    return res.status(401).send('Invalid signature');
  }
  
  const webhook = client.parseWebhook(req.rawBody, signature);
  
  if (webhook.event === 'payment.confirmed') {
    const payment = webhook.payment;
    const userId = payment.metadata?.userId;
    
    // YOUR CUSTOM LOGIC HERE - You decide what happens:
    
    // Example: Grant premium features
    if (payment.rewardCurrency === 'premium') {
      await database.users.update(userId, { isPremium: true });
    }
    
    // Example: Add in-game currency
    if (payment.rewardCurrency === 'coins') {
      await database.users.incrementCoins(userId, payment.rewardAmount);
    }
    
    // You have full control - implement whatever logic you need!
  }
  
  res.status(200).send({ received: true });
});
```

**Webhook Verification Process:**
1. Create a webhook endpoint in the dashboard
2. Verify the endpoint ownership (BlaziumPay will send a challenge token)
3. Save the webhook secret securely (shown only once after verification)
4. Configure the secret in your SDK client
5. Only verified webhooks receive events - unverified webhooks are ignored by BlaziumPay

## API Reference

### Initialize Client

```javascript
const client = new BlaziumPayClient({
  apiKey: string,              // Required: Your API key
  webhookSecret?: string,      // Optional: For webhook verification
  baseUrl?: string,            // Optional: Custom API URL
  timeout?: number,            // Optional: Request timeout (default: 15000ms)
  environment?: 'production' | 'sandbox'
});
```

### Create Payment

```javascript
const payment = await client.createPayment(
  {
    amount: number,              // Amount in fiat currency
    currency: string,            // USD, EUR, TRY, etc.
    description?: string,        // Payment description
    metadata?: object,           // Custom data
    redirectUrl?: string,        // Success redirect
    cancelUrl?: string,          // Cancel redirect
    expiresIn?: number,          // Expiration in seconds (60-86400)
    rewardAmount?: number,       // LOCKED reward amount
    rewardCurrency?: string,     // Reward currency
    rewardData?: object          // Additional reward data
  },
  {
    idempotencyKey?: string      // Prevent duplicates
  }
);
```

### Get Payment

```javascript
const payment = await client.getPayment(paymentId);

console.log(payment.status);       // PENDING, CONFIRMED, etc.
console.log(payment.rewardAmount); // Locked reward
console.log(payment.txHash);       // Blockchain transaction
```

### List Payments

```javascript
const response = await client.listPayments({
  status: 'CONFIRMED',  // Optional: PaymentStatus
  currency: 'USD',      // Optional: Filter by currency
  page: 1,              // Optional: Page number
  pageSize: 20          // Optional: Items per page
});

console.log(response.data);        // Array of payments
console.log(response.pagination);  // Pagination info
```

### Cancel Payment

```javascript
const payment = await client.cancelPayment(paymentId);
```

### Get Payment Statistics

```javascript
const stats = await client.getStats();

console.log('Total:', stats.total);
console.log('Confirmed:', stats.confirmed);
console.log('Pending:', stats.pending);
```

### Wait for Confirmation

```javascript
try {
  const confirmed = await client.waitForPayment(
    paymentId,
    300000,  // 5 minutes timeout
    3000     // Poll every 3 seconds
  );
  
  console.log('Payment confirmed!', confirmed.txHash);
} catch (error) {
  console.error('Payment timeout or failed');
}
```

### Balance & Withdrawals

```javascript
// Check merchant balance
const balance = await client.getBalance('TON');

console.log('Total Earned:', balance.totalEarned);
console.log('Available:', balance.availableBalance);
console.log('Pending:', balance.pendingBalance);

// Request withdrawal
const withdrawal = await client.requestWithdrawal({
  chain: 'TON',
  amount: 10,
  destinationAddress: 'YOUR_TON_ADDRESS'
});

// List withdrawals
const withdrawals = await client.listWithdrawals();

// Get specific withdrawal
const withdrawalDetails = await client.getWithdrawal(withdrawalId);
```

### Webhook Verification

```javascript
// Verify signature (timing-safe)
const isValid = client.verifyWebhookSignature(rawBody, signature);

// Parse webhook
const webhook = client.parseWebhook(rawBody, signature);

console.log(webhook.event);   // payment.confirmed, etc.
console.log(webhook.payment); // Full payment object
```

### Utility Methods

```javascript
client.isPaid(payment);              // true if CONFIRMED
client.isPartiallyPaid(payment);     // true if underpaid
client.isExpired(payment);            // true if expired
client.isFinal(payment);              // true if no more updates
client.getPaymentProgress(payment);   // % of amount paid
client.formatAmount(1.5, 'TON');      // "1.5000 TON"
```

## Error Handling

```javascript
import {
  BlaziumError,
  AuthenticationError,
  ValidationError,
  NetworkError,
  TimeoutError,
  PaymentError
} from '@blazium/js-sdk';

try {
  const payment = await client.createPayment({ /* ... */ });
} catch (error) {
  if (error instanceof AuthenticationError) {
    console.error('Invalid API key');
  } else if (error instanceof ValidationError) {
    console.error('Invalid input:', error.details);
  } else if (error instanceof NetworkError) {
    console.error('Network error, retry...');
  } else if (error instanceof TimeoutError) {
    console.error('Request timed out');
  } else if (error instanceof RateLimitError) {
    console.error('Rate limited. Retry after:', error.retryAfter, 'seconds');
  }
}
```

## Enums and Constants

```javascript
import {
  BlaziumChain,
  BlaziumFiat,
  PaymentStatus,
  BlaziumEnvironment,
  WithdrawalStatus,
  WebhookEventType
} from '@blazium/js-sdk';

// Usage examples
const chain = BlaziumChain.TON;
const status = PaymentStatus.CONFIRMED;
const event = WebhookEventType.PAYMENT_CONFIRMED;
```

## Security Best Practices

1. **Never trust frontend signals** - Always verify payments server-side
2. **Verify webhook signatures** - Use `verifyWebhookSignature()` - CRITICAL for security
3. **Use idempotency keys** - Prevent duplicate payments
4. **Implement your own reward logic** - BlaziumPay does NOT automatically grant rewards. You must implement webhook handlers to grant premium features, add currency, or perform other actions
5. **Use rewardAmount as metadata** - Store what you promise users, but implement your own logic to grant it
6. **Store API keys securely** - Use environment variables
7. **Implement timeout handling** - Network issues happen
8. **Log webhook failures** - Monitor for issues
9. **Make webhook handlers idempotent** - Handle duplicate webhook deliveries gracefully

## Requirements

- Node.js >= 18
- ES Modules support (package.json with `"type": "module"`)
- No database required
- Works with any backend framework (Express, Fastify, NestJS, etc.)

## Support

- Documentation: https://docs.blaziumpay.com
- Issues: https://github.com/blaziumpay/js-sdk/issues
- Email: support@blaziumpay.com

## License

MIT © BlaziumPay

