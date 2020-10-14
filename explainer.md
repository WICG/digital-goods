# Digital Goods API - Explainer

Authors: Matt Giuca \<<mgiuca@chromium.org>\>,
         Glen Robertson \<<glenrob@chromium.org>\>,
         Jay Harris \<<harrisjay@chromium.org>\>

This document proposes the Digital Goods API for querying and managing digital products to facilitate in-app purchases from web applications, in conjunction with the [Payment Request API](https://www.w3.org/TR/payment-request/) (which is used to make the actual purchases). The API would be linked to a digital distribution service connected to via the user agent.


## The problem

The problem this API solves is that Payment Request by itself is inadequate for making in-app purchases in existing digital stores, because that API simply asks the user to make a payment of a certain amount (e.g., “Please authorize a transaction of US$3.00”), which is sufficient for websites selling their own products, but established digital distribution services require apps to make purchases by item IDs, not monetary amounts (e.g., “Please authorize the purchase of SHINY\_SWORD”); the price being configured per-region on the backend.

The Payment Request API can be used, with [a minor modification](https://github.com/w3c/payment-request/issues/912), to make in-app purchases, using the digital distribution service as a payment method, by supplying the desired item IDs as `data` in the `modifiers` member for that particular payment method. However, there are ancillary operations relating to in-app purchases that are not part of that API:

*   Querying the details (e.g., name, description, regional price) of digital items from the store backend.
    *   Note: Even though the web app developer is ultimately responsible for configuring these items on the server, and could therefore keep track of these without an API, it is important to have a single source of truth, to ensure that the price of items displayed in the app exactly matches the prices that the user will eventually be charged, especially as prices can differ by region, or change at planned times (such as when sale events begin or end).
*   Consuming or acknowledging purchases. Digital stores typically do not consider a purchase finalized until the client acknowledges the purchase through a separate API call. This acknowledgment is supposed to be performed once the client “activates” the purchase inside the app.

It is typically a requirement for listing an application in a digital store that in-app purchases are made through that store’s billing API. Therefore, access to these operations is a requirement for web apps to be listed in various application stores, if they wish to sell in-app products.

## The proposed API

The Digital Goods API allows the user agent to provide the above operations, alongside digital store integration via the Payment Request API.

Sites using the proposed API would still need to be configured to work with each individual store they are listed in, but having a standard API means they can potentially have that integration work across multiple browsers. This is similar to how the existing Payment Request API works (sites still need to integrate with each payment provider, e.g., Google Pay, Apple Pay, but their implementation is browser agnostic).

Usage of the API would begin with a call to `Window.getDigitalGoodsService()`, which returns a promise yielding null if there is no DigitalGoodsService:

```js
const itemService = await getDigitalGoodsService("https://example.com/billing");
if (itemService === null) {
    // Our preferred item service is not available.
    // Use a normal web-based payment flow.
    return;
}
```

### Querying item details

The `getDetails` method returns server-side details about a given set of items, intended to be displayed to the user in a menu, so that they can see the available purchase options and prices without having to go through a purchase flow.


```js
details = await itemService.getDetails(['shiny_sword', 'gem']);
for (item in details) {
  const priceStr = new Intl.NumberFormat(
      locale,
      {style: 'currency', currency: item.price.currency}
    ).format(item.price.value);
  AddShopMenuItem(item.id, item.title, priceStr, item.description);
}
```


The returned `itemDetails` sequence may be in any order and may not include an item if it doesn't exist on the server (i.e. there is not a 1:1 correspondence between the input list and output).
 
The item ID is a string representing the primary key of the items, configured in the store server. There is no function to get a list of item IDs; those should be hard-coded in the client code or fetched from the developer’s own server.

The item’s `price` is a <code>[PaymentCurrencyAmount](https://developer.mozilla.org/en-US/docs/Web/API/PaymentCurrencyAmount)</code> containing the current price of the item in the user’s current region and currency. It is designed to be formatted for the user’s current locale using <code>[Intl.NumberFormat](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/NumberFormat)</code>, as shown above.

### Making a purchase

The purchase flow itself uses the [Payment Request API](https://w3c.github.io/payment-request/). We don’t show the full payment request code here, but note that the item ID for any items the user chooses to purchase should be sent in the `data` field of a `modifiers` entry for the given payment method, in a manner specific to the store. For example:

```js
new PaymentRequest(
  [{supportedMethods: 'https://example.com/billing',
    data: {itemId: item.id}}]);
```

Note that as part of this proposal, we are proposing to [remove the requirement](https://github.com/w3c/payment-request/issues/912) of the `total` member of the `details` dictionary, since the source of truth for the item price (that will be displayed to the user in the purchase confirmation dialog) is known by the server, based on the item ID. The exact format of the `data` member is up to the store (the spec simply says this is an `object`). Some stores may allow multiple items to be purchased at the same time, others only a single item.

### Acknowledging a purchase

Some stores will require that the user acknowledge a purchase once it has succeeded. In this case, the payment response will return a `PurchaseToken`, which can be used with the `acknowledge` method.

Items that are designed to be purchased multiple times must be acknowledged with the `repeatable` flag. An example of a repeatable purchase is an in-game powerup that makes the player stronger for a short period of time. Once it is acknowledged with the `repeatable` flag, it can be purchased again.

```js
itemService.acknowledge(purchaseToken, 'repeatable');
```

Items that are designed to be purchased once and last permanently in the user’s app must be acknowledged with the `onetime` flag. An example of a one-time purchase is a “remove ads” option. Once acknowledged with the `onetime` flag, the app is expected to remember the user’s purchase and continue providing the purchased capability.


```js
itemService.acknowledge(purchaseToken, 'onetime');
```

## Full API interface


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
};

enum PurchaseType {
  "repeatable",
  "onetime",
};

dictionary ItemDetails {
  DOMString itemId;
  DOMString title;
  DOMString description;
  PaymentCurrencyAmount price;
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

This API should be used in a secure context.

This API assumes that the user agent has some existing authentication process for the user, e.g. some extra UI when the API is initialised, or some implicit platform or browsing context. Because an authenticated user is likely needed for the API to be meaningful, and information is only exposed for purchases that user has already made through a separate API, there is minimal additional potentially-identifying information to be gained through this API.

## Analysis of various APIs

### [Play Store BillingClient API](https://developer.android.com/reference/com/android/billingclient/api/BillingClient)
*    Uses the term “SKU” for items.
*    No server-side distinction between “consumable” and “one-time purchase” items. Choice is dynamic: call “consume” to consume an item and make it available for purchase again, call “acknowledge” to acknowledge purchase of a one-time item and not make it available again.

### [Samsung in-app purchases API](https://developer.samsung.com/iap/programming-guide.html)
*    Uses the term “Item” for items.
*    Configure each item in the server UI to be either “consumable” or “one-time purchase”. Call “consume” to consume a consumable item. No acknowledgement required for one-time purchase items.

## Open questions

*   Do we need to support [pending transactions](https://developer.android.com/google/play/billing/billing_library_overview#pending)? (i.e., when your app starts, you’re expected to query pending transactions which were made out-of-app, and acknowledge them).
    *   In the Play Billing backend, this means you’re supposed to call [BillingClient.queryPurchases](https://developer.android.com/reference/com/android/billingclient/api/BillingClient#querypurchases) to get the list of pending unacknowledged transactions.
    *   See [this post](https://android-developers.googleblog.com/2019/06/advanced-in-app-billing-handling.html) for details.

## Resolved issues
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
