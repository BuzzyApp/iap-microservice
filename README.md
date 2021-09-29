# IAP Microservice

A microservice for processing and managing Android and iOS In App Payments and Subscriptions.

The microservice is designed to be the source-of-truth for your payments and subscriptions for your app.
It will keep track of customers, payments, and most importantly subscriptions and their statuses/states.

Features:
 - Receipt Validation - Validate iOS/Android purchase receipts/tokens
 - Webhooks - Notify your backend when Apple/Google processes subscription payments
 - Subscription tracking - Automatically tracks the state of subscriptions over their entire lifecycle
 - Currency Conversion - (Android Only) Automatically converts purchase/refund amounts to the user's currency
 - Refund tracking - Automatically tracks and sends webhooks for refund events

### Feedback wanted!
If you find that this doc is missing something important, or you are stuck getting things working,
please submit a new GitHub issue about it so that we can try and solve your problem and make it better for everyone else.

## Important
The microservice's authentication system is basic and has been designed with the assumption
that you will using server-to-server communication with the microservice.
DO NOT embed your IAP Microservice API key in your app, as it will no longer be secret and anyone that finds it will
have complete access to your IAP Microservice. You have been warned!

## Requirements
 - MySQL Database
 - Docker

## Setup
 - Setup incoming webhooks
 - Setup outgoing webhooks
 - Setup API Authentication
 - Create & migrate database
 - Create products
 - Process receipts

## Incoming Webhooks
Incoming webhooks process messages from Apple and Google.

TODO: Add instructions

## Outgoing Webhooks
Outgoing webhooks are messages sent from IAP Microservice to your backend after an event from Apple or Google happens,
or subscription expires/goes into retry period/etc

TODO: Add instructions

## Database
TODO: Add instructions

## API
Unfortunately there is no UI at this point, so all management will need to be done via the API.

### Authentication
Set `API_KEY` to a random string in `.env`, and then add the `Authorization` header to your API requests using that key.

E.g:
```
Authorization=a8a7s8hasd98nasg7384u9jASND
```

### Creating Products
The microservice needs to be aware of the products you are selling.
When a receipt is processed by the microservice, it will record the purchase price of the associated product.
For purchases on Android the microservice will also convert the currency at the time of purchase.
Currency conversion is not available for iOS purchases as iOS receipts do not contain currency information.

To create/update a Product, POST JSON to `/product` with the following:
```
id: (optional) Product ID to update. Ommit to create a new product
sku_android: Android Product ID
sku_ios:     Apple Product ID
price:       Base price in *cents*
currency:    The currency code the base price is in
```

Returns: The Product that was created/updated
```
Product = {
  id? string;
  skuAndroid?: string;
  skuIOS?: string;
  price: number;
  currency: string;
};
```

### Basic Receipt Validation
If you don't need all the bells and whistles, basic receipt validation can be done without touching the database.

To validate a receipt, POST JSON to `/validate` with the following:
```
platform:  'android' or 'ios'
token:     The raw iOS Receipt base64 or Android receipt token
sku:       (required on Android only) The Android Product ID that the receipt is for 
```
Returns: The raw validation response from Apple/Google

### Receipt Processing
A user has just purchased something in your app - that's great! Let the microservice know about it.

To process a new receipt, POST JSON to `/receipt` with the following:
```
platform:     'android' or 'ios'
token:        The raw iOS Receipt base64 or Android receipt token
sku:          (required on Android only) The Android Product ID that the receipt is for 
user_id:      The user's ID in your system. Use this to keep track of purchases from your users in your backend.
sync_user:    (Optional) Boolean - If tue, all related purchases will be changed to this user id. Useful if user_id from your backend changes over time for some reason. 
skip_webhook: (Optional) Boolean - If true, don't send a webhook in response to processing this receipt. Useful to prevent double processing a new purchase that your backend is already aware of. 
sync_user:    Boolean - If tue, all related purchases will be changed to this user id. Useful if user_id from your backend changes over time for some reason. 
import:       (Optional) Boolean - Should future purchases related to this token be processed as well?
              This flag is useful when importing existing tokens to import the full transaction history
```

Returns: A Purchase object representing the parsed purchase (please see `src/types.d.ts` for complete typings)
```
Purchase = {
  id?: string;
  receiptId: string;
  productId?: string;
  linkedPurchaseId?: string;
  originalPurchaseId?: string;

  userId?: string; // Arbitrary ID provided by user
  // Generic Order Properties
  isSandbox: boolean;
  isRefunded: boolean;
  quantity: number;
  platform: Platform;
  orderId?: string;
  productSku: string;
  productType: "consumable" | "non_consumable" | "renewable_subscription";
  purchaseDate: Date;
  price: number;
  currency: string;
  convertedPrice: number;
  convertedCurrency: string;
  receiptDate: Date;
  refundDate: Date | null;
  refundReason: "issue" | "subscription_replace" | "other" | null;

  // Subscription Flags
  isSubscription: boolean;
  isTrial?: boolean;
  isIntroOfferPeriod?: boolean;
  isSubscriptionActive?: boolean;
  isSubscriptionRenewable?: boolean;
  isSubscriptionRetryPeriod?: boolean;
  isSubscriptionGracePeriod?: boolean;
  isSubscriptionPaused?: boolean;
  isTrialConversion?: boolean;

  // Additional Subscription Info
  originalOrderId?: string;
  linkedOrderId?: string;
  subscriptionPeriodType?: SubscriptionPeriodType;
  subscriptionState?: SubscriptionState;
  subscriptionStatus?: SubscriptionStatus;
  subscriptionGroup?: string;
  subscriptionRenewalProductSku?: string;
  cancellationReason?: CancellationReason;
  expirationDate?: Date;
  gracePeriodEndDate?: Date;
  linkedToken?: string; // Android only
};
```

### Get a Single Purchase
To get the latest status of a Purchase, GET `/purchase/{id}` to retrieve the purchase status

### Get all Purchases for a User
To get a list/array of all Purchases related to given user_ud, GET `/purchase/{user_id}/purchases` to retrieve the purchases

### Get Subscription Status/Get all purchases for a given Subscription
To retrieve the up-to-date payment history of a subscription, GET `/user/{user_id}/purchases/{original_purchase_id}`

Where `original_purchase_id` is the Purchase ID of the original Subscription Purchase (this is embedded in all purchase relating to a given subscription)

To get the current status of a subscription, simply inspect the subscription fields in the first item of the returned array


## Cron
For the microservice to keep track of subscriptions it needs to periodically refresh receipts with Apple and Google.

To trigger a receipt refresh, setup `cron` to GET `/cron`. Suggested interval is every 30 minutes.

TODO: Add instructions

## License
MIT - see [LICENSE.md](https://github.com/BuzzyPhone/iap-microservice/blob/master/LICENSE.md)
