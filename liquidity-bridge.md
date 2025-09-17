# Liquidity Bridge API Documentation (WIP)


## API endpoints

### [Get currencies](#get-currencies)

_**GET** /api/v2/liquidity-bridge/currencies_

Returns supported currencies for deposit and payout with details and available pairs

Response type:
```typescript
type CurrenciesResponse = {
  currencyType: CurrencyType;
  currencyCode: string;
  paymentChannels: PaymentChannel[];
  currencyDetails: OrderCurrencyDetails;
  pairs: CurrencyType[]
}[]
```

Response example:

```typescript
const response = [
  {
    currencyType: "fiat",
    currencyCode: "NGN",
    paymentChannels: ["bank", "mobile_money", "airtime"],
    currencyDetails: {
      countryIsoCode: "NG",
      countryName: "Nigeria",
      countryCode: "234",
      currencySymbol: "₦",
      countryIcon: "https://cdn.example.com/flags/ng.png",
      carriers: []
    },
    pairs: ["crypto", "merchant_balance"]
  },
  {
    currencyType: "crypto",
    currencyCode: "POLYGON_USDT",
    paymentChannels: ["crypto"],
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
  }
]
```

### [Get order limits](#get-order-limits)

_**GET** /api/v2/liquidity-bridge/order-limits_

Returns min and max order limits for a deposit and payout currency pair

Query params:
- depositPaymentChannel: string (required) - The payment channel to deposit from (see PaymentChannel)
- depositCurrencyType: string (required) - The currency to deposit (see CurrencyType)
- depositCurrencyCode: string (required) - The currency to deposit (e.g., "NGN", "KES", "CELO_CUSD", "TRON_USDT")
- payoutCurrencyType: string (required) - The currency to payout (see CurrencyType)
- payoutCurrencyCode: string (required) - The currency to payout (e.g., "NGN", "KES", "CELO_CUSD", "TRON_USDT")

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

### (Get Quote)[#get-quote]
_**POST** /api/v2/liquidity-bridge/quote_

Returns pricing quote for a deposit and payout currency pair

Request body:
- deposit.paymentChannel: string (required) - The payment channel to deposit from (see PaymentChannel)
- deposit.currencyType: string (required) - The currency to deposit (see CurrencyType)
- deposit.currencyCode: string (required) - The currency to deposit (e.g., "NGN", "KES", "CELO_CUSD", "TRON_USDT")
- deposit.amount: string (optional) - The amount user pays
- payout.currencyType: string (required) - The currency to payout (see CurrencyType)
- payout.currencyCode: string (required) - The currency to payout (e.g., "
- payout.amount: string (optional) - The amount user receives
  (Note: Either deposit.amount or payout.amount must be provided, but not both)


Response type:
```typescript
type QuoteResponse = {
  quoteId: string;
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
      carriers: []
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
          id: "provider-fee",
          recipient: "provider",
          type: "flat_amount",
          value: 50,
          min: 0,
          max: "Infinity",
        },
        {
          id: "platform-fee",
          recipient: "platform",
          type: "percentage",
          value: 2,
          min: 0,
          max: "Infinity",
        }
      ],
      chargedFees: [
        {
          id: "provider-fee",
          type: "flat_amount",
          recipient: "provider",
          amount: 50,
        },
        {
          id: "platform-fee",
          type: "percentage",
          recipient: "platform",
          amount: 199,
        }
      ],
      chargedFeesUsd: [
        {
          id: "provider-fee",
          type: "flat_amount",
          recipient: "provider",
          amount: 0.03,
        },
        {
          id: "platform-fee",
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
      exchangeRateAfterFees: 0.9985,
      amountBeforeFees: 6.5,
      amountAfterFees: 6.49,
      amountBeforeFeesUsd: 6.5,
      amountAfterFeesUsd: 6.49,
      feeSettings: [
        {
          id: "gas-fee",
          recipient: "blockchain",
          type: "flat_amount",
          value: 0.01,
          min: 0,
          max: "Infinity",
        }
      ],
      chargedFees: [
        {
          id: "gas-fee",
          type: "flat_amount",
          recipient: "blockchain",
          amount: 0.01,
        }
      ],
      chargedFeesUsd: [
        {
          id: "gas-fee",
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


### (Create Order)[#create-order]

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
    amount?: number;
    fields: Record<string, any>;
  },
  payout: {
    paymentChannel: PaymentChannel;
    currencyType: CurrencyType;
    currencyCode: string;
    amount?: number;
    fields: Record<string, any>;
  };
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
    fields: {
      phoneNumber: "2348012345678",
      bankCode: "120001:02",
      bankAccountNumber: "1234567890",
    },
  },
  payout: {
    paymentChannel: "crypto",
    currencyType: "crypto",
    currencyCode: "POLYGON_USDT",
    fields: {
      blockchainWalletAddress: "0x5b7ae3c6c83F4A3F94b35c77233b13191eBGAD21",
    }
  }
}

const response = {
  order: {
    _id: "5f8d0d55b54764421b7156c5",
    countryIsoCode: "NG",
    userId: "68628fa56ff494df5f39faf6",
    userEmail: "user@example.com",
    merchantOrderParams: "order-12345",
    status: "deposit_awaiting",
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
        carriers: []
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
            id: "provider-fee",
            recipient: "provider",
            type: "flat_amount",
            value: 50,
            min: 0,
            max: "Infinity",
          },
          {
            id: "platform-fee",
            recipient: "platform",
            type: "percentage",
            value: 2,
            min: 0,
            max: "Infinity",
          }
        ],
        chargedFees: [
          {
            id: "provider-fee",
            type: "flat_amount",
            recipient: "provider",
            amount: 50,
          },
          {
            id: "platform-fee",
            type: "percentage",
            recipient: "platform",
            amount: 199,
          }
        ],
        chargedFeesUsd: [
          {
            id: "provider-fee",
            type: "flat_amount",
            recipient: "provider",
            amount: 0.03,
          },
          {
            id: "platform-fee",
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
        exchangeRateAfterFees: 0.9985,
        amountBeforeFees: 6.5,
        amountAfterFees: 6.49,
        amountBeforeFeesUsd: 6.5,
        amountAfterFeesUsd: 6.49,
        feeSettings: [
          {
            id: "gas-fee",
            recipient: "blockchain",
            type: "flat_amount",
            value: 0.01,
            min: 0,
            max: "Infinity",
          }
        ],
        chargedFees: [
          {
            id: "gas-fee",
            type: "flat_amount",
            recipient: "blockchain",
            amount: 0.01,
          }
        ],
        chargedFeesUsd: [
          {
            id: "gas-fee",
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
    statusChangeHistory: []
  },
  quoteUsed: true,
}

```

### [Trigger intermediate action](#trigger-intermediate-action)

_**POST** /api/v2/liquidity-bridge/order/intermediate-action

Triggers an intermediate action for a deposit order (e.g., STK Push or OTP STK Push).
Needs to be called within the timeout period and before max attempts are reached.

Request body:
```typescript
type TriggerIntermediateActionRequest = {
  orderId: string; // The ID of the order to trigger the action for
  fields: Record<string, string>; // The fields required to trigger the intermediate action
}
```

Returns the same response as Get Order (see below)


### Get Order

_**GET** /api/v2/liquidity-bridge/order

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
The response example is the same as the Create Order response example above.

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

export enum CurrencyType {
  FIAT = 'fiat',
  CRYPTO = 'crypto',
  MERCHANT_BALANCE = 'merchant_balance',
}
export type OrderCurrencyDetails =
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

export type Cashout = {
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
  min: number;
  max: number | 'Infinity';
}
  | {
  id: string;
  recipient: FeeRecipient;
  type: FeeType.PERCENTAGE;
  value: number;
  min: number;
  max: number | 'Infinity';
  minCap?: number;
  maxCap?: number;
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

export type TransferInstructions =
  | UssdTransferInstructions
  | ManualTransferInstructions
  | RedirectTransferInstructions
  | StkPushTransferInstructions
  | OtpStkPushTransferInstructions;

export type UssdTransferInstructions = {
  type: TransferType.USSD;
  ussdCode: string;
  instructionsText: string;
  warningText?: string;
  transferDetails: TransferDetail[];
  fieldsToConfirmOrder?: RequiredField[];
};
export type ManualTransferInstructions = {
  type: TransferType.MANUAL;
  instructionsText: string;
  warningText?: string;
  transferDetails: TransferDetail[];
  fieldsToConfirmOrder: RequiredField[];
};
export type RedirectTransferInstructions = {
  type: TransferType.REDIRECT;
  paymentUrl: string;
  redirectedToPaymentUrl: boolean;
  intermediateActionButtonText: string;
  instructionsText: string;
  warningText?: string;
  transferDetails: TransferDetail[];
  fieldsToConfirmOrder: RequiredField[];
};

export type StkPushTransferInstructions = {
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

export type OtpStkPushTransferInstructions = {
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

export enum TransferType {
  MANUAL = 'manual',
  REDIRECT = 'redirect',
  STK_PUSH = 'stk_push',
  OTP_STK_PUSH = 'otp_stk_push',
  USSD = 'ussd',
}

export type TransferDetail = {
  id: TransferDetailId;
  label: string;
  description?: string;
  value?: string;
};

export enum TransferDetailId {
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

export type RequiredField = {
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

export enum FieldType {
  NUMBER = 'number',
  STRING = 'string',
  DATE = 'date',
  BOOLEAN = 'boolean',
  EMAIL = 'email',
  PHONE = 'phone',
  ENUM = 'enum',
}

export enum OrderStatus {
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

```


