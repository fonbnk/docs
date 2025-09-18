# Liquidity Bridge API Documentation (WIP)

> Draft V2 API for Fonbnk merchants that unifies on-ramp/off-ramp into a single Deposit/Payout flow.

## Table of Contents

- [Overview](#overview)
- [Order Flow Overview](#order-flow-overview)
- [Fiat-to-Crypto Example Flow](#fiat-to-crypto-example-flow)
- [Transfer Types Explanation](#transfer-types-explanation)
- [Authentication & Request Signing](#authentication--request-signing)
- [API Endpoints](#api-endpoints)
  - [Get currencies](#get-currencies)
  - [Get order limits](#get-order-limits)
  - [Get Quote](#get-quote)
  - [Create Order](#create-order)
  - [Get user KYC state](#get-user-kyc-state)
  - [Submit user KYC](#submit-user-kyc)
  - [Trigger intermediate action](#trigger-intermediate-action)
  - [Confirm order](#confirm-order)
  - [Cancel order](#cancel-order)
  - [Get Order](#get-order)
  - [Get orders](#get-orders)
  - [Get merchant balance](#get-merchant-balance)
- [Types](#types-used-in-the-above-definitions)

## Overview

This is a draft API for V2 of the current Fonbnk API for merchants.

In the previous version we had a distinction between "onramp" and "offramp" endpoints with a separate set of endpoints
for each, different request and response formats, and different integrations for each. In this new version we unify the concepts into a single "Liquidity Bridge" API, and we now operate with the concepts
of "deposit" and "payout" instead of "onramp" and "offramp".

The merchant has the access to these currency types:

- Fiat (e.g., NGN, KES, GHS)
- Crypto (e.g., CELO_CUSD, TRON_USDT, POLYGON_USDT)
- Merchant Balance (currently USD)

Merchant can create orders to deposit one currency type and payout another (or the same) currency type. Examples:

- Deposit Fiat (NGN) via Bank Transfer, Payout Crypto (POLYGON_USDT)
- Deposit Crypto (CELO_CUSD) via Crypto Transfer, Payout fiat mobile money (KES)
- Deposit Merchant Balance (USD), Payout Crypto (TRON_USDT)
- Deposit Fiat (GHS) via Mobile Money, Payout Merchant Balance (USD)

With the new architecture, a merchant has access to:

- off-ramp
- on-ramp
- funding merchant balance from supported deposit methods
- withdrawing merchant balance to supported payout methods
- accepting crypto/fiat payments to merchant balance, then paying out via any supported method


## Order flow overview

1. Call [Get currencies](#get-currencies) to list supported currencies, channels, and pairs.
2. Call [Get order limits](#get-order-limits) for the selected deposit/payout configuration.
3. Call [Get user KYC state](#get-user-kyc-state) to determine if KYC is required for the intended amounts and currency types.
   - If KYC is required, call [Submit user KYC](#submit-user-kyc) and wait until status is approved.
4. Call [Get Quote](#get-quote) with the deposit/payout configuration and one side’s amount (deposit.amount or payout.amount, not both).
   - Use deposit.fieldsToCreateOrder and payout.fieldsToCreateOrder to collect all required fields from the user.
5. Call [Create Order](#create-order) with quoteId and the collected fields.
6. User completes the deposit per transfer instructions on the order.
7. If the transfer requires an intermediate action (stk_push / otp_stk_push), call [Trigger intermediate action](#trigger-intermediate-action).
8. If required, call [Confirm order](#confirm-order) to confirm the deposit.
9. Track order status as deposit validates and payout processes.
10. Use [Get Order](#get-order) to poll for status and details at any time.

## Fiat-to-Crypto Example Flow

Let’s do a NGN (fiat) deposit to POLYGON_USDT (crypto) payout. First, call [Get currencies](#get-currencies) and assume you receive:

<details>
<summary>Example response</summary>

```json
[
  {
    "currencyType": "fiat",
    "currencyCode": "NGN",
    "paymentChannels": [
      {
        "type": "bank",
        "transferTypes": [
          "manual",
          "redirect"
        ],
        "isDepositAllowed": true,
        "isPayoutAllowed": true
      },
      {
        "type": "airtime",
        "transferTypes": [
          "ussd"
        ],
        "carriers": [
          {
            "code": "MTN",
            "name": "MTN"
          },
          {
            "code": "AIRTEL",
            "name": "Airtel"
          },
          {
            "code": "GLO",
            "name": "Glo"
          },
          {
            "code": "9MOBILE",
            "name": "9Mobile"
          }
        ],
        "isDepositAllowed": true,
        "isPayoutAllowed": false
      },
      {
        "type": "mobile_money",
        "transferTypes": [
          "stk_push",
          "otp_stk_push"
        ],
        "carriers": [
          {
            "code": "MTN",
            "name": "MTN Mobile Money"
          },
          {
            "code": "AIRTEL",
            "name": "Airtel Money"
          },
          {
            "code": "GLO",
            "name": "Glo Mobile Money"
          },
          {
            "code": "9MOBILE",
            "name": "9Mobile Money"
          }
        ],
        "isDepositAllowed": true,
        "isPayoutAllowed": true
      }
    ],
    "currencyDetails": {
      "countryIsoCode": "NG",
      "countryName": "Nigeria",
      "countryCode": "234",
      "currencySymbol": "₦",
      "countryIcon": "https://cdn.example.com/flags/ng.png"
    },
    "pairs": [
      "crypto",
      "merchant_balance"
    ]
  },
  {
    "currencyType": "crypto",
    "currencyCode": "POLYGON_USDT",
    "paymentChannels": [
      {
        "type": "crypto",
        "transferTypes": [
          "manual"
        ],
        "isDepositAllowed": true,
        "isPayoutAllowed": true
      }
    ],
    "currencyDetails": {
      "network": "POLYGON",
      "asset": "USDT",
      "networkTitle": "Polygon",
      "assetTitle": "USDT",
      "contractAddress": "0xc2132D05D31c914a87C6611C10748AEb04B58e8F",
      "decimals": 6,
      "networkIcon": "https://cdn.example.com/networks/polygon.png",
      "assetIcon": "https://cdn.example.com/assets/usdt.png"
    },
    "pairs": [
      "fiat",
      "merchant_balance"
    ]
  }
]
```
</details>

We see NGN supports deposit via bank/airtime/mobile money, and payout via bank/mobile money. POLYGON_USDT supports both deposit and payout. So we can do Fiat→Crypto (NGN→POLYGON_USDT).

Next, call [Get order limits](#get-order-limits) with:

- depositPaymentChannel: "bank"
- depositCurrencyType: "fiat"
- depositCurrencyCode: "NGN"
- payoutPaymentChannel: "crypto"
- payoutCurrencyType: "crypto"
- payoutCurrencyCode: "POLYGON_USDT"

<details>
<summary>Example response</summary>

```json
{
  "deposit": {
    "min": 1556,
    "max": 311184,
    "minUsd": 1,
    "maxUsd": 200
  },
  "payout": {
    "min": 1,
    "max": 200,
    "minUsd": 1,
    "maxUsd": 200
  }
}
```
</details>

Assume the user wants to receive 100 POLYGON_USDT. Check KYC via [Get user KYC state](#get-user-kyc-state):

- userEmail: "someuser@example.com"
- countryIsoCode: "NG"

<details>
<summary>Example response</summary>

```json
{
  "passedKycType": null,
  "reachedKycLimit": false,
  "currentKycStatus": null,
  "currentKycStatusDescription": null,
  "kycDocuments": [
    {
      "_id": "67da909b739fc481aa525c43",
      "type": "basic",
      "title": "Voter ID",
      "value": "VOTER_ID",
      "requiredFields": [
        {"key": "first_name", "type": "string", "label": "First Name", "required": true},
        {"key": "last_name", "type": "string", "label": "Last Name", "required": true},
        {"key": "dob", "type": "date", "label": "Date of birth", "required": true},
        {
          "key": "id_number",
          "type": "string",
          "label": "ID number",
          "required": true,
          "format": "0000000000000000000",
          "regexp": "^[a-zA-Z0-9 ]{9,29}$",
          "regexpFlags": "i"
        }
      ]
    },
    {
      "_id": "67da93c0dfd3a00f3380b857",
      "type": "advanced",
      "title": "Driving License",
      "value": "DRIVERS_LICENSE",
      "requiredFields": [
        {"key": "first_name", "type": "string", "label": "First Name", "required": true},
        {"key": "last_name", "type": "string", "label": "Last Name", "required": true},
        {"key": "dob", "type": "date", "label": "Date of birth", "required": true},
        {"key": "images", "type": "smile-identity-images", "label": "Verification images", "required": true}
      ]
    }
  ],
  "kycRules": [
    {"operationType": "deposit", "currencyType": "crypto", "min": 0, "max": 100, "type": "basic"},
    {"operationType": "payout", "currencyType": "crypto", "min": 100, "max": "Infinity", "type": "basic"}
  ]
}
```
</details>

In this example, to payout ≥ 100 USD in crypto, advanced KYC is required. Collect:

- first_name
- last_name
- dob (YYYY-MM-DD)
- images (selfie, front, back)

Submit via [Submit user KYC](#submit-user-kyc):

<details>
<summary>Example request</summary>

```json
{
  "userEmail": "someuser@example.com",
  "countryIsoCode": "NG",
  "documentId": "67da93c0dfd3a00f3380b857",
  "userFields": {
    "first_name": "John",
    "last_name": "Doe",
    "dob": "1990-01-01",
    "images": [
      {"image_type_id": 0, "image": "https://cdn.com/selfie.jpg"},
      {"image_type_id": 1, "image": "https://cdn.com/front.jpg"},
      {"image_type_id": 5, "image": "https://cdn.com/back.jpg"}
    ]
  }
}
```
</details>

Wait until currentKycStatus becomes "approved" (poll [Get user KYC state](#get-user-kyc-state)).

Then [Get Quote](#get-quote):

<details>
<summary>Example request</summary>

```json
{
  "deposit": {"paymentChannel": "bank", "currencyType": "fiat", "currencyCode": "NGN"},
  "payout": {"paymentChannel": "crypto", "currencyType": "crypto", "currencyCode": "POLYGON_USDT", "amount": 100}
}
```
</details>

<details>
<summary>Example response</summary>

```json
{
  "quoteId": "68628fa56ff494df5f39faf5",
  "quoteExpiresAt": "2024-10-10T10:10:10.000Z",
  "deposit": {
    "paymentChannel": "bank",
    "currencyType": "fiat",
    "currencyCode": "NGN",
    "currencyDetails": {"countryIsoCode": "NG", "countryName": "Nigeria", "countryCode": "234", "currencySymbol": "₦", "currencyIcon": "https://cdn.example.com/flags/ng.png"},
    "cashout": {
      "exchangeRate": 1500,
      "exchangeRateAfterFees": 1531.1269,
      "amountBeforeFees": 153128,
      "amountAfterFees": 150015,
      "amountBeforeFeesUsd": 102.085333,
      "amountAfterFeesUsd": 100.01,
      "chargedFees": [
        {"id": "provider_fee", "type": "flat_amount", "recipient": "provider", "amount": 50},
        {"id": "platform_fee", "type": "percentage", "recipient": "platform", "amount": 3063}
      ],
      "chargedFeesUsd": [
        {"id": "provider_fee", "type": "flat_amount", "recipient": "provider", "amount": 0.033333},
        {"id": "platform_fee", "type": "percentage", "recipient": "platform", "amount": 2.042}
      ],
      "totalChargedFees": 3113,
      "totalChargedFeesUsd": 2.075333,
      "chargedFeesPerRecipient": {"provider": 50, "platform": 3063},
      "chargedFeesPerRecipientUsd": {"provider": 0.033333, "platform": 2.042},
      "feeSettings": [
        {"id": "provider_fee", "recipient": "provider", "type": "flat_amount", "value": 50, "min": 0, "max": "Infinity"},
        {"id": "platform_fee", "recipient": "platform", "type": "percentage", "value": 2, "min": 0, "max": "Infinity"}
      ]
    },
    "fieldsToCreateOrder": [
      {"key": "phoneNumber", "label": "Phone Number", "required": true, "type": "phone"},
      {"key": "bankCode", "label": "Bank name", "required": true, "type": "enum", "options": [{"value": "120001:02", "label": "9Payment Service Bank"}, {"value": "801:02", "label": "Abbey Mortgage Bank"}]},
      {"key": "bankAccountNumber", "label": "Bank Account Number", "required": true, "type": "string"}
    ],
    "transferType": "manual"
  },
  "payout": {
    "paymentChannel": "crypto",
    "currencyType": "crypto",
    "currencyCode": "POLYGON_USDT",
    "currencyDetails": {"network": "POLYGON", "asset": "USDT", "networkTitle": "Polygon", "assetTitle": "USDT", "contractAddress": "0xc2132D05D31c914a87C6611C10748AEb04B58e8F", "decimals": 6, "networkIcon": "https://cdn.example.com/networks/polygon.png", "assetIcon": "https://cdn.example.com/assets/usdt.png"},
    "cashout": {
      "exchangeRate": 1,
      "exchangeRateAfterFees": 1.001,
      "amountBeforeFees": 100.01,
      "amountAfterFees": 100,
      "amountBeforeFeesUsd": 100.01,
      "amountAfterFeesUsd": 100,
      "feeSettings": [{"id": "gas_fee", "recipient": "blockchain", "type": "flat_amount", "value": 0.01, "min": 0, "max": "Infinity"}],
      "chargedFees": [{"id": "gas_fee", "type": "flat_amount", "recipient": "blockchain", "amount": 0.01}],
      "chargedFeesUsd": [{"id": "gas_fee", "type": "flat_amount", "recipient": "blockchain", "amount": 0.01}],
      "totalChargedFees": 0.01,
      "totalChargedFeesUsd": 0.01,
      "chargedFeesPerRecipient": {"blockchain": 0.01},
      "chargedFeesPerRecipientUsd": {"blockchain": 0.01}
    },
    "fieldsToCreateOrder": [{"key": "blockchainWalletAddress", "label": "Your wallet address", "required": true, "type": "string"}]
  }
}
```
</details>

To receive 100 POLYGON_USDT, user must deposit 153128 NGN. Collect these fields:

- phoneNumber
- bankCode (from enum options)
- bankAccountNumber
- blockchainWalletAddress

Create the order via [Create Order](#create-order):

<details>
<summary>Example request</summary>

```json
{
  "quoteId": "68628fa56ff494df5f39faf5",
  "userEmail": "someuser@example.com",
  "userIp": "174.3.2.22",
  "deposit": {"paymentChannel": "bank", "currencyType": "fiat", "currencyCode": "NGN"},
  "payout": {"paymentChannel": "crypto", "currencyType": "crypto", "currencyCode": "POLYGON_USDT", "amount": 100},
  "fieldsToCreateOrder": {
    "phoneNumber": "2348012345678",
    "bankCode": "120001:02",
    "bankAccountNumber": "1234567890",
    "blockchainWalletAddress": "0x5b7ae3c6c83F4A3F94b35c77233b13191eBGAD21"
  }
}
```
</details>

<details>
<summary>Example response (transfer instructions excerpt)</summary>

```typescript
const response = {
  "order": {
    "_id": "68728fa56ff494df5f39faf5",
    "countryIsoCode": "NG",
    "userId": "57a28fa56ff494df5f39faf5",
    "userEmail": "someuser@example.com",
    "status": "deposit_awaiting",
    "deposit": {
      //...
      transferInstructions: {
        type: "manual",
        instructionsText: "Please use the following bank details to make a transfer...",
        warningText: "Make sure to include the reference code in your transfer.",
        transferDetails: [
          { id: "recipientBankName", label: "Bank Name", value: "9Payment Service Bank" },
          { id: "recipientBankAccountNumber", label: "Account Number", value: "1234567890" },
          { id: "recipientBankAccountName", label: "Account Name", value: "Example Company Ltd" },
          { id: "amountToSend", label: "Amount to Send", value: "153128" },
          { id: "bankTransferNarration", label: "Transfer Narration / Reference", value: "ORDER-5F8D0D55B54764421B7156C5", description: "Use this as the transfer reference." }
        ],
        fieldsToConfirmOrder: [],
      }
    },
    "payout": {
      //...
    }
  }
}
```
</details>

User makes the transfer with the exact amount and reference. Then call [Confirm order](#confirm-order) if no extra fields are required:

<details>
<summary>Example request</summary>

```json
{ "orderId": "68728fa56ff494df5f39faf5" }
```
</details>

The system validates the deposit and processes payout. Use [Get Order](#get-order) to track status until "payout_successful".

## Transfer types explanation

We support the following transfer types for deposits:

- manual – user manually makes a transfer with provided instructions
- redirect – user is redirected to a third-party payment page to complete payment
- ussd – user dials a USSD code on their phone
- stk_push – user receives a push on their phone to approve the payment
- otp_stk_push – stk_push that requires providing an OTP code to initiate

Transfer instructions format differs by type:

- manual – includes transferDetails for making the transfer
- redirect – includes paymentUrl to redirect the user
- ussd – includes ussdCode to dial
- stk_push – includes intermediate action metadata:
  - intermediateActionMaxAttempts
  - intermediateActionAttempts
  - intermediateActionNextAttemptAvailableAt
  - intermediateActionTimeoutMs
- otp_stk_push – same as stk_push plus fieldsForIntermediateAction (e.g., otpCode)

Cross-links:
- For stk_push/otp_stk_push retries or OTP submission, see [Trigger intermediate action](#trigger-intermediate-action).
- For cases where confirmation fields are required post-deposit, see [Confirm order](#confirm-order).

## Authentication & Request Signing

Environments:
- Sandbox https://sandbox-api.fonbnk.com
- Production https://api.fonbnk.com

All requests are signed with HMAC-SHA256 using your clientId and clientSecret.

How to compute signature:
- timestamp = Unix epoch in milliseconds
- stringToSign = `${timestamp}:${endpoint}` where endpoint includes the path and query string (e.g., `/api/v2/liquidity-bridge/order-limits?foo=bar`)
- key = Base64-decoded clientSecret
- signature = Base64(HMAC-SHA256(key, UTF8(stringToSign)))
- Send headers: x-client-id, x-timestamp, x-signature

<details>
<summary>Pseudocode</summary>

```pseudocode
timestamp = CurrentTimestamp();
stringToSign = timestamp + ":" + endpoint;
signature = Base64 ( HMAC-SHA256 ( Base64-Decode ( clientSecret ), UTF8 ( concatenatedString ) ) );
```
</details>

<details>
<summary>TypeScript example</summary>

```typescript
import crypto from 'crypto';
const BASE_URL = 'https://api.fonbnk.com';
const ENDPOINT = '/api/v2/liquidity-bridge/order-limits';
const CLIENT_ID = '';
const CLIENT_SECRET = '';

const generateSignature = ({
  clientSecret,
  timestamp,
  endpoint,
}: {
  clientSecret: string;
  timestamp: string;
  endpoint: string;
}) => {
  let hmac = crypto.createHmac('sha256', Buffer.from(clientSecret, 'base64'));
  let stringToSign = `${timestamp}:${endpoint}`;
  hmac.update(stringToSign);
  return hmac.digest('base64');
};

const main = async () => {
  const timestamp = new Date().getTime();
  const queryParams = new URLSearchParams({
    depositPaymentChannel: 'bank',
    depositCurrencyType: 'fiat',
    depositCurrencyCode: 'NGN',
    payoutPaymentChannel: 'crypto',
    payoutCurrencyType: 'crypto',
    payoutCurrencyCode: 'POLYGON_USDT',
  });
  const endpoint = `${ENDPOINT}?${queryParams.toString()}`;
  const signature = generateSignature({
    clientSecret: CLIENT_SECRET,
    timestamp: timestamp.toString(),
    endpoint,
  });
  const headers = {
    'Content-Type': 'application/json',
    'x-client-id': CLIENT_ID,
    'x-timestamp': timestamp.toString(),
    'x-signature': signature,
  };
  const response = await fetch(`${BASE_URL}${endpoint}`, {
    method: 'GET',
    headers,
  });
  const data = await response.json();
  console.log(JSON.stringify(data, null, 2));
};

main().catch(console.error);
```
</details>

## API endpoints

### Get currencies

_**GET** /api/v2/liquidity-bridge/currencies_

Returns supported currencies for deposit and payout with details and available pairs.

Response type:

```typescript
type CurrenciesResponse = {
  currencyType: CurrencyType;
  currencyCode: string;
  paymentChannels: {
    type: PaymentChannel,
    transferTypes: TransferType[],
    isDepositAllowed: boolean,
    isPayoutAllowed: boolean,
    carriers?: { code: string; name: string; }[]
  }[];
  currencyDetails: OrderCurrencyDetails;
  pairs: (keyof typeof CurrencyType)[] | (CurrencyType)[]; // available counter currency types
}[]
```

<details>
<summary>Response example</summary>

```typescript
const response = [
  {
    currencyType: "fiat",
    currencyCode: "NGN",
    paymentChannels: [
      { type: "bank", transferTypes: ["manual", "redirect"], isDepositAllowed: true, isPayoutAllowed: true },
      { type: "airtime", transferTypes: ["ussd"], carriers: [
        {code: "MTN", name: "MTN"},
        {code: "AIRTEL", name: "Airtel"},
        {code: "GLO", name: "Glo"},
        {code: "9MOBILE", name: "9Mobile"},
      ], isDepositAllowed: true, isPayoutAllowed: false },
      { type: "mobile_money", transferTypes: ["stk_push", "otp_stk_push"], carriers: [
        {code: "MTN", name: "MTN Mobile Money"},
        {code: "AIRTEL", name: "Airtel Money"},
        {code: "GLO", name: "Glo Mobile Money"},
        {code: "9MOBILE", name: "9Mobile Money"},
      ], isDepositAllowed: true, isPayoutAllowed: true }
    ],
    currencyDetails: { countryIsoCode: "NG", countryName: "Nigeria", countryCode: "234", currencySymbol: "₦", countryIcon: "https://cdn.example.com/flags/ng.png" },
    pairs: ["crypto", "merchant_balance"]
  },
  {
    currencyType: "crypto",
    currencyCode: "POLYGON_USDT",
    paymentChannels: [
      { type: "crypto", transferTypes: ["manual"], isDepositAllowed: true, isPayoutAllowed: true }
    ],
    currencyDetails: { network: "POLYGON", asset: "USDT", networkTitle: "Polygon", assetTitle: "USDT", contractAddress: "0xc2132D05D31c914a87C6611C10748AEb04B58e8F", decimals: 6, networkIcon: "https://cdn.example.com/networks/polygon.png", assetIcon: "https://cdn.example.com/assets/usdt.png" },
    pairs: ["fiat", "merchant_balance"]
  },
  {
    currencyType: "merchant_balance",
    currencyCode: "USD",
    paymentChannels: [ { type: "merchant_balance", transferTypes: [], isDepositAllowed: true, isPayoutAllowed: true } ],
    currencyDetails: { merchantName: "Example Company Ltd" },
    pairs: ["fiat", "crypto"]
  }
]
```
</details>

### Get order limits

_**GET** /api/v2/liquidity-bridge/order-limits_

Returns min and max order limits for a deposit and payout currency pair.

Query params:

- depositPaymentChannel: string (required)
- depositCurrencyType: string (required)
- depositCurrencyCode: string (required)
- depositCarrierCode: string (optional)
- payoutPaymentChannel: string (required)
- payoutCurrencyType: string (required)
- payoutCurrencyCode: string (required)
- payoutCarrierCode: string (optional)

Response type:

```typescript
type OrderLimitsResponse = {
  deposit: { min: number; max: number; minUsd: number; maxUsd: number },
  payout: { min: number; max: number; minUsd: number; maxUsd: number },
}
```

<details>
<summary>Response example</summary>

```typescript
const queryParams = {
  depositPaymentChannel: "bank",
  depositCurrencyType: "fiat",
  depositCurrencyCode: "NGN",
  payoutPaymentChannel: "crypto",
  payoutCurrencyType: "crypto",
  payoutCurrencyCode: "POLYGON_USDT",
}

const response = {
  deposit: { min: 1556, max: 311184, minUsd: 1, maxUsd: 200 },
  payout: { min: 1, max: 200, minUsd: 1, maxUsd: 200 },
}
```
</details>

### Get Quote

_**POST** /api/v2/liquidity-bridge/quote_

Returns a pricing quote for a deposit and payout pair. Provide either deposit.amount or payout.amount (not both).

Request body:

- deposit.paymentChannel: string (required)
- deposit.currencyType: string (required)
- deposit.currencyCode: string (required)
- deposit.amount: number (optional)
- deposit.carrierCode: string (optional)
- payout.paymentChannel: string (required)
- payout.currencyType: string (required)
- payout.currencyCode: string (required)
- payout.amount: number (optional)
- payout.carrierCode: string (optional)

Response type:

```typescript
type QuoteResponse = {
  quoteId: string;
  quoteExpiresAt: Date;
  deposit: {
    paymentChannel: PaymentChannel;
    currencyType: CurrencyType;
    currencyCode: string;
    currencyDetails: OrderCurrencyDetails;
    cashout: Cashout;
    fieldsToCreateOrder: RequiredField[];
    transferType: TransferType;
  },
  payout: {
    paymentChannel: PaymentChannel;
    currencyType: CurrencyType;
    currencyCode: string;
    currencyDetails: OrderCurrencyDetails;
    cashout: Cashout;
    fieldsToCreateOrder: RequiredField[];
  }
}
```

<details>
<summary>Request + Response example</summary>

```typescript
const requestBody = {
  deposit: { paymentChannel: "bank", currencyType: "fiat", currencyCode: "NGN", amount: 10000 },
  payout: { paymentChannel: "crypto", currencyType: "crypto", currencyCode: "POLYGON_USDT" },
}

const response = {
  quoteId: "68628fa56ff494df5f39faf5",
  deposit: {
    paymentChannel: "bank",
    currencyType: "fiat",
    currencyCode: "NGN",
    currencyDetails: {
      countryIsoCode: "NG",
      countryName: "Nigeria",
      countryCode: "234",
      currencySymbol: "₦",
      countryIcon: "https://cdn.example.com/flags/ng.png",
    },
    cashout: {
      exchangeRate: 1500,
      exchangeRateAfterFees: 1538.46,
      amountBeforeFees: 10000,
      amountAfterFees: 9751,
      amountBeforeFeesUsd: 6.67,
      amountAfterFeesUsd: 6.5,
      feeSettings: [
        { id: "provider_fee", recipient: "provider", type: "flat_amount", value: 50, min: 0, max: "Infinity" },
        { id: "platform_fee", recipient: "platform", type: "percentage", value: 2, min: 0, max: "Infinity" }
      ],
      chargedFees: [
        { id: "provider_fee", type: "flat_amount", recipient: "provider", amount: 50 },
        { id: "platform_fee", type: "percentage", recipient: "platform", amount: 199 }
      ],
      chargedFeesUsd: [
        { id: "provider_fee", type: "flat_amount", recipient: "provider", amount: 0.03 },
        { id: "platform_fee", type: "percentage", recipient: "platform", amount: 0.13 }
      ],
      totalChargedFees: 249,
      totalChargedFeesUsd: 0.16,
      chargedFeesPerRecipient: { provider: 50, platform: 199 },
      chargedFeesPerRecipientUsd: { provider: 0.03, platform: 0.13 },
    },
    fieldsToCreateOrder: [
      { key: "phoneNumber", label: 'Phone Number', required: true, type: "phone" },
      { key: "bankCode", label: 'Bank name', required: true, type: "enum", options: [
        { "value": "120001:02", "label": "9Payment Service Bank" },
        { "value": "801:02", "label": "Abbey Mortgage Bank" }
      ] },
      { key: "bankAccountNumber", label: 'Bank Account Number', required: true, type: "string" },
    ],
    transferType: "manual",
  },
  payout: {
    paymentChannel: "crypto",
    currencyType: "crypto",
    currencyCode: "POLYGON_USDT",
    currencyDetails: {
      network: "POLYGON",
      asset: "USDT",
      networkTitle: "Polygon",
      assetTitle: "USDT",
      contractAddress: "0xc2132D05D31c914a87C6611C10748AEb04B58e8F",
      decimals: 6,
      networkIcon: "https://cdn.example.com/networks/polygon.png",
      assetIcon: "https://cdn.example.com/assets/usdt.png",
    },
    cashout: {
      exchangeRate: 1,
      exchangeRateAfterFees: 1.0015,
      amountBeforeFees: 6.5,
      amountAfterFees: 6.49,
      amountBeforeFeesUsd: 6.5,
      amountAfterFeesUsd: 6.49,
      feeSettings: [ { id: "gas_fee", recipient: "blockchain", type: "flat_amount", value: 0.01, min: 0, max: "Infinity" } ],
      chargedFees: [ { id: "gas_fee", type: "flat_amount", recipient: "blockchain", amount: 0.01 } ],
      chargedFeesUsd: [ { id: "gas_fee", type: "flat_amount", recipient: "blockchain", amount: 0.01 } ],
      totalChargedFees: 0.01,
      totalChargedFeesUsd: 0.01,
      chargedFeesPerRecipient: { blockchain: 0.01 },
      chargedFeesPerRecipientUsd: { blockchain: 0.01 },
    },
    fieldsToCreateOrder: [ { key: "blockchainWalletAddress", label: 'Your wallet address', required: true, type: "string" } ],
  }
}
```
</details>

> Note: Always honor quoteExpiresAt. If a quote expires, request a new one before creating the order.

### Create Order

_**POST** /api/v2/liquidity-bridge/order_

Creates an order to process a deposit and payout based on a quote if provided.

Request body:

```typescript
type CreateOrderRequest = {
  quoteId?: string;
  userEmail: string;
  userIp: string;
  deposit: {
    paymentChannel: PaymentChannel;
    currencyType: CurrencyType;
    currencyCode: string;
    carrierCode?: string;
    amount?: number;
  },
  payout: {
    paymentChannel: PaymentChannel;
    currencyType: CurrencyType;
    currencyCode: string;
    carrierCode?: string;
    amount?: number;
  };
  fieldsToCreateOrder: Record<string, any>; // union of required fields from deposit and payout
  redirectUrl?: string; // for redirect transfer type
  orderParams?: string; // merchant-defined reference
  callbackUrl?: string; // button URL on status page in the widget
}
```

Response type:

```typescript
type CreateOrderResponse = {
  order: {
    _id: string,
    countryIsoCode: string;
    userId: string;
    userEmail: string;
    merchantOrderParams?: string;
    status: OrderStatus;
    deposit: {
      paymentChannel: PaymentChannel;
      currencyType: CurrencyType;
      currencyCode: string;
      currencyDetails: OrderCurrencyDetails;
      cashout: Cashout;
      fieldsToCreateOrder: RequiredField[];
      providedFieldsToCreateOrder: Record<string, string>;
      transferInstructions: TransferInstructions;
    },
    payout: {
      paymentChannel: PaymentChannel;
      currencyType: CurrencyType;
      currencyCode: string;
      currencyDetails: OrderCurrencyDetails;
      cashout: Cashout;
      fieldsToCreateOrder: RequiredField[];
      providedFieldsToCreateOrder: Record<string, string>;
    },
    createdAt: Date;
    updatedAt: Date;
    statusChangeHistory: { oldStatus: OrderStatus; newStatus: OrderStatus; date: Date; }[]
  },
  quoteUsed: boolean,
}
```

<details>
<summary>Request + Response example</summary>

```typescript
const requestBody = {
  quoteId: "68628fa56ff494df5f39faf5",
  userEmail: "user@example.com",
  userIp: "143.0.2.4",
  deposit: { paymentChannel: "bank", currencyType: "fiat", currencyCode: "NGN", amount: 10000 },
  payout: { paymentChannel: "crypto", currencyType: "crypto", currencyCode: "POLYGON_USDT" },
  fieldsToCreateOrder: {
    blockchainWalletAddress: "0x5b7ae3c6c83F4A3F94b35c77233b13191eBGAD21",
    phoneNumber: "2348012345678",
    bankCode: "120001:02",
    bankAccountNumber: "1234567890",
  }
}

const response = {
  order: {
    // see Get Order response example below
  },
  quoteUsed: true,
}
```
</details>

### Get user KYC state

_**GET** /api/v2/liquidity-bridge/user/kyc_

Returns the KYC state of a user. If the user doesn’t exist, it is created and the KYC state is returned. Also returns KYC rules and available documents for the user’s country.

Query params:

- userEmail: string (required)
- countryIsoCode: string (required)

Response type:

```typescript
type GetUserKycResponse = {
  passedKycType?: KycType;
  reachedKycLimit: boolean;
  currentKycStatus?: KycStatus;
  currentKycStatusDescription?: string;
  kycDocuments: KycDocument[],
  kycRules: {
    operationType: OperationType;
    currencyType: CurrencyType;
    min: number; // USD
    max: number | 'Infinity';
    type: KycType;
  }[]
}
```

<details>
<summary>Response example</summary>

```typescript
const response = {
  passedKycType: "basic",
  reachedKycLimit: false,
  currentKycStatus: "approved",
  currentKycStatusDescription: "Partial Match",
  kycDocuments: [
    {
      "_id": "67da909b739fc481aa525c43",
      "type": "basic",
      "title": "Voter ID",
      "value": "VOTER_ID",
      "requiredFields": [
        {"key":"first_name","type":"string","label":"First Name","required":true},
        {"key":"last_name","type":"string","label":"Last Name","required":true},
        {"key":"dob","type":"date","label":"Date of birth","required":true},
        {"key":"id_number","type":"string","label":"ID number","required":true,"format":"0000000000000000000","regexp":"^[a-zA-Z0-9 ]{9,29}$","regexpFlags":"i"}
      ]
    },
    {
      "_id": "67da93c0dfd3a00f3380b857",
      "type": "advanced",
      "title": "Driving License",
      "value": "DRIVERS_LICENSE",
      "requiredFields": [
        {"key":"first_name","type":"string","label":"First Name","required":true},
        {"key":"last_name","type":"string","label":"Last Name","required":true},
        {"key":"dob","type":"date","label":"Date of birth","required":true},
        {"key":"images","type":"smile-identity-images","label":"Verification images","required":true}
      ]
    },
  ],
  kycRules: [
    { operationType: 'deposit', currencyType: 'crypto', min: 0, max: 100, type: "basic" },
    { operationType: 'deposit', currencyType: 'crypto', min: 100, max: 'Infinity', type: "basic" },
    { operationType: 'payout', currencyType: 'crypto', min: 0, max: 'Infinity', type: "basic" },
  ]
}
```
</details>

### Submit user KYC

_**POST** /api/v2/liquidity-bridge/user/kyc_

Submits KYC documents for a user. Returns the same structure as [Get user KYC state](#get-user-kyc-state).

Request body:

```typescript
type SubmitUserKycRequest = {
  userEmail: string;
  countryIsoCode: string;
  documentId: string;
  userFields: Record<string, any>;
}
```

<details>
<summary>Request example</summary>

```typescript
const requestBody = {
  userEmail: "user@example.com",
  countryIsoCode: "NG",
  documentId: "67da909b739fc481aa525c43",
  userFields: {
    first_name: "John",
    last_name: "Doe",
    dob: "1990-01-01",
    id_number: "A123456789",
  }
}
```
</details>

### Trigger intermediate action

_**POST** /api/v2/liquidity-bridge/order/intermediate-action_

Triggers an intermediate action for a deposit order (e.g., STK Push or OTP STK Push). Must be called within the timeout and before max attempts are reached.

Request body:

```typescript
type TriggerIntermediateActionRequest = {
  orderId: string;
  fieldsForIntermediateAction: Record<string, string>;
}
```

<details>
<summary>Request example</summary>

```json
{
  "orderId": "68728fa56ff494df5f39faf5",
  "fieldsForIntermediateAction": { "otpCode": "123456" }
}
```
</details>

Returns the same structure as [Get Order](#get-order).

### Confirm order

_**POST** /api/v2/liquidity-bridge/order/confirm_

Confirms a deposit order. If transferInstructions.fieldsToConfirmOrder is non-empty, include them.

Request body:

```typescript
type ConfirmOrderRequest = {
  orderId: string;
  fieldsToConfirmOrder?: Record<string, string>;
}
```

<details>
<summary>Minimal request</summary>

```json
{ "orderId": "68728fa56ff494df5f39faf5" }
```
</details>

Returns the same structure as [Get Order](#get-order).

### Cancel order

_**POST** /api/v2/liquidity-bridge/order/cancel_

Cancels a deposit order if it is still in a cancellable state.

Request body:

```typescript
type CancelOrderRequest = { orderId: string }
```

Returns the same structure as [Get Order](#get-order).

### Get Order

_**GET** /api/v2/liquidity-bridge/order_

Retrieves an order by its ID or merchant order params.

Query params:

- orderId?: string
- orderParams?: string

Response type:

```typescript
type GetOrderResponse = {
  _id: string,
  countryIsoCode: string;
  userId: string;
  userEmail: string;
  merchantOrderParams?: string;
  status: OrderStatus;
  deposit: {
    paymentChannel: PaymentChannel;
    currencyType: CurrencyType;
    currencyCode: string;
    currencyDetails: OrderCurrencyDetails;
    cashout: Cashout;
    fieldsToCreateOrder: RequiredField[];
    providedFieldsToCreateOrder: Record<string, string>;
    transferInstructions: TransferInstructions;
  },
  payout: {
    paymentChannel: PaymentChannel;
    currencyType: CurrencyType;
    currencyCode: string;
    currencyDetails: OrderCurrencyDetails;
    cashout: Cashout;
    fieldsToCreateOrder: RequiredField[];
    providedFieldsToCreateOrder: Record<string, string>;
  },
  createdAt: Date;
  updatedAt: Date;
  statusChangeHistory: { oldStatus: OrderStatus; newStatus: OrderStatus; date: Date; }[]
}
```

<details>
<summary>Response example</summary>

```typescript
const response = {
  _id: "5f8d0d55b54764421b7156c5",
  countryIsoCode: "NG",
  userId: "68628fa56ff494df5f39faf6",
  userEmail: "user@example.com",
  merchantOrderParams: "order-12345",
  status: "payout_pending",
  deposit: {
    paymentChannel: "bank",
    currencyType: "fiat",
    currencyCode: "NGN",
    currencyDetails: {
      countryIsoCode: "NG",
      countryName: "Nigeria",
      countryCode: "234",
      currencySymbol: "₦",
      countryIcon: "https://cdn.example.com/flags/ng.png",
    },
    cashout: {
      exchangeRate: 1500,
      exchangeRateAfterFees: 1538.46,
      amountBeforeFees: 10000,
      amountAfterFees: 9751,
      amountBeforeFeesUsd: 6.67,
      amountAfterFeesUsd: 6.5,
      feeSettings: [
        { id: "provider_fee", recipient: "provider", type: "flat_amount", value: 50, min: 0, max: "Infinity" },
        { id: "platform_fee", recipient: "platform", type: "percentage", value: 2, min: 0, max: "Infinity" }
      ],
      chargedFees: [
        { id: "provider_fee", type: "flat_amount", recipient: "provider", amount: 50 },
        { id: "platform_fee", type: "percentage", recipient: "platform", amount: 199 }
      ],
      chargedFeesUsd: [
        { id: "provider_fee", type: "flat_amount", recipient: "provider", amount: 0.03 },
        { id: "platform_fee", type: "percentage", recipient: "platform", amount: 0.13 }
      ],
      totalChargedFees: 249,
      totalChargedFeesUsd: 0.16,
      chargedFeesPerRecipient: { provider: 50, platform: 199 },
      chargedFeesPerRecipientUsd: { provider: 0.03, platform: 0.13 },
    },
    fieldsToCreateOrder: [
      { key: "phoneNumber", label: 'Phone Number', required: true, type: "phone" },
      { key: "bankCode", label: 'Bank name', required: true, type: "enum", options: [
        { "value": "120001:02", "label": "9Payment Service Bank" },
        { "value": "801:02", "label": "Abbey Mortgage Bank" }
      ] },
      { key: "bankAccountNumber", label: 'Bank Account Number', required: true, type: "string" },
    ],
    providedFieldsToCreateOrder: {
      phoneNumber: "2348012345678",
      bankCode: "120001:02",
      bankAccountNumber: "1234567890",
    },
    transferInstructions: {
      type: "manual",
      instructionsText: "Please use the following bank details to make a transfer...",
      warningText: "Make sure to include the reference code in your transfer.",
      transferDetails: [
        { id: "recipientBankName", label: "Bank Name", value: "9Payment Service Bank" },
        { id: "recipientBankAccountNumber", label: "Account Number", value: "1234567890" },
        { id: "recipientBankAccountName", label: "Account Name", value: "Example Company Ltd" },
        { id: "amountToSend", label: "Amount to Send", value: "10000" },
        { id: "bankTransferNarration", label: "Transfer Narration / Reference", value: "ORDER-5F8D0D55B54764421B7156C5", description: "Use this as the transfer reference." }
      ],
      fieldsToConfirmOrder: [],
    }

  },
  payout: {
    paymentChannel: "crypto",
    currencyType: "crypto",
    currencyCode: "POLYGON_USDT",
    currencyDetails: {
      network: "POLYGON",
      asset: "USDT",
      networkTitle: "Polygon",
      assetTitle: "USDT",
      contractAddress: "0xc2132D05D31c914a87C6611C10748AEb04B58e8F",
      decimals: 6,
      networkIcon: "https://cdn.example.com/networks/polygon.png",
      assetIcon: "https://cdn.example.com/assets/usdt.png",
    },
    cashout: {
      exchangeRate: 1,
      exchangeRateAfterFees: 1.0015,
      amountBeforeFees: 6.5,
      amountAfterFees: 6.49,
      amountBeforeFeesUsd: 6.5,
      amountAfterFeesUsd: 6.49,
      feeSettings: [ { id: "gas_fee", recipient: "blockchain", type: "flat_amount", value: 0.01, min: 0, max: "Infinity" } ],
      chargedFees: [ { id: "gas_fee", type: "flat_amount", recipient: "blockchain", amount: 0.01 } ],
      chargedFeesUsd: [ { id: "gas_fee", type: "flat_amount", recipient: "blockchain", amount: 0.01 } ],
      totalChargedFees: 0.01,
      totalChargedFeesUsd: 0.01,
      chargedFeesPerRecipient: { blockchain: 0.01 },
      chargedFeesPerRecipientUsd: { blockchain: 0.01 },
    },
    fieldsToCreateOrder: [ { key: "blockchainWalletAddress", label: 'Your wallet address', required: true, type: "string" } ],
    providedFieldsToCreateOrder: {
      blockchainWalletAddress: "0x5b7ae3c6c83F4A3F94b35c77233b13191eBGAD21",
    }
  },
  createdAt: new Date("2025-10-01T12:00:00Z"),
  updatedAt: new Date("2025-10-01T12:00:00Z"),
  statusChangeHistory: [
    { oldStatus: "deposit_awaiting", newStatus: "deposit_validating", date: new Date("2025-10-01T12:05:00Z") },
    { oldStatus: "deposit_validating", newStatus: "deposit_successful", date: new Date("2025-10-01T12:10:00Z") },
    { oldStatus: "deposit_successful", newStatus: "payout_pending", date: new Date("2025-10-01T12:15:00Z") }
  ]
}
```
</details>

### Get orders

_**GET** /api/v2/liquidity-bridge/orders_

Retrieves a list of orders with cursor pagination and optional filters.

Query params:
- cursor: string (optional)
- limit: number (optional, 1–100)
- userEmail: string (optional)
- status: OrderStatus (optional)
- fromDate: string (optional, unix ms)
- toDate: string (optional, unix ms)
- depositCurrencyCode: string (optional)
- depositPaymentChannel: PaymentChannel (optional)
- depositCurrencyType: CurrencyType (optional)
- payoutCurrencyCode: string (optional)
- payoutPaymentChannel: PaymentChannel (optional)
- payoutCurrencyType: CurrencyType (optional)
- depositUserWalletAddress: string (optional)
- payoutUserWalletAddress: string (optional)
- depositUserPhoneNumber: string (optional)
- payoutUserPhoneNumber: string (optional)

Response type:

```typescript
type GetOrdersResponse = {
  nextCursor?: string;
  list: GetOrderResponse[];
}
```

> Pagination: pass nextCursor from the previous response to fetch the next page. Omit cursor to start from the beginning.

### Get merchant balance

_**GET** /api/v2/liquidity-bridge/merchant-balance_

Returns the current merchant balance (currently only in USD).

Response type:

```typescript
type MerchantBalanceResponse = { USD: number }
```

## Types used in the above definitions:

<details>
<summary>Expand all TypeScript types</summary>

```typescript

enum PaymentChannel {
  BANK = 'bank',
  AIRTIME = 'airtime',
  MOBILE_MONEY = 'mobile_money',
  PAYBILL = 'paybill',
  BUY_GOODS = 'buy_goods',
  MERCHANT_BALANCE = 'merchant_balance',
  CRYPTO = 'crypto',
}

enum CurrencyType {
  FIAT = 'fiat',
  CRYPTO = 'crypto',
  MERCHANT_BALANCE = 'merchant_balance',
}

type OrderCurrencyDetails =
  | OrderCryptoDetails
  | OrderFiatDetails
  | OrderMerchantDetails;

type OrderCryptoDetails = {
  network: string;
  asset: string;
  networkTitle: string;
  assetTitle: string;
  contractAddress: string;
  decimals: number;
  networkIcon: string;
  assetIcon: string;
};

type OrderFiatDetails = {
  countryIsoCode: string;
  countryName: string;
  countryCode: string;
  currencySymbol: string;
  countryIcon: string;
  carriers?: { code: string; name: string; }[];
}

type OrderMerchantDetails = { merchantName: string }

type Cashout = {
  exchangeRate: number;
  feeSettings: FeeSetting[];
  exchangeRateAfterFees: number;
  amountBeforeFees: number;
  amountAfterFees: number;
  amountBeforeFeesUsd: number;
  amountAfterFeesUsd: number;
  chargedFees: ChargedFee[];
  chargedFeesUsd: ChargedFee[];
  totalChargedFees: number;
  totalChargedFeesUsd: number;
  chargedFeesPerRecipient: Partial<Record<FeeRecipient, number>>;
  chargedFeesPerRecipientUsd: Partial<Record<FeeRecipient, number>>;
};

type FeeSetting =
  | { id: string; recipient: FeeRecipient; type: FeeType.FLAT_AMOUNT; value: number; min: number; max: number | 'Infinity' }
  | { id: string; recipient: FeeRecipient; type: FeeType.PERCENTAGE; value: number; min: number; max: number | 'Infinity'; minCap?: number; maxCap?: number };

type ChargedFee = { id: string; type: FeeType; recipient: FeeRecipient; amount: number };

enum FeeRecipient { MERCHANT = 'merchant', PROVIDER = 'provider', PLATFORM = 'platform', BLOCKCHAIN = 'blockchain' }

enum FeeType { PERCENTAGE = 'percentage', FLAT_AMOUNT = 'flat_amount' }

type TransferInstructions =
  | UssdTransferInstructions
  | ManualTransferInstructions
  | RedirectTransferInstructions
  | StkPushTransferInstructions
  | OtpStkPushTransferInstructions;

type UssdTransferInstructions = {
  type: TransferType.USSD;
  ussdCode: string;
  instructionsText: string;
  warningText?: string;
  transferDetails: TransferDetail[];
  fieldsToConfirmOrder?: RequiredField[];
};

type ManualTransferInstructions = {
  type: TransferType.MANUAL;
  instructionsText: string;
  warningText?: string;
  transferDetails: TransferDetail[];
  fieldsToConfirmOrder: RequiredField[];
};

type RedirectTransferInstructions = {
  type: TransferType.REDIRECT;
  paymentUrl: string;
  redirectedToPaymentUrl: boolean;
  intermediateActionButtonText: string;
  instructionsText: string;
  warningText?: string;
  transferDetails: TransferDetail[];
  fieldsToConfirmOrder: RequiredField[];
};

type StkPushTransferInstructions = {
  type: TransferType.STK_PUSH;
  isIntermediateActionAvailable: boolean;
  intermediateActionButtonText: string;
  intermediateActionMaxAttempts: number;
  intermediateActionAttempts: number;
  intermediateActionNextAttemptAvailableAt: Date;
  intermediateActionTimeoutMs: number;
  instructionsText: string;
  warningText?: string;
  transferDetails: TransferDetail[];
  fieldsToConfirmOrder: RequiredField[];
};

type OtpStkPushTransferInstructions = {
  type: TransferType.OTP_STK_PUSH;
  isIntermediateActionAvailable: boolean;
  intermediateActionButtonText: string;
  intermediateActionMaxAttempts: number;
  intermediateActionAttempts: number;
  intermediateActionNextAttemptAvailableAt: Date;
  intermediateActionTimeoutMs: number;
  fieldsForIntermediateAction: RequiredField[];
  instructionsText: string;
  warningText?: string;
  transferDetails: TransferDetail[];
  fieldsToConfirmOrder: RequiredField[];
};

enum TransferType { MANUAL = 'manual', REDIRECT = 'redirect', STK_PUSH = 'stk_push', OTP_STK_PUSH = 'otp_stk_push', USSD = 'ussd' }

type TransferDetail = { id: TransferDetailId; label: string; description?: string; value?: string };

enum TransferDetailId {
  RECIPIENT_WALLET_ADDRESS = 'recipientWalletAddress',
  SENDER_WALLET_ADDRESS = 'senderWalletAddress',
  AMOUNT_TO_SEND = 'amountToSend',
  CRYPTO_TRANSACTION_REQUEST_ADDITIONAL_DATA = 'cryptoTransactionRequestAdditionalData',
  RECIPIENT_BANK_NAME = 'recipientBankName',
  RECIPIENT_BANK_ACCOUNT_NUMBER = 'recipientBankAccountNumber',
  RECIPIENT_BANK_ACCOUNT_NAME = 'recipientBankAccountName',
  RECIPIENT_PHONE_NUMBER = 'recipientPhoneNumber',
  BANK_TRANSFER_NARRATION = 'bankTransferNarration',
}

type RequiredField = {
  key: string;
  type: FieldType;
  label: string;
  required: boolean;
  options?: { value: string; label: string }[];
  defaultValue?: string;
};

enum FieldType { NUMBER = 'number', STRING = 'string', DATE = 'date', BOOLEAN = 'boolean', EMAIL = 'email', PHONE = 'phone', ENUM = 'enum' }

enum OrderStatus {
  DEPOSIT_AWAITING = 'deposit_awaiting',
  DEPOSIT_VALIDATING = 'deposit_validating',
  DEPOSIT_INVALID = 'deposit_invalid',
  DEPOSIT_SUCCESSFUL = 'deposit_successful',
  DEPOSIT_CANCELED = 'deposit_canceled',
  PAYOUT_PENDING = 'payout_pending',
  PAYOUT_SUCCESSFUL = 'payout_successful',
  PAYOUT_FAILED = 'payout_failed',
  REFUNDING_INITIATED = 'refunding_initiated',
  REFUNDING_PENDING = 'refunding_pending',
  REFUNDING_SUCCESSFUL = 'refunding_successful',
  REFUNDING_FAILED = 'refunding_failed',
  EXPIRED = 'expired',
}

enum KycType { BASIC = 'basic', ADVANCED = 'advanced' }

enum KycStatus { INITIATED = 'initiated', APPROVED = 'approved', REJECTED = 'rejected', INVALID = 'invalid' }

type KycDocument = { _id: string; title: string; value: string; type: KycType; requiredFields: DynamicField[] }

type DynamicField =
  | { key: string; type: 'number' | 'string' | 'date' | 'boolean' | 'email' | 'phone' | 'smile-identity-images'; label: string; required: boolean; defaultValue?: string | number | boolean; regexp?: string; regexpFlags?: string; format?: string }
  | { key: string; type: 'enum'; label: string; required: boolean; options: { value: string; label: string }[]; defaultValue?: string; regexp?: string; regexpFlags?: string; format?: string };

enum OperationType { DEPOSIT = 'deposit', PAYOUT = 'payout' }

```
</details>
