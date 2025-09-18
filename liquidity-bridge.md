# Liquidity Bridge API Documentation (WIP)

## Overview

This is a draft API for V2 of the current Fonbnk API for merchants.

In the previous version we had a distinction between "onramp" and "offramp" endpoints with a separate set of endpoints
for each, different request and response formats, and different integrations for each.
In this new version we unify the concepts into a single "Liquidity Bridge" API, and we now operate with the concepts
of "deposit" and "payout" instead of "onramp" and "offramp".
The merchant has the access to the next currency types:

- Fiat (e.g., NGN, KES, GHS)
- Crypto (e.g., CELO_CUSD, TRON_USDT, POLYGON_USDT)
- Merchant Balance (at the moment only in USD)

Merchant then can create orders to deposit one currency type and payout another currency type, for example:

- Deposit Fiat (NGN) via Bank Transfer, Payout Crypto (POLYGON_USDT)
- Deposit Crypto (CELO_CUSD) via Crypto Transfer, Payout fiat mobile money (KES)
- Deposit Merchant Balance (USD) via Merchant Balance, Payout Crypto (TRON_USDT)
- Deposit Fiat (GHS) via Mobile Money, Payout Merchant Balance (USD)
- etc.

With the new architecture, a merchant has the access to:

- off-ramp
- on-ramp
- funding their merchant balance from any of the supported deposit methods
- withdrawing their merchant balance to any of the supported payout methods
- accepting crypto/fiat payments from users to their merchant balance and then using that balance to payout to any of
  the supported payout methods

## Order flow overview

1. Call "Get Currencies" endpoint to get the list of supported currencies and corresponding pairs
2. Call "Get Order Limits" endpoint to get the min and max limits for the selected deposit and payout currency pair.
3. Call "Get user KYC state" endpoint to check if the user needs to complete KYC based on the selected deposit and
   payout
   currency types and the order limits. If KYC is required, call "Submit user KYC" endpoint to submit the required KYC
   documents and wait for approval before proceeding.
4. Call "Get Quote" endpoint to get a pricing quote for the selected deposit and payout currency pair and amount. Get
   fieldsToCreateOrder from both deposit and payout to know what fields are required to create the order and ask the
   user for those fields.
5. Call "Create Order" endpoint with the quoteId from the previous step and the fields collected from the user to create
   the order.
6. The user makes the deposit based on the transfer instructions provided in the order response.
7. If the transfer type requires an intermediate action (init otp stk push by providing otp code or retry stk push), the
   merchant calls the "Trigger Intermediate Action" endpoint to trigger the action with the required fields if needed.
8. The merchant calls the "Confirm Order" endpoint to confirm the deposit if required.
9. The order status is updated as the deposit is validated and the payout is processed.
10. The merchant can call the "Get Order" endpoint to retrieve the order status and details.

## Fiat-to-Crypto Example Flow

Let's try to do a fiat-to-crypto order as an example.
At first, we need to get the list of supported currencies and pairs by calling the "Get Currencies" endpoint.

Let's say we received the following response:

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

We see that the NGN fiat currency supports deposit via bank, airtime, and mobile money, and payout via bank and mobile
money.
We also see that the POLYGON_USDT crypto currency supports both deposit and payout.
So we can create a fiat-to-crypto order with NGN as the deposit currency and POLYGON_USDT as the payout currency.

Next, we need to get the order limits for this currency pair by calling the "Get Order Limits" endpoint with the
following query params:

- depositPaymentChannel: "bank"
- depositCurrencyType: "fiat"
- depositCurrencyCode: "NGN"
- payoutPaymentChannel: "crypto"
- payoutCurrencyType: "crypto"
- payoutCurrencyCode: "POLYGON_USDT"

Let's say we received the following response:

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

We see that the minimum deposit amount is 1556 NGN and the maximum is 311184 NGN, which is equivalent to 1 to 200 USD.
Let's say the user wants to receive 100 POLYGON_USDT.

Next, we need to check the user's KYC state by calling the "Get user KYC state" endpoint with the following query
params:

- userEmail: "someuser@example.com"
- countryIsoCode: "NG"

Let's say we received the following response:

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
        {
          "key": "first_name",
          "type": "string",
          "label": "First Name",
          "required": true
        },
        {
          "key": "last_name",
          "type": "string",
          "label": "Last Name",
          "required": true
        },
        {
          "key": "dob",
          "type": "date",
          "label": "Date of birth",
          "required": true
        },
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
        {
          "key": "first_name",
          "type": "string",
          "label": "First Name",
          "required": true
        },
        {
          "key": "last_name",
          "type": "string",
          "label": "Last Name",
          "required": true
        },
        {
          "key": "dob",
          "type": "date",
          "label": "Date of birth",
          "required": true
        },
        {
          "key": "images",
          "type": "smile-identity-images",
          "label": "Verification images",
          "required": true
        }
      ]
    }
  ],
  "kycRules": [
    {
      "operationType": "deposit",
      "currencyType": "crypto",
      "min": 0,
      "max": 100,
      "type": "basic"
    },
    {
      "operationType": "payout",
      "currencyType": "crypto",
      "min": 100,
      "max": "Infinity",
      "type": "basic"
    }
  ]
}
```

We see that the user has not passed any KYC yet. We also see that the user needs to pass the advanced KYC to be able to
payout crypto amounts from 100 USD.
So we need to ask the user to submit an advanced KYC document. We must ask a user to provide the following fields:

- first_name
- last_name
- dob (date of birth, format: YYYY-MM-DD)
- images. It requires you to submit a photos of both sides of user's document and a user's selfie. Let's imagine that
  you took these photos and uploaded to the file storage under these
  URLs: https://cdn.com/selfie.jpg, https://cdn.com/front.jpg, https://cdn.com/back.jpg,

After collecting these fields from the user, we can call the "Submit user KYC" endpoint with the following request body:

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
      {
        "image_type_id": 0,
        "image": "https://cdn.com/selfie.jpg"
      },
      {
        "image_type_id": 1,
        "image": "https://cdn.com/front.jpg"
      },
      {
        "image_type_id": 5,
        "image": "https://cdn.com/back.jpg"
      }
    ]
  }
}
```

After the KYC is submitted, we need to wait for it to be approved before proceeding. We can poll the "Get user KYC
state" endpoint until the currentKycStatus becomes "approved". If the KYC is rejected, we need to ask the user to submit
a different document.
If the reachedKycLimit becomes true, the user can't retry KYC anymore and you must contact support.
Once the KYC is approved, we can proceed to get a quote for the order by calling the "Get Quote" endpoint with the
following request body:

```json
{
  "deposit": {
    "paymentChannel": "bank",
    "currencyType": "fiat",
    "currencyCode": "NGN"
  },
  "payout": {
    "paymentChannel": "crypto",
    "currencyType": "crypto",
    "currencyCode": "POLYGON_USDT",
    "amount": 100
  }
}
```

Let's say we received the following response:

```json
{
  "quoteId": "68628fa56ff494df5f39faf5",
  "quoteExpiresAt": "2024-10-10T10:10:10.000Z",
  "deposit": {
    "paymentChannel": "bank",
    "currencyType": "fiat",
    "currencyCode": "NGN",
    "currencyDetails": {
      "countryIsoCode": "NG",
      "countryName": "Nigeria",
      "countryCode": "234",
      "currencySymbol": "₦",
      "countryIcon": "https://cdn.example.com/flags/ng.png"
    },
    "cashout": {
      "exchangeRate": 1500,
      "exchangeRateAfterFees": 1469.5059,
      "amountBeforeFees": 153128,
      "amountAfterFees": 150015,
      "amountBeforeFeesUsd": 102.085333,
      "amountAfterFeesUsd": 100.01,
      "chargedFees": [
        {
          "id": "provider_fee",
          "type": "flat_amount",
          "recipient": "provider",
          "amount": 50
        },
        {
          "id": "platform_fee",
          "type": "percentage",
          "recipient": "platform",
          "amount": 3063
        }
      ],
      "chargedFeesUsd": [
        {
          "id": "provider_fee",
          "type": "flat_amount",
          "recipient": "provider",
          "amount": 0.033333
        },
        {
          "id": "platform_fee",
          "type": "percentage",
          "recipient": "platform",
          "amount": 2.042
        }
      ],
      "totalChargedFees": 3113,
      "totalChargedFeesUsd": 2.075333,
      "chargedFeesPerRecipient": {
        "provider": 50,
        "platform": 3063
      },
      "chargedFeesPerRecipientUsd": {
        "provider": 0.033333,
        "platform": 2.042
      },
      "feeSettings": [
        {
          "id": "provider_fee",
          "recipient": "provider",
          "type": "flat_amount",
          "value": 50,
          "min": 0,
          "max": "Infinity"
        },
        {
          "id": "platform_fee",
          "recipient": "platform",
          "type": "percentage",
          "value": 2,
          "min": 0,
          "max": "Infinity"
        }
      ]
    },
    "fieldsToCreateOrder": [
      {
        "key": "phoneNumber",
        "label": "Phone Number",
        "required": true,
        "type": "phone"
      },
      {
        "key": "bankCode",
        "label": "Bank name",
        "required": true,
        "type": "enum",
        "options": [
          {
            "value": "120001:02",
            "label": "9Payment Service Bank"
          },
          {
            "value": "801:02",
            "label": "Abbey Mortgage Bank"
          }
        ]
      },
      {
        "key": "bankAccountNumber",
        "label": "Bank Account Number",
        "required": true,
        "type": "string"
      }
    ],
    "transferType": "manual"
  },
  "payout": {
    "paymentChannel": "crypto",
    "currencyType": "crypto",
    "currencyCode": "POLYGON_USDT",
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
    "cashout": {
      "exchangeRate": 1,
      "exchangeRateAfterFees": 1.001,
      "amountBeforeFees": 100.01,
      "amountAfterFees": 100,
      "amountBeforeFeesUsd": 100.01,
      "amountAfterFeesUsd": 100,
      "feeSettings": [
        {
          "id": "gas_fee",
          "recipient": "blockchain",
          "type": "flat_amount",
          "value": 0.01,
          "min": 0,
          "max": "Infinity"
        }
      ],
      "chargedFees": [
        {
          "id": "gas_fee",
          "type": "flat_amount",
          "recipient": "blockchain",
          "amount": 0.01
        }
      ],
      "chargedFeesUsd": [
        {
          "id": "gas_fee",
          "type": "flat_amount",
          "recipient": "blockchain",
          "amount": 0.01
        }
      ],
      "totalChargedFees": 0.01,
      "totalChargedFeesUsd": 0.01,
      "chargedFeesPerRecipient": {
        "blockchain": 0.01
      },
      "chargedFeesPerRecipientUsd": {
        "blockchain": 0.01
      }
    },
    "fieldsToCreateOrder": [
      {
        "key": "blockchainWalletAddress",
        "label": "Your wallet address",
        "required": true,
        "type": "string"
      }
    ]
  }
}
```

We see that in order to receive 100 POLYGON_USDT, the user needs to deposit 153128 NGN. Also we see that both deposit
and payout require some fields to be provided when creating the order.
So we need to ask the user to provide the following fields:

- phoneNumber
- bankCode (from the provided enum options)
- bankAccountNumber
- blockchainWalletAddress

After collecting these fields from the user, we can call the "Create Order" endpoint with the following request body:

```json
{
  "quoteId": "68628fa56ff494df5f39faf5",
  "userEmail": "someuser@example.com",
  "userIp": "174.3.2.22",
  "deposit": {
    "paymentChannel": "bank",
    "currencyType": "fiat",
    "currencyCode": "NGN"
  },
  "payout": {
    "paymentChannel": "crypto",
    "currencyType": "crypto",
    "currencyCode": "POLYGON_USDT",
    "amount": 100
  },
  "fieldsToCreateOrder": {
    "phoneNumber": "2348012345678",
    "bankCode": "120001:02",
    "bankAccountNumber": "1234567890",
    "blockchainWalletAddress": "0x5b7ae3c6c83F4A3F94b35c77233b13191eBGAD21"
  }
}
```

Let's say we received the following response:

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
          {
            id: "recipientBankName",
            label: "Bank Name",
            value: "9Payment Service Bank",
          },
          {
            id: "recipientBankAccountNumber",
            label: "Account Number",
            value: "1234567890",
          },
          {
            id: "recipientBankAccountName",
            label: "Account Name",
            value: "Example Company Ltd",
          },
          {
            id: "amountToSend",
            label: "Amount to Send",
            value: "153128",
          },
          {
            id: "bankTransferNarration",
            label: "Transfer Narration / Reference",
            value: "ORDER-5F8D0D55B54764421B7156C5",
            description: "Use this as the transfer reference.",
          }
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

Order is now created and we have received the transfer instructions for the deposit.
Now the user needs to make a bank transfer of 153128 NGN to the provided bank account details with the provided
transfer narration/reference.
Once the user has made the transfer, we need to confirm the deposit by calling the "Confirm Order" endpoint, this order doesn't require any additional fields to be provided for confirmation so we can call it right away.

```json
{
  "orderId": "68728fa56ff494df5f39faf5"
}
```

After calling the "Confirm Order" endpoint, the order status will be updated to "deposit_processing" and the system
will start validating the deposit. Once the deposit is validated, the payout will be processed and the order status will be updated to "payout_successful".
You can call the "Get Order" endpoint to retrieve the order status and details at any time.


## API endpoints

### Get currencies

_**GET** /api/v2/liquidity-bridge/currencies_

Returns supported currencies for deposit and payout with details and available pairs

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
    carriers?: {
      code: string;
      name: string;
    }[]
  }[];
  currencyDetails: OrderCurrencyDetails;
}[]
```

Response example:

```typescript
const response = [
  {
    currencyType: "fiat",
    currencyCode: "NGN",
    paymentChannels: [
      {
        type: "bank",
        transferTypes: ["manual", "redirect"],
        isDepositAllowed: true,
        isPayoutAllowed: true,
      },
      {
        type: "airtime",
        transferTypes: ["ussd"],
        carriers: [
          {code: "MTN", name: "MTN"},
          {code: "AIRTEL", name: "Airtel"},
          {code: "GLO", name: "Glo"},
          {code: "9MOBILE", name: "9Mobile"},
        ],
        isDepositAllowed: true,
        isPayoutAllowed: false,
      },
      {
        type: "mobile_money",
        transferTypes: ["stk_push", "otp_stk_push"],
        carriers: [
          {code: "MTN", name: "MTN Mobile Money"},
          {code: "AIRTEL", name: "Airtel Money"},
          {code: "GLO", name: "Glo Mobile Money"},
          {code: "9MOBILE", name: "9Mobile Money"},
        ],
        isDepositAllowed: true,
        isPayoutAllowed: true,
      },
    ],
    currencyDetails: {
      countryIsoCode: "NG",
      countryName: "Nigeria",
      countryCode: "234",
      currencySymbol: "₦",
      countryIcon: "https://cdn.example.com/flags/ng.png"
    },
    pairs: ["crypto", "merchant_balance"]
  },
  {
    currencyType: "crypto",
    currencyCode: "POLYGON_USDT",
    paymentChannels: [
      {
        type: "crypto",
        transferTypes: ["manual"],
        isDepositAllowed: true,
        isPayoutAllowed: true,
      }
    ],
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
    pairs: ["fiat", "merchant_balance"]
  },
  {
    currencyType: "merchant_balance",
    currencyCode: "USD",
    paymentChannels: [
      {
        type: "merchant_balance",
        transferTypes: [],
        isDepositAllowed: true,
        isPayoutAllowed: true,
      }
    ],
    currencyDetails: {
      merchantName: "Example Company Ltd",
    },
    pairs: ["fiat", "crypto"]
  }
]
```

### Get order limits

_**GET** /api/v2/liquidity-bridge/order-limits_

Returns min and max order limits for a deposit and payout currency pair

Query params:

- depositPaymentChannel: string (required) - The payment channel to deposit from (see PaymentChannel)
- depositCurrencyType: string (required) - The currency to deposit (see CurrencyType)
- depositCurrencyCode: string (required) - The currency to deposit (e.g., "NGN", "KES", "CELO_CUSD", "TRON_USDT")
- depositCarrierCode: string (optional) - The mobile carrier code if depositing via airtime or mobile money
- payoutPaymentChannel: string (required) - The payment channel to payout to (see PaymentChannel)
- payoutCurrencyType: string (required) - The currency to payout (see CurrencyType)
- payoutCurrencyCode: string (required) - The currency to payout (e.g., "NGN", "KES", "CELO_CUSD", "TRON_USDT")
- payoutCarrierCode: string (optional) - The mobile carrier code if paying out via airtime or mobile money

Response type:

```typescript
type OrderLimitsResponse = {
  deposit: {
    min: number,
    max: number,
    minUsd: number,
    maxUsd: number,
  },
  payout: {
    min: number,
    max: number,
    minUsd: number,
    maxUsd: number,
  }
}
```

Response example:

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
  deposit: {
    min: 1556,
    max: 311184,
    minUsd: 1,
    maxUsd: 200,
  },
  payout: {
    min: 1,
    max: 200,
    minUsd: 1,
    maxUsd: 200,
  }
}
```

### Get Quote

_**POST** /api/v2/liquidity-bridge/quote_

Returns pricing quote for a deposit and payout currency pair

Request body:

- deposit.paymentChannel: string (required) - The payment channel to deposit from (see PaymentChannel)
- deposit.currencyType: string (required) - The currency to deposit (see CurrencyType)
- deposit.currencyCode: string (required) - The currency to deposit (e.g., "NGN", "KES", "CELO_CUSD", "TRON_USDT")
- deposit.amount: string (optional) - The amount user pays
- deposit.carrierCode: string (optional) - The mobile carrier code if depositing via airtime or mobile money
- payout.paymentChannel: string (required) - The payment channel to payout to (see PaymentChannel)
- payout.currencyType: string (required) - The currency to payout (see CurrencyType)
- payout.currencyCode: string (required) - The currency to payout (e.g., "
- payout.amount: string (optional) - The amount user receives
- payout.carrierCode: string (optional) - The mobile carrier code if paying out via airtime or mobile money
  (Note: Either deposit.amount or payout.amount must be provided, but not both)

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
    transferType: TransferType
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

Response example:

```typescript
const requestBody = {
  deposit: {
    paymentChannel: "bank",
    currencyType: "fiat",
    currencyCode: "NGN",
    amount: 10000,
  },
  payout: {
    paymentChannel: "crypto",
    currencyType: "crypto",
    currencyCode: "POLYGON_USDT",
  }
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
        {
          id: "provider_fee",
          recipient: "provider",
          type: "flat_amount",
          value: 50,
          min: 0,
          max: "Infinity",
        },
        {
          id: "platform_fee",
          recipient: "platform",
          type: "percentage",
          value: 2,
          min: 0,
          max: "Infinity",
        }
      ],
      chargedFees: [
        {
          id: "provider_fee",
          type: "flat_amount",
          recipient: "provider",
          amount: 50,
        },
        {
          id: "platform_fee",
          type: "percentage",
          recipient: "platform",
          amount: 199,
        }
      ],
      chargedFeesUsd: [
        {
          id: "provider_fee",
          type: "flat_amount",
          recipient: "provider",
          amount: 0.03,
        },
        {
          id: "platform_fee",
          type: "percentage",
          recipient: "platform",
          amount: 0.13,
        }
      ],
      totalChargedFees: 249,
      totalChargedFeesUsd: 0.16,
      chargedFeesPerRecipient: {
        provider: 50,
        platform: 199,
      },
      chargedFeesPerRecipientUsd: {
        provider: 0.03,
        platform: 0.13,
      },
    },
    fieldsToCreateOrder: [
      {
        key: "phoneNumber",
        label: 'Phone Number',
        required: true,
        type: "phone",
      },
      {
        key: "bankCode",
        label: 'Bank name',
        required: true,
        type: "enum",
        options: [
          {
            "value": "120001:02",
            "label": "9Payment Service Bank"
          },
          {
            "value": "801:02",
            "label": "Abbey Mortgage Bank"
          }
        ],
      },
      {
        key: "bankAccountNumber",
        label: 'Bank Account Number',
        required: true,
        type: "string",
      },
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
      feeSettings: [
        {
          id: "gas_fee",
          recipient: "blockchain",
          type: "flat_amount",
          value: 0.01,
          min: 0,
          max: "Infinity",
        }
      ],
      chargedFees: [
        {
          id: "gas_fee",
          type: "flat_amount",
          recipient: "blockchain",
          amount: 0.01,
        }
      ],
      chargedFeesUsd: [
        {
          id: "gas_fee",
          type: "flat_amount",
          recipient: "blockchain",
          amount: 0.01,
        }
      ],
      totalChargedFees: 0.01,
      totalChargedFeesUsd: 0.01,
      chargedFeesPerRecipient: {
        blockchain: 0.01,
      },
      chargedFeesPerRecipientUsd: {
        blockchain: 0.01,
      },
    },
    fieldsToCreateOrder: [
      {
        key: "blockchainWalletAddress",
        label: 'Your wallet address',
        required: true,
        type: "string",
      }
    ],
  }
}
```

### Create Order

_**POST** /api/v2/liquidity-bridge/order_

Creates an order to process a deposit and payout based on a quote if provided

Request body:

```typescript
type CreateOrderRequest = {
  quoteId?: string;
  userEmail: string; // The email of the user creating the order
  userIp: string; // The IP address of the user creating the order
  deposit: {
    paymentChannel: PaymentChannel;
    currencyType: CurrencyType;
    currencyCode: string;
    carrierCode?: string; // The mobile carrier code if depositing via airtime or mobile money
    amount?: number;
  },
  payout: {
    paymentChannel: PaymentChannel;
    currencyType: CurrencyType;
    currencyCode: string;
    carrierCode?: string; // The mobile carrier code if paying out via airtime or mobile money
    amount?: number;
  };
  fieldsToCreateOrder: Record<string, any>; // combined fields for both deposit and payout required to create the order
  redirectUrl?: string; // The URL to redirect to after completing the deposit (for redirect transfer type)
  orderParams?: string; // Optional parameters to associate with the order
  callbackUrl?: string; // URL to the button on the order status page in the widget
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
    statusChangeHistory: {
      oldStatus: OrderStatus;
      newStatus: OrderStatus;
      date: Date;
    }[]
  },
  quoteUsed: boolean,
}
```

Response example:

```typescript
const requestBody = {
  quoteId: "68628fa56ff494df5f39faf5",
  userEmail: "user@example.com",
  userIp: "143.0.2.4",
  deposit: {
    paymentChannel: "bank",
    currencyType: "fiat",
    currencyCode: "NGN",
    amount: 10000,
  },
  payout: {
    paymentChannel: "crypto",
    currencyType: "crypto",
    currencyCode: "POLYGON_USDT",
  },
  fieldsToCreateOrder: {
    blockchainWalletAddress: "0x5b7ae3c6c83F4A3F94b35c77233b13191eBGAD21",
    phoneNumber: "2348012345678",
    bankCode: "120001:02",
    bankAccountNumber: "1234567890",
  }
}

const response = {
  order: {
    //see Get Order response example below
  },
  quoteUsed: true,
}

```

### Get user KYC state

_**GET** /api/v2/liquidity-bridge/user/kyc_

Returns the KYC state of a user, if user don't exist, it will create a new user and return the KYC state.
Also returns the KYC rules and available KYC documents for the user's country.

Query params:

- userEmail: string (required) - The email of the user to get the KYC state for
- countryIsoCode: string (required) - User's country ISO code (e.g., "NG", "KE", "GH")

Response type:

```typescript
const queryParams = {
  userEmail: "user@example.com",
  countryIsoCode: "NG",
};

type GetUserKycResponse = {
  passedKycType?: KycType;
  reachedKycLimit: boolean; // if true, user can't retry KYC anymore
  currentKycStatus?: KycStatus;
  currentKycStatusDescription?: string;
  kycDocuments: KycDocument[],
  kycRules: {
    operationType: OperationType;
    currencyType: CurrencyType;
    min: number; // in USD
    max: number | 'Infinity';
    type: KycType;
  }[]
}
```

Response example:

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
        {
          "key": "first_name",
          "type": "string",
          "label": "First Name",
          "required": true
        },
        {
          "key": "last_name",
          "type": "string",
          "label": "Last Name",
          "required": true
        },
        {
          "key": "dob",
          "type": "date",
          "label": "Date of birth",
          "required": true
        },
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
        {
          "key": "first_name",
          "type": "string",
          "label": "First Name",
          "required": true
        },
        {
          "key": "last_name",
          "type": "string",
          "label": "Last Name",
          "required": true
        },
        {
          "key": "dob",
          "type": "date",
          "label": "Date of birth",
          "required": true
        },
        {
          "key": "images",
          "type": "smile-identity-images",
          "label": "Verification images",
          "required": true
        }
      ]
    },
  ],
  kycRules: [
    {
      operationType: 'deposit',
      currencyType: 'crypto',
      min: 0,
      max: 100,
      type: "basic",
    },
    {
      operationType: 'deposit',
      currencyType: 'crypto',
      min: 100,
      max: 'Infinity',
      type: "basic",
    },
    {
      operationType: 'payout',
      currencyType: 'crypto',
      min: 0,
      max: 'Infinity',
      type: "basic",
    }
  ]
}
```

### Submit user KYC

_**POST** /api/v2/liquidity-bridge/user/kyc_

Submits KYC documents for a user

Request body:

```typescript
type SubmitUserKycRequest = {
  userEmail: string;
  countryIsoCode: string;
  documentId: string;
  userFields: Record<string, string>;
}
```

Request example:

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

Returns the same response as Get user KYC state (see above)

### Trigger intermediate action

_**POST** /api/v2/liquidity-bridge/order/intermediate-action_

Triggers an intermediate action for a deposit order (e.g., STK Push or OTP STK Push).
Needs to be called within the timeout period and before max attempts are reached.

Request body:

```typescript
type TriggerIntermediateActionRequest = {
  orderId: string; // The ID of the order to trigger the action for
  fieldsForIntermediateAction: Record<string, string>; // The fields required to trigger the intermediate action
}
```

Returns the same response as Get Order (see below)

### Confirm order

_**POST** /api/v2/liquidity-bridge/order/confirm_

Confirms a deposit order

Request body:

```typescript
type ConfirmOrderRequest = {
  orderId: string; // The ID of the order to confirm
  fieldsToConfirmOrder?: Record<string, string>; // The fields required to confirm the order (if they are required in the transfer instructions)
}
```

Returns the same response as Get Order (see below)

### Cancel order

_**POST** /api/v2/liquidity-bridge/order/cancel_
Cancels a deposit order if it is still in a cancellable state

Request body:

```typescript
type CancelOrderRequest = {
  orderId: string; // The ID of the order to cancel
}
```

Returns the same response as Get Order (see below)

### Get Order

_**GET** /api/v2/liquidity-bridge/order_

Retrieves an order by its ID or merchant order params

Query params:

- orderId?: string (optional) - The ID of the order to retrieve
- orderParams?: string (optional) - The merchant order params associated with the order

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
  statusChangeHistory: {
    oldStatus: OrderStatus;
    newStatus: OrderStatus;
    date: Date;
  }[]
}

````

Response example:

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
        {
          id: "provider_fee",
          recipient: "provider",
          type: "flat_amount",
          value: 50,
          min: 0,
          max: "Infinity",
        },
        {
          id: "platform_fee",
          recipient: "platform",
          type: "percentage",
          value: 2,
          min: 0,
          max: "Infinity",
        }
      ],
      chargedFees: [
        {
          id: "provider_fee",
          type: "flat_amount",
          recipient: "provider",
          amount: 50,
        },
        {
          id: "platform_fee",
          type: "percentage",
          recipient: "platform",
          amount: 199,
        }
      ],
      chargedFeesUsd: [
        {
          id: "provider_fee",
          type: "flat_amount",
          recipient: "provider",
          amount: 0.03,
        },
        {
          id: "platform_fee",
          type: "percentage",
          recipient: "platform",
          amount: 0.13,
        }
      ],
      totalChargedFees: 249,
      totalChargedFeesUsd: 0.16,
      chargedFeesPerRecipient: {
        provider: 50,
        platform: 199,
      },
      chargedFeesPerRecipientUsd: {
        provider: 0.03,
        platform: 0.13,
      },
    },
    fieldsToCreateOrder: [
      {
        key: "phoneNumber",
        label: 'Phone Number',
        required: true,
        type: "phone",
      },
      {
        key: "bankCode",
        label: 'Bank name',
        required: true,
        type: "enum",
        options: [
          {
            "value": "120001:02",
            "label": "9Payment Service Bank"
          },
          {
            "value": "801:02",
            "label": "Abbey Mortgage Bank"
          }
        ],
      },
      {
        key: "bankAccountNumber",
        label: 'Bank Account Number',
        required: true,
        type: "string",
      },
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
        {
          id: "recipientBankName",
          label: "Bank Name",
          value: "9Payment Service Bank",
        },
        {
          id: "recipientBankAccountNumber",
          label: "Account Number",
          value: "1234567890",
        },
        {
          id: "recipientBankAccountName",
          label: "Account Name",
          value: "Example Company Ltd",
        },
        {
          id: "amountToSend",
          label: "Amount to Send",
          value: "10000",
        },
        {
          id: "bankTransferNarration",
          label: "Transfer Narration / Reference",
          value: "ORDER-5F8D0D55B54764421B7156C5",
          description: "Use this as the transfer reference.",
        }
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
      feeSettings: [
        {
          id: "gas_fee",
          recipient: "blockchain",
          type: "flat_amount",
          value: 0.01,
          min: 0,
          max: "Infinity",
        }
      ],
      chargedFees: [
        {
          id: "gas_fee",
          type: "flat_amount",
          recipient: "blockchain",
          amount: 0.01,
        }
      ],
      chargedFeesUsd: [
        {
          id: "gas_fee",
          type: "flat_amount",
          recipient: "blockchain",
          amount: 0.01,
        }
      ],
      totalChargedFees: 0.01,
      totalChargedFeesUsd: 0.01,
      chargedFeesPerRecipient: {
        blockchain: 0.01,
      },
      chargedFeesPerRecipientUsd: {
        blockchain: 0.01,
      },
    },
    fieldsToCreateOrder: [
      {
        key: "blockchainWalletAddress",
        label: 'Your wallet address',
        required: true,
        type: "string",
      }
    ],
    providedFieldsToCreateOrder: {
      blockchainWalletAddress: "0x5b7ae3c6c83F4A3F94b35c77233b13191eBGAD21",
    }
  },
  createdAt: new Date("2025-10-01T12:00:00Z"),
  updatedAt: new Date("2025-10-01T12:00:00Z"),
  statusChangeHistory: [
    {
      oldStatus: "deposit_awaiting",
      newStatus: "deposit_validating",
      date: new Date("2025-10-01T12:05:00Z"),
    },
    {
      oldStatus: "deposit_validating",
      newStatus: "deposit_successful",
      date: new Date("2025-10-01T12:10:00Z"),
    },
    {
      oldStatus: "deposit_successful",
      newStatus: "payout_pending",
      date: new Date("2025-10-01T12:15:00Z"),
    }
  ]
}
```

### Get merchant balance

_**GET** /api/v2/liquidity-bridge/merchant-balance_

Returns the current merchant balance (currently only in USD)

Response type:

```typescript
type MerchantBalanceResponse = {
  USD: number;
}
```

## Types used in the above definitions:

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
  carriers?: {
    code: string;
    name: string;
  }[];
}

type OrderMerchantDetails = {
  merchantName: string;
}

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
  | {
  id: string;
  recipient: FeeRecipient;
  type: FeeType.FLAT_AMOUNT;
  value: number;
  min: number; // minimum order amount for the fee to apply
  max: number | 'Infinity'; // maximum order amount for the fee to apply
}
  | {
  id: string;
  recipient: FeeRecipient;
  type: FeeType.PERCENTAGE;
  value: number;
  min: number; // minimum order amount for the fee to apply
  max: number | 'Infinity'; // maximum order amount for the fee to apply
  minCap?: number; // minimum fee amount
  maxCap?: number; // maximum fee amount
};

type ChargedFee = {
  id: string;
  type: FeeType;
  recipient: FeeRecipient;
  amount: number;
};

enum FeeRecipient {
  MERCHANT = 'merchant',
  PROVIDER = 'provider',
  PLATFORM = 'platform',
  BLOCKCHAIN = 'blockchain',
}

enum FeeType {
  PERCENTAGE = 'percentage',
  FLAT_AMOUNT = 'flat_amount',
}

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

enum TransferType {
  MANUAL = 'manual',
  REDIRECT = 'redirect',
  STK_PUSH = 'stk_push',
  OTP_STK_PUSH = 'otp_stk_push',
  USSD = 'ussd',
}

type TransferDetail = {
  id: TransferDetailId;
  label: string;
  description?: string;
  value?: string;
};

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
  options?: {
    value: string;
    label: string;
  }[];
  defaultValue?: string;
};

enum FieldType {
  NUMBER = 'number',
  STRING = 'string',
  DATE = 'date',
  BOOLEAN = 'boolean',
  EMAIL = 'email',
  PHONE = 'phone',
  ENUM = 'enum',
}

enum OrderStatus {
  DEPOSIT_AWAITING = 'deposit_awaiting', // Waiting for user to make the deposit
  DEPOSIT_VALIDATING = 'deposit_validating', // Deposit made, waiting for confirmation
  DEPOSIT_INVALID = 'deposit_invalid', // Deposit was invalid (e.g., wrong amount, wrong account)
  DEPOSIT_SUCCESSFUL = 'deposit_successful',
  DEPOSIT_CANCELED = 'deposit_canceled', // User canceled the order before making the deposit
  PAYOUT_PENDING = 'payout_pending', // Deposit confirmed, payout is being processed
  PAYOUT_SUCCESSFUL = 'payout_successful', // Payout completed successfully
  PAYOUT_FAILED = 'payout_failed',
  REFUNDING_INITIATED = 'refunding_initiated',
  REFUNDING_PENDING = 'refunding_pending',
  REFUNDING_SUCCESSFUL = 'refunding_successful',
  REFUNDING_FAILED = 'refunding_failed',
  EXPIRED = 'expired',
}

enum KycType {
  BASIC = 'basic',
  ADVANCED = 'advanced',
}

export enum KycStatus {
  INITIATED = 'initiated',
  APPROVED = 'approved',
  REJECTED = 'rejected',
  INVALID = 'invalid',
}

type KycDocument = {
  "_id": string,
  "title": string,
  "value": string,
  "type": KycType,
  "requiredFields": DynamicField[],
}

export type DynamicField =
  | {
  key: string;
  type:
    | 'number'
    | 'string'
    | 'date'
    | 'boolean'
    | 'email'
    | 'phone'
    | 'smile-identity-images';
  label: string;
  required: boolean;
  defaultValue?: string | number | boolean;
  regexp?: string;
  regexpFlags?: string;
  format?: string;
}
  | {
  key: string;
  type: 'enum';
  label: string;
  required: boolean;
  options: {
    value: string;
    label: string;
  }[];
  defaultValue?: string;
  regexp?: string;
  regexpFlags?: string;
  format?: string;
};

enum OperationType {
  DEPOSIT = 'deposit',
  PAYOUT = 'payout',
}

```


