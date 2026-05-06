# SDKDocumentation

## Introduction

Welcome to the WhiteBird SDK documentation!

This SDK provides the ability to integrate **WhiteBird instant crypto exchange** functionality directly into your website (JS SDK).\
Authorization flow depends on the selected SDK mode.

The SDK supports interaction with WhiteBird in three modes:

* LoginMode
* AuthMode
* TokensMode

#### Presentation

* You can review the visual flow in [Figma](https://www.figma.com/design/QhTl1W0BEncjvGXRu03UiW/SDK-flow?node-id=0-1\&p=f)
* Demo is available at: [SDK Demo](https://sdk.dev.wbdevel.net/v2.0/assets/sdk-demo/index.html)\
  (`MerchantId: 11111111-1111-1111-1111-111111111111`)

Descriptions of each mode are available [below](./#brief-description-of-modes) in this documentation.

## High-level process overview

### General WhiteBird SDK flow

### One-line flow

`Initialize SDK -> resolve mode -> apply status gates -> allow operations`

### Scenarios

* **LoginMode:** SDK handles user auth flow, then routes by status.
* **AuthMode:** SDK handles auth, returns tokens via `onLogin(...)`, then applies status gates.
* **TokensMode:** SDK starts with provided tokens, skips login screen, then applies status gates.

### Status routing table

| Condition                                                    | SDK behavior             | Result                                      |
| ------------------------------------------------------------ | ------------------------ | ------------------------------------------- |
| `NOT_VERIFIED`                                               | Verification is required | Agreements -> SumSub                        |
| `PENDING`                                                    | Access is limited        | Wait AML/KYC decision                       |
| `VERIFIED` + `testingNeeded=true` + `testingCompleted=false` | Crypto test gate         | Complete crypto test                        |
| `VERIFIED` + compliance checks passed                        | Full access              | Exchange / wallet operations                |
| `FROZEN`                                                     | Restricted mode          | Operations are limited or blocked by policy |
| `ARREST`                                                     | Blocked mode             | Exchange access denied                      |

***

### Registration process

### One-line flow

`Sign up -> agreements -> email confirmation -> conditional phone verification -> register -> auto-login`

### Scenarios

* User submits sign-up form and accepts agreements.
* Email confirmation is required.
* For `countryCode == BY`, SMS phone confirmation is required before completion.
* For non-BY users, registration proceeds without mandatory SMS step.
* On success, SDK performs login and continues with authorized session.

### Registration routing table

| Condition            | Behavior              | Result                     |
| -------------------- | --------------------- | -------------------------- |
| Email not confirmed  | Registration paused   | Stay in confirmation step  |
| `countryCode == BY`  | Phone code required   | Continue after SMS confirm |
| `countryCode != BY`  | No mandatory SMS gate | Continue to registration   |
| Registration success | Auto-login            | Tokens/session created     |

***

### Verification process

### One-line flow

`Agreements -> SumSub KYC -> AML review -> verified status -> optional crypto-test gate`

### Scenarios

* User confirms legal statements and offer agreement.
* SDK runs SumSub (documents + liveness).
* User enters AML pending state (`PENDING`) until decision.
* If approved, user becomes `VERIFIED`.
* If crypto test is required, it must be completed before full access.

### Verification routing table

| Condition                     | Behavior             | Result                    |
| ----------------------------- | -------------------- | ------------------------- |
| Agreements not confirmed      | Verification blocked | Stay on agreements        |
| SumSub completed, AML pending | Limited access       | `PENDING`                 |
| AML approved                  | Verification done    | `VERIFIED`                |
| Crypto test required          | Additional gate      | Complete test to continue |

***

### Authorization process

### One-line flow

`Credentials -> MFA/Captcha checks -> token issuance -> status-based access`

### Scenarios

* User signs in with email/password.
* If MFA is enabled, 2FA code is required.
* If captcha is required, captcha challenge must pass.
* On success, tokens are issued.
* Access to operations still depends on verification/compliance status.

### Authorization routing table

| Condition           | Behavior         | Result                                 |
| ------------------- | ---------------- | -------------------------------------- |
| Invalid credentials | Auth fails       | Stay on sign-in                        |
| MFA enabled         | 2FA required     | Continue after valid code              |
| Captcha required    | Captcha required | Continue after valid challenge         |
| Auth success        | Tokens issued    | Authorized session (with status gates) |

## Brief description of modes

Terminology:

* Client – the company integrating the SDK
* User – the end user of the product

### LoginMode

This mode is intended for user authentication _(authorization/registration/verification)_ via WhiteBird. It allows users to log in with WhiteBird credentials, perform exchange operations, view their transaction history, and contact support. Suitable for scenarios where you need to add the ability to deposit/withdraw funds or exchange cryptocurrency to fiat and back.

### AuthMode

In this mode, the SDK is used for user authentication _(authorization/registration/verification)_ via WhiteBird and provides user authorization tokens, which can be used for interaction with the WhiteBird API. In this case, the client is **not** a _user identification agent_ for us, and implements their own custom UI for exchange, which makes operations through our API using the client’s authorization tokens.

### TokensMode

This mode is designed to run our SDK with access tokens. It includes all the capabilities of LoginMode but removes WhiteBird authorization.\
It implies a seamless transition from the client’s app to the WhiteBird platform and back. In this mode, the client is assumed to be a _user identification agent_ for us.

## User identification agent

The main feature enabled by this is the use of SDK in TokensMode. The user will not need to log into the WhiteBird platform themselves; the client does this on their behalf, obtaining Auth tokens through **backend-to-backend** interaction over REST API, using the merchant’s API Key.

## Integration scenarios

### 1) LoginMode — WhiteBird handles everything

Use this scenario when the partner wants WhiteBird to fully own the user flow inside SDK.

**Flow**

1. Partner opens WhiteBird SDK in `LoginMode`.
2. User signs in or signs up inside SDK.
3. User completes onboarding steps inside SDK.
4. User completes KYC / compliance / crypto test steps if required.
5. SDK routes the user according to status and compliance rules.
6. Once all required checks are completed, SDK opens exchange flow.

**Example**

```js
wbExchangeSdk.setup({
  // required params:
  el: document.getElementById("wbExchangeSdkWrapper"),
  mode: wbExchangeSdk.mode.LoginMode,
  merchantId: "xxxx",
  merchantPass: "xxxx", // parameter for SDK, used as Authorization: Basic <merchantPass>. Needed to generate accessToken and refreshToken

  // LoginMode
  onUserData: ({ email, accessToken, refreshToken }) => {
    console.log("", email, accessToken, refreshToken);
  },
});
```

### 2) API register + tokens + LoginMode — partner-authenticated LoginMode flow

Use this scenario when the partner performs registration and login on its side, generates WhiteBird client tokens backend-to-backend, and opens SDK in an already authenticated context.

**Backend flow**

1. Partner registers the user backend-to-backend in WhiteBird with a lightweight payload (`email`, `phone`):\
   `POST /api/v2/auth/merchant/client/register`
2. Partner receives `clientId` (`{ id, status }`).
3. Partner generates tokens backend-to-backend for this client (**required**):\
   `POST /api/v2/auth/merchant/client/token/generate`

**User flow**

1. Partner opens WhiteBird SDK in `LoginMode` with `isAuthAgent: true` and generated tokens.
2. User enters SDK in partner-authenticated context.
3. SDK applies onboarding / KYC / compliance routing.
4. After all required checks are completed, exchange becomes available in SDK.

**Example**

```js
wbExchangeSdk.setup({
  // required params:
  el: document.getElementById("wbExchangeSdkWrapper"),
  mode: wbExchangeSdk.mode.LoginMode,
  merchantId: "xxxx",
  merchantPass: "xxxx", // parameter for SDK, used as Authorization: Basic <merchantPass>. Needed to generate accessToken and refreshToken

  // partner-generated client tokens
  accessToken: "****",
  refreshToken: "****",

  // required for this flow behavior
  isAuthAgent: true,

  // LoginMode callback
  onUserData: ({ email, accessToken, refreshToken }) => {
    console.log("", email, accessToken, refreshToken);
  },
});
```

### 3) API KYC register + tokens + TokensMode — partner is an identification agent

Use this scenario when the partner already has full KYC/PID data and sends it to WhiteBird. Partner owns onboarding and KYC on its side, and SDK is used only for final compliance checks and exchange access.

**Backend flow**

1. Partner registers user backend-to-backend with full KYC payload:\
   `POST /api/v2/kyc/merchant/client/register`
2. Partner receives `clientId` (`{ id, status }`).
3. (Optional) [Partner checks current status](<Registration API.md#id-2.-client-status>):\
   `POST /api/v2/kyc/merchant/client/status`
4. Partner generates tokens backend-to-backend:\
   `POST /api/v2/auth/merchant/client/token/generate`
5. Partner starts SDK in `TokensMode` with issued tokens.

**User flow**

1. User enters SDK already authorized by tokens (no login / sign-up screen).
2. SDK applies status / compliance gates.
3. If any remaining verification step is required, user completes it in SDK.
4. After required checks are completed, exchange becomes available in SDK.

**Example**

```js
wbExchangeSdk.setup({
  // required params:
  el: document.getElementById("wbExchangeSdkWrapper"),
  mode: wbExchangeSdk.mode.TokensMode,
  merchantId: "xxxx",
  merchantPass: "xxxx", // parameter for SDK, used as Authorization: Basic <merchantPass>. Needed to generate accessToken and refreshToken

  // TokensMode
  accessToken: "****",
  refreshToken: "****",
});
```

### 4) AuthMode — onboarding and KYC in WhiteBird, then API integration with client tokens

Use this scenario when the partner wants WhiteBird to handle authentication, onboarding, and KYC, and then continue via client tokens and client endpoints on partner side.

**Flow**

1. Partner starts SDK in `AuthMode`.
2. User signs in or signs up inside SDK.
3. User completes onboarding and KYC inside WhiteBird SDK.
4. After successful authorization, SDK returns client tokens via `onLogin(...)`.
5. Partner uses these client tokens for further API integration through client endpoints.

**Example**

```js
wbExchangeSdk.setup({
  // required params:
  el: document.getElementById("wbExchangeSdkWrapper"),
  mode: wbExchangeSdk.mode.AuthMode,
  merchantId: "xxxx",
  merchantPass: "xxxx", // parameter for SDK, used as Authorization: Basic <merchantPass>. Needed to generate accessToken and refreshToken

  // AuthMode
  onLogin: ({ email, accessToken, refreshToken, isUserVerified }) => {
    console.log("", email, accessToken, refreshToken, isUserVerified);
  },
});
```

### 5) API register + tokens + AuthMode — login/onboarding on partner side, KYC in WhiteBird, merchant API after that

Use this scenario when the partner keeps login and onboarding on its side, delegates KYC UX to WhiteBird via SDK, and continues the operational flow via merchant endpoints.

**Backend flow**

1. Partner registers user backend-to-backend in WhiteBird:\
   `POST /api/v2/auth/merchant/client/register`
2. Partner generates tokens backend-to-backend:\
   `POST /api/v2/auth/merchant/client/token/generate`

**User flow**

1. Partner handles login and onboarding on its own side.
2. Partner opens WhiteBird SDK in `AuthMode` for KYC / compliance UX.
3. User completes required KYC / compliance steps in WhiteBird.
4. SDK returns client tokens via `onLogin(...)` if they are needed for follow-up steps.
5. Partner continues operational flow via merchant endpoints (server-to-server with `x-api-key`).

Repeated logins usually repeat the steps of token generation -> SDK open.

**Example**

```js
wbExchangeSdk.setup({
  // required params:
  el: document.getElementById("wbExchangeSdkWrapper"),
  mode: wbExchangeSdk.mode.AuthMode,
  merchantId: "xxxx",
  merchantPass: "xxxx", // parameter for SDK, used as Authorization: Basic <merchantPass>. Needed to generate accessToken and refreshToken

  // AuthMode
  onLogin: ({ email, accessToken, refreshToken, isUserVerified }) => {
    console.log("", email, accessToken, refreshToken, isUserVerified);
  },
});
```

## Adding to a website

### Connection via CDN

```html
<script src="https://sdk.dev.wbdevel.net/v2.0/integration/wbExchangeSdk-v001.js"></script>
```

### Container

* The SDK content is designed for mobile view – the container should not be less than 360px wide.
* The SDK container can be placed anywhere on the website.
* Pass the HTML element of this container in the configuration.
* !! The SDK does not modify the container’s styles, all container styling is handled by the application.
* The SDK iframe takes up all available space within the container.

## Merchant go-live checklist

Use this checklist as the final setup gate before switching the merchant environment to production.

1. **Initialize SDK:** production `merchantId` + `merchantPass` are configured, production SDK URL is used, container is at least `360px`, and `wbExchangeSdk.setup(...)` opens without runtime errors. [General flow details](./#general-whitebird-sdk-flow)
2. **Resolve mode:** selected mode (`LoginMode`, `AuthMode`, `TokensMode`) matches the merchant architecture, and only mode-relevant params are passed (`accessToken`, `refreshToken`, `isAuthAgent`, callbacks). [Modes](./#brief-description-of-modes) | [Integration scenarios](./#integration-scenarios)
3. **Apply status gates:** merchant UI and backend correctly process `NOT_VERIFIED`, `PENDING`, `VERIFIED`, `FROZEN`, `ARREST`, and crypto-test-required users. [Status routing table](./#status-routing-table)
4. **Allow operations:** exchange and wallet actions are enabled only after all required gates pass; blocked states do not allow restricted operations. [General flow details](./#general-whitebird-sdk-flow)
5. **Registration flow:** sign-up entry point is available, agreements are shown/accepted, email confirmation is completed, and phone verification for `countryCode == BY` is handled correctly.  [Registration API](<Registration API.md>)
6. **Verification flow:** `agreements -> SumSub -> AML review` route works as expected, including `PENDING` waiting state and transition to `VERIFIED`. [Verification process](./#verification-process) | [Verification routing table](./#verification-routing-table)
7. **Optional crypto test:** when `testingNeeded=true` and `testingCompleted=false`, user is routed to test gate and gains full access only after completion. [Verification routing table](./#verification-routing-table)
8. **Authorization flow:** credentials login works, MFA/Captcha branches are supported when enabled, and token issuance completes without exposing merchant secrets in frontend. [Authorization process](./#authorization-process) | [Authorization routing table](./#authorization-routing-table)
9. **Callbacks and token lifecycle:** `onLogin` / `onUserData`, `onOrderCreated`, `onOrderCompleted`, `onExit` are wired; token refresh/re-generation is handled for repeated session entry. [Optional SDK parameters](./#optional-sdk-configuration-parameters) | [Token generation API](<Registration API.md#id-5.-sdk-token-generation>)

## Optional SDK configuration parameters

**externalClientId** - string, should be provided by the merchant to link WhiteBird users with merchant’s users.

**email** - string, allows pre-filling the email field to reduce user actions during WhiteBird login.

**currencyAmount** - int, allows pre-filling the currency amount for the exchange.

**currencyFrom** - string from Currency enum. Allows pre-filling the currency to be exchanged from.

**disableCurrencyFrom** - bool, disable currencyFrom selector, also it blocks amount field (if it was provided).

**currencyTo** - string from Currency enum. Allows pre-filling the currency to be exchanged to.

**disableCurrencyTo** - bool, disable currencyTo field, also it blocks cryptoWallet field (if it was provided).

**cryptoWallet** - string, allows pre-filling the user’s wallet field.

**showBackButtonOnHomePage** - bool, shows a "back" button in our UI and reacts to it using the onExit callback.

**onExit** - () -> void, callback to handle "back" button press.

**onOrderCreated** - ({orderId, internalCryptoAddress}) -> void, callback for order creation notification for merchant.

**onUserData** - ({ email, accessToken, refreshToken }) => {} -> callback is called in LoginMode when receiving user data from the SDK.

**currencyToAmount** - int, allows pre-filling the target amount for exchange.

**disableAmount** - bool, disables amount input editing in SDK UI.

**isAuthAgent** - bool, indicates partner identification-agent mode behavior.

**redirectUrl** - string, URL used by SDK for redirect/navigation after specific flow actions.

**startAppPage** - string, allows starting SDK from a specific internal page/state.

**disableAddCard** - bool, disables add-card flow in SDK UI.

**onOrderCompleted** - ({orderId, status}) -> void, callback for order completion notification for merchant.

**refId** - string, optional external reference identifier passed through SDK context.

**debug** - bool, enables SDK debug logging in browser console.

**isTgBot** - bool, enables Telegram WebApp-specific behavior for link opening.

It is recommended to call `wbExchangeSdk.cleanup()` after finishing work with the SDK.

```typescript
let currencyFrom: Currency;
let currencyTo: Currency;

enum Currency {
    BYN,
    RUB,
    USD,
    EUR,
    BTC,
    ETH,
    USDT, // ERC-20
    USDC, // ERC-20
    TRX,
    USDT_TRC, // TRC-20
    TON,
    USDT_TON, // TON
    WBP, // TRC-20
}
```
