

Responses to the [Self-Review Questionnaire: Security and Privacy](https://w3ctag.github.io/security-questionnaire/) for the [Digital Goods API](https://github.com/WICG/digital-goods/)


## 2.1 What information might this feature expose to Web sites or other parties, and for what purposes is that exposure necessary?

The API allows an origin to query the details of digital goods available from a store backend, for that origin and user.
It exposes the user's chosen currency (as part of the price details) and which items have been purchased.
This information is needed for the origin to display purchases to the user and give the user access to features/purchases based on purchases made.

The API also allows an origin to know that a purchase has been acknowledged, which is necessary to confirm that a transaction has completed.

## 2.2 Is this specification exposing the minimum amount of information necessary to power the feature?

Yes.

## 2.3 How does this specification deal with personal information or personally-identifiable information or information derived thereof?

The API involves an already-authenticated user and their purchases from that origin, but doesn't expose any other information about the user. No information is persisted.

## 2.4 How does this specification deal with sensitive information?

No special treatment.

## 2.5 Does this specification introduce new state for an origin that persists across browsing sessions?

Yes, but this state cannot be created explicitly - the user has to buy something through a separate API for state to show in this API.

## 2.6 What information from the underlying platform, e.g. configuration data, is exposed by this specification to an origin?

The origin will be able to make deductions from the presence or absence of the API.
For example, a store backend may be available in certain contexts only, or a user agent may support the API on specific platforms only.

## 2.7 Does this specification allow an origin access to sensors on a user’s device

No.

## 2.8 What data does this specification expose to an origin? Please also document what data is identical to data exposed by other features, in the same or different contexts.

Data about the user's digital goods purchases. If those purchases were made through the PaymentRequest API on the origin then it would have access to that information already.

## 2.9 Does this specification enable new script execution/loading mechanisms?

No.

## 2.10 Does this specification allow an origin to access other devices?

No.

## 2.11 Does this specification allow an origin some measure of control over a user agent’s native UI?

No.

## 2.12 What temporary identifiers might this this specification create or expose to the web?

None.

## 2.13 How does this specification distinguish between behavior in first-party and third-party contexts?

Supporting the API in third-party contexts is probably not necessary?

## 2.14 How does this specification work in the context of a user agent’s Private Browsing or "incognito" mode?

The specification assumes an authenticated user, which is not usually the case in incognito mode.
The user agent _should_ act as if there are no available payment methods in incognito mode.

## 2.15 Does this specification have a "Security Considerations" and "Privacy Considerations" section?

Yes. https://github.com/WICG/digital-goods/blob/master/explainer.md#security-and-privacy-considerations

## 2.16 Does this specification allow downgrading default security characteristics?

No.

## 2.17 What should this questionnaire have asked?

No more questions.
