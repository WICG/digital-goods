# Digital Goods API - Explainer

Authors: Matt Giuca \<<mgiuca@chromium.org>\>,
         Glen Robertson \<<glenrob@chromium.org>\>

* [The problem](#the-problem)
* [The proposed API](#the-proposed-api)
  + [Getting a service instance](#getting-a-service-instance)
  + [Querying item details](#querying-item-details)
  + [Making a purchase](#making-a-purchase)
  + [Acknowledging a purchase](#acknowledging-a-purchase)
  + [Consuming a purchase](#consuming-a-purchase)
  + [Checking existing purchases](#checking-existing-purchases)
* [Full API interface](#full-api-interface)
  + [API v2.1](#api-v21)
  + [API v2.0](#api-v20)
  + [API v1.0 (deprecated)](#api-v10-deprecated)
* [Formatting the price](#formatting-the-price)
* [Security and Privacy Considerations](#security-and-privacy-considerations)

This document proposes the Digital Goods API for querying and managing digital products to facilitate in-app purchases from web applications. It is **complementary to the [Payment Request API](https://www.w3.org/TR/payment-request/)**, which is used to make purchases of the digital products. This API would be linked to a digital store connected to via the user agent.


## The problem

The problem this API solves is that *Payment Request by itself is inadequate for making in-app purchases in existing digital stores*, because that API simply asks the user to make a payment of a certain amount (e.g., “Please authorize a transaction of US$3.00”), which is sufficient for websites selling their own products, but established digital distribution services require apps to make purchases by item IDs, not monetary amounts (e.g., “Please authorize the purchase of SHINY\_SWORD”); the price being configured per-region on the backend.

The Payment Request API can be used, with [a minor modification](https://github.com/w3c/payment-request/issues/912), to make in-app purchases using the digital distribution service as a payment method, by supplying the desired item IDs as `data` in the `modifiers` member for that particular payment method. However, there are ancillary operations relating to in-app purchases that are not part of that API:

*   Querying the details (e.g., name, description, regional price) of digital items from the store backend.
    *   Note: Even though the web app developer is ultimately responsible for configuring these items on the server, and could therefore keep track of these without an API, it is important to have a single source of truth. This ensures that the price of items displayed in the app exactly matches the prices that the user will eventually be charged, especially as prices can differ by region, or change at planned times (such as when sale events begin or end).
*   Consuming or acknowledging purchases. Digital stores typically do not consider a purchase finalized until the client acknowledges the purchase through a separate API call. This acknowledgment is supposed to be performed once the client “activates” the purchase inside the app.
*   Checking the digital items currently owned by the user.

It is typically a requirement for listing an application in a digital store that purchases are made through that store’s billing API. Therefore, access to these operations is a requirement for web sites to be listed in various digital stores, if they wish to sell digital products.

### Example Use Cases

*   Listing the available subscription options for your site's service, in the user's currency, as configured with a store backend.
*   Check that a user has a purchased resource in your web game, and use it up when appropriate, using the store backend's infrastructure.
*   Checking with a store backend that a user is eligible to access premium content on your site, having purchased it or used a promotional code in the past.

## The proposed API

The Digital Goods API allows the user agent to provide the above operations, alongside digital store integration via the Payment Request API.

Sites using the proposed API would still need to be configured to work with each individual store they are listed in, but having a standard API means they can potentially have that integration work across multiple browsers. This is similar to how the existing Payment Request API works (sites still need to integrate with each payment provider, e.g., Google Pay, Apple Pay, but their implementation is browser agnostic).

### Getting a service instance

Usage of the API would begin with a call to `window.getDigitalGoodsService()`, which might only be available in certain contexts (eg. HTTPS, browser, OS). If available, the method can be called with a service provider URL.The method returns a promise that is rejected if the given service provider is not available:

```js
if (window.getDigitalGoodsService === undefined) {
  // Digital Goods API is not supported in this context.
  return;
}
try {
  const digitalGoodsService = await window.getDigitalGoodsService("https://example.com/billing");
  // Use the service here.
  ...
} catch (error) {
  // Our preferred service provider is not available.
  // Use a normal web-based payment flow.
  return;
}
```

#### Note

For backwards compatibility with Digital Goods API v1.0 while both are available, developers should also check whether the returned `digitalGoodsService` object is `null`:

```js
if (digitalGoodsService === null) {
  // Our preferred service provider is not available.
  // Use a normal web-based payment flow.
  return;
}
```

### Querying item details

The `getDetails` method returns server-side details about a given set of items, intended to be displayed to the user in a menu, so that they can see the available purchase options and prices without having to go through a purchase flow.


```js
details = await digitalGoodsService.getDetails(['shiny_sword', 'gem', 'monthly_subscription']);
for (item of details) {
  const priceStr = new Intl.NumberFormat(
      locale,
      {style: 'currency', currency: item.price.currency}
    ).format(item.price.value);
  AddShopMenuItem(item.itemId, item.title, priceStr, item.description);
}
```

The returned `itemDetails` sequence may be in any order and may not include an item if it doesn't exist on the server (i.e. there is not a 1:1 correspondence between the input list and output).
 
The item ID is a string representing the primary key of the items, configured in the store server. There is no function to get a list of item IDs; those should be hard-coded in the client code or fetched from the developer’s own server.

The item’s `price` is a <code>[PaymentCurrencyAmount](https://developer.mozilla.org/en-US/docs/Web/API/PaymentCurrencyAmount)</code> containing the current price of the item in the user’s current region and currency. It is designed to be formatted for the user’s current locale using <code>[Intl.NumberFormat](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/NumberFormat)</code>, as shown above.

The item can optionally have various periods, specified using [ISO 8601 durations](https://en.wikipedia.org/wiki/ISO_8601#Durations). The introductory price period can run for multiple such periods, as specified by `introductoryPriceCycles`. For further discussion of periods and introductory price cycles, see [Issue#20](https://github.com/WICG/digital-goods/issues/20).

### Making a purchase

The purchase flow itself uses the [Payment Request API](https://w3c.github.io/payment-request/). We don’t show the full payment request code here, but note that the item ID for any items the user chooses to purchase should be sent in the `data` field of a `modifiers` entry for the given payment method, in a manner specific to the store. For example:

```js
const details = await digitalGoodsService.getDetails(['monthly_subscription']);
const item = details[0];
new PaymentRequest(
  [{supportedMethods: 'https://example.com/billing',
    data: {itemId: item.itemId}}]);
```

Note that as part of this proposal, we are proposing to [remove the requirement](https://github.com/w3c/payment-request/issues/912) of the `total` member of the `details` dictionary, since the source of truth for the item price (that will be displayed to the user in the purchase confirmation dialog) is known by the server, based on the item ID. The exact format of the `data` member is up to the store (the spec simply says this is an `object`). Some stores may allow multiple items to be purchased at the same time, others only a single item.

### Acknowledging a purchase

The payment response will return a "purchase token" string, which can be used for direct communication between the developer's server and the service provider beyond the Digital Goods API. Such communication can allow the developer to independently verify information about the purchase before granting entitlements. Some stores might require that the developer acknowledge a purchase once it has succeeded, to confirm that it has been recorded.

### Consuming a purchase

Purchases that are designed to be purchased multiple times usually need to be marked as "consumed" before they can be purchased again by the user. An example of a consumable purchase is an in-game powerup that makes the player stronger for a short period of time. This can be done with the `consume` method:

```js
digitalGoodsService.consume(purchaseToken);
```

It is preferable to use a direct developer-to-provider API to consume purchases, if one is available, in order to more verifiably ensure that a purchase was used up.

### Checking existing purchases

The `listPurchases` method allows a client to get a list of items that are currently owned or purchased by the user. This may be necessary to check for entitlements (e.g. whether a subscription, promotional code, or permanent upgrade is active) or to recover from network interruptions during a purchase (e.g. item is purchased but not yet acknowledged). The method returns item IDs and purchase tokens, which should be verified using a direct developer-to-provider API before granting entitlements.

```js
purchases = await digitalGoodsService.listPurchases();
for (p of purchases) {
  VerifyAndGrantEntitlement(p.itemId, p.purchaseToken);
}
```

The `listPurchaseHistory` method allows a client to get a list of previous purchases by the user, regardless of current ownership state. Depending on the underlying service provider support, this might be limited to a single purchase record per item.

## Full API interface

### API v2.1
Expected to be added in Chrome M102+. This is a non-breaking change adding additional methods and optional fields. Use of the new methods/fields will require developers to update supporting code in their apps, such as [Android Browser Helper](https://github.com/GoogleChrome/android-browser-helper).

*   Adds to DigitalGoodsService
    *   `Promise<sequence<PurchaseDetails>> listPurchaseHistory();`
*   Adds to ItemDetails
    *   `ItemType type;`
    *   `sequence<DOMString> iconURLs;`
    *   `[EnforceRange] unsigned long long introductoryPriceCycles;`
*   Adds `enum ItemType`

```webidl
[SecureContext]
partial interface Window {
  // Rejects the promise if there is no Digital Goods Service associated with
  // the given service provider.
  Promise<DigitalGoodsService> getDigitalGoodsService(DOMString serviceProvider);
};

[SecureContext]
interface DigitalGoodsService {
  Promise<sequence<ItemDetails>> getDetails(sequence<DOMString> itemIds);
  
  Promise<sequence<PurchaseDetails>> listPurchases();
  
  Promise<sequence<PurchaseDetails>> listPurchaseHistory();

  Promise<void> consume(DOMString purchaseToken);
};

dictionary ItemDetails {
  required DOMString itemId;
  required DOMString title;
  required PaymentCurrencyAmount price;
  ItemType type;
  DOMString description;
  sequence<DOMString> iconURLs;
  // Periods are specified as ISO 8601 durations.
  // https://en.wikipedia.org/wiki/ISO_8601#Durations
  DOMString subscriptionPeriod;
  DOMString freeTrialPeriod;
  PaymentCurrencyAmount introductoryPrice;
  DOMString introductoryPricePeriod;
  [EnforceRange] unsigned long long introductoryPriceCycles;
};

enum ItemType {
  "product",
  "subscription",
};

dictionary PurchaseDetails {
  required DOMString itemId;
  required DOMString purchaseToken;
};
```

### API v2.0
In Origin Trial in Chrome M96-M99.

```webidl
[SecureContext]
partial interface Window {
  // Rejects the promise if there is no Digital Goods Service associated with
  // the given service provider.
  Promise<DigitalGoodsService> getDigitalGoodsService(DOMString serviceProvider);
};

[SecureContext]
interface DigitalGoodsService {
  Promise<sequence<ItemDetails>> getDetails(sequence<DOMString> itemIds);
  
  Promise<sequence<PurchaseDetails>> listPurchases();

  Promise<void> consume(DOMString purchaseToken);
};

dictionary ItemDetails {
  required DOMString itemId;
  required DOMString title;
  required PaymentCurrencyAmount price;
  DOMString description;
  // Periods are specified as ISO 8601 durations.
  // https://en.wikipedia.org/wiki/ISO_8601#Durations
  DOMString subscriptionPeriod;
  DOMString freeTrialPeriod;
  PaymentCurrencyAmount introductoryPrice;
  DOMString introductoryPricePeriod;
};

dictionary PurchaseDetails {
  required DOMString itemId;
  required DOMString purchaseToken;
};
```

### API v1.0 (deprecated)
Origin trial ran in Chrome from [M89 to M95 (inclusive)](https://chromestatus.com/feature/5339955595313152).


```webidl
[SecureContext]
partial interface Window {
  // Resolves the promise with null if there is no service associated with the
  // given payment method.
  Promise<DigitalGoodsService?> getDigitalGoodsService(DOMString paymentMethod);
};

[SecureContext]
interface DigitalGoodsService {
  Promise<sequence<ItemDetails>> getDetails(sequence<DOMString> itemIds);

  Promise<void> acknowledge(DOMString purchaseToken,
                            PurchaseType purchaseType);
  
  Promise<sequence<PurchaseDetails>> listPurchases();
};

enum PurchaseType {
  "repeatable",
  "onetime",
};

dictionary ItemDetails {
  required DOMString itemId;
  required DOMString title;
  required PaymentCurrencyAmount price;
  DOMString description;
  // Periods are specified as ISO 8601 durations.
  // https://en.wikipedia.org/wiki/ISO_8601#Durations
  DOMString subscriptionPeriod;
  DOMString freeTrialPeriod;
  PaymentCurrencyAmount introductoryPrice;
  DOMString introductoryPricePeriod;
};

dictionary PurchaseDetails {
  required DOMString itemId;
  required DOMString purchaseToken;
  boolean acknowledged = false;
  PurchaseState purchaseState;
  // Timestamp in ms since 1970-01-01 00:00 UTC.
  DOMTimeStamp purchaseTime;
  boolean willAutoRenew = false;
};

enum PurchaseState {
  "purchased",
  "pending",
};
```

## Formatting the price

The ItemDetails struct contains a `price` member which gives the price of the item in the user’s currency (this is a [PaymentCurrencyAmount](https://developer.mozilla.org/en-US/docs/Web/API/PaymentCurrencyAmount), the same format used by the [Payment Request](https://w3c.github.io/payment-request/#dom-paymentcurrencyamount) and [Payment Handler](https://www.w3.org/TR/payment-handler/#dom-paymentrequestevent-total) APIs). This is provided purely informationally, so that the price can be displayed to the user before they choose to purchase the item (the price will be re-confirmed to the user in the user-agent-controlled payment dialog). The website does _not_ need to do anything with this price (e.g., pass it into the Payment Request flow), but _should_ show it to the user.

To format the price, use the [Intl.NumberFormat API](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/NumberFormat), for example:


```js
new Intl.NumberFormat(
  locale,
  {style: 'currency',
   currency: price.currency}).format(price.value);
```

This will correctly format the price in the given locale (which should be set to the user’s locale), in the currency that the user will use to make the purchase.

## Security and Privacy Considerations

This API should be used in a secure context. Additionally a user agent could restrict use of the API using [feature policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Feature_Policy) and/or restrict it to top-level contexts only. The digital products managed by the API are expected to be specific to one origin, so information retrieved through the API would relate to the current origin only.

This API assumes that the user agent has some existing authentication process for the user, e.g. some extra UI when the API is initialised, or some implicit platform or browsing context. Because an authenticated user is likely needed for the API to be meaningful, and information is only exposed for purchases that user has already made from this origin, there is minimal additional potentially-identifying information to be gained through this API.

## Analysis of various APIs

### [Play Store BillingClient API](https://developer.android.com/reference/com/android/billingclient/api/BillingClient)
*    Uses the term “SKU” for items.
*    No server-side distinction between “consumable” and “one-time purchase” items. Choice is dynamic: call “consume” to consume an item and make it available for purchase again, call “acknowledge” to acknowledge purchase of a one-time item and not make it available again.

### [Samsung in-app purchases API](https://developer.samsung.com/iap/programming-guide.html)
*    Uses the term “Item” for items.
*    Configure each item in the server UI to be either “consumable” or “one-time purchase”. Call “consume” to consume a consumable item. No acknowledgement required for one-time purchase items.

## Open questions
*   Please check our [issue tracker](https://github.com/WICG/digital-goods/issues).

## Resolved issues
*   Do we need to support [pending transactions](https://developer.android.com/google/play/billing/billing_library_overview#pending)? (i.e., when your app starts, you’re expected to query pending transactions which were made out-of-app, and acknowledge them).
    *   In the Play Billing backend, this means you’re supposed to call [BillingClient.queryPurchases](https://developer.android.com/reference/com/android/billingclient/api/BillingClient#querypurchases) to get the list of pending unacknowledged transactions.
    *   See [this post](https://android-developers.googleblog.com/2019/06/advanced-in-app-billing-handling.html) for details.
    *   Added listPurchases.
*   Can we combine acknowledge() and consume()? Only reason we can see to _not_ do that is that the Play Billing implementation would not know which method to call, unless we can get it from the SkuDetails, which I don’t see a field for.
    *   It [looks like](https://developer.android.com/google/play/billing/billing_onetime) the Play Store doesn’t distinguish the two on the server. The only way to distinguish this is whether you call acknowledge() or consume().
    *   Could look at this as a Boolean option on a single method, “make\_available\_again”.
*   How should price be presented through the API? Options:
    *   As a [PaymentCurrencyAmount](https://developer.mozilla.org/en-US/docs/Web/API/PaymentCurrencyAmount) (a {3-letter currency code, string value} pair). e.g. `"price": {"currency": "USD", "value": "3.50"}`.
        *   Pro: Consistent with Payment Request and Payment Handler APIs (though note that compatibility is not needed in this case).
        *   Con: Formatting this using Intl.NumberFormat will roundtrip the value through a double, which could result in rounding errors.
    *   As a currency code and integer amount in micros. e.g. `"priceCurrency": "USD", "priceAmountMicros": 3500000`.
        *   Pro: Directly maps from the Play Billing API.
        *   Con: Formatting this using Intl.NumberFormat requires dividing by 1000000 (into a double), which could result in rounding errors.
    *   As an already-formatted string. e.g. `"price": "$3.50"`.
        *   Pro: No formatting / roundtripping needed on the site.
        *   Pro: Play Billing _may_ have a policy that the string needs to be displayed exactly as given by this API, and this is the only way to guarantee that.
        *   Con: No access to the value numerically, or the currency code. No way to distinguish, e.g., USD and AUD.
        *   Con: Difficult to standardize the format of the string (lots of complexity around locale). Hard questions around whether we just pull the price() string straight out of Billing API, or whether the user agent is expected to roundtrip it to conform to a more concrete set of rules.
        *   Con: No way to localize to the user’s locale, which might be important. For example, formatting EUR in en-UK looks like “€3.50”, while the same currency in de-DE looks like “3,50€”.
