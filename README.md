# Purs Developer Documentation

## Introduction

- This document provides a comprehensive guide for onboarding your merchants and connecting their accounts with **Purs** and start accepting payments.
- Before a merchant can link their account with **Purs** via your platform, a few prerequisites must be completed.

## Prerequisites

### **CLIENT_ID** and **CLIENT_SECRET**

- Purs will provide you with a **CLIENT_ID** and **CLIENT_SECRET.**

    > **Note**: The **CLIENT_ID** and **CLIENT_SECRET** should be stored securely on your servers. These credentials should never be passed to your front-end clients.

### **REDIRECT_URL**

- You will need to provide Purs with a callback URL (referred to as **REDIRECT_URL**) which you own. Purs will redirect authenticated merchants to this **REDIRECT_URL** along with additional query parameter called `code`. The usage of this `code` is explained in the OAuth2 Flow section.

    > **Example**: If your merchants portal URL is `https://merchants.cann-x.com` then the REDIRECT_URL could be `https://merchants.cann-x.com/callback`

## Key Platform Operations

There are two primary operations that the Cann-X platform will need to support:

<details><summary><h1><b>OAuth2 Flow</b></h1></summary>

- This allows merchants to connect their Purs Merchant Account with Cann-X. Once connected, this merchant can accept Purs payments on the Cann-X platform. This operation only needs to be completed once for each Cann-X merchant.

### **Checkout Flow**

- This process allows Cann-X to create Purs payment requests for Cann-X customers to complete. This operation will be done each time a transaction is to be completed on the Cann-X platform.

## Environment URLs

> ⚠️ **Important**: Because the "Purs Checkout Widget URL" is currently inactive in Production, you won't be able to complete payment requests in Production just yet. When you are ready to go live, we will activate this URL.

| Environment | **BASE_URL** | **OAUTH_URL**  | Purs Merchant Portal URL |
| --- | --- | --- | --- |
| **Sandbox** | `sandbox-api.purs.digital` | `sandbox-auth.purs.digital` | `sandbox-merchants.purs.digital` |
| **Production** | `api.purs.digital` | `auth.purs.digital` | `merchants.purs.digital` |

## OAuth2 Flow

The process of linking a merchant account with **Purs** will ad to the standard **OAuth2** authentication protocol.

### Diagram

![OAuth Flow Diagram](https://github.com/user-attachments/assets/2b61ba33-adaf-4be3-a2bc-24ef0f61cf4e)



> **Note**: for **Sandbox** testing, you will need to create a dummy merchant account in the Purs **Sandbox**. Navigate to the URL listed in the Environment URLs table and create a dummy Purs merchant account.

### Initiate OAuth2 Authorization

- To connect a seller's Purs Merchant Account with their Cann-X seller account, Cann-X will need to have a "Connect with Purs" button (likely somew in the admin portal for your merchants).
- When clicked, this button should navigate to the `https://{OAUTH_URL}/oauth2/authorize` URL with the appropriate query parameters.
- The merchant will be prompted to enter their Purs Merchant Portal login credentials.
- Once they authenticate, they will be redirected to the **REDIRECT_URL** Cann-X provided Purs. An extra query parameter will be present when the seller is redirected — a query parameter called `code`.
- **Endpoint details for `/oauth2/authorize` - [here](#OAuth2-Authorization)**
- See the next section to understand what to do with the `code` that is provided by Purs as a query parameter attached to your **REDIRECT_URL**.

### Retrieve and Store Tokens

- Extract the value of this `code` query parameter and make a `POST` request to Purs to exchange this short-lived `code` for OAuth tokens.
- You will need the **CLIENT_ID** and **CLIENT_SECRET** which Purs has provided you.
- Make sure to make this request from your backend where the **CLIENT_ID** and **CLIENT_SECRET** are stored securely.
- **Endpoint details for `/oauth/token` - [here](#Get-new-tokens)**

### Refresh Tokens

- Since the `access_token` and `id_token` expire, you should refresh them with `refresh_token` to make valid requests.
- **Endpoint details for `/oauth/token` (refresh) - [here](#Redresh-tokens)**

### Revoke Tokens

- This is to revoke the tokens for a particular merchant.
- **Endpoint details for `/oauth/revoke` - [here](#Revoke-tokens)**
</details>

<details><summary><h1><b>Checkout Flow</b></h1></summary>

There are 2 steps in this process  in the sequence diagram below.

- 🟧 - Getting the Purs Checkout Widget URL.
- 🟩 - Calling the `PursCheckoutWidget` method with the URL received from the above step.

![Subscription](https://github.com/user-attachments/assets/4680586b-3317-47b1-b877-3115c03e1221)



### 🟧 Purs Checkout Widget URL

- Purs checkout widget is a way for Cann-X customers to make payments.
- **Endpoint details to get the Purs Checkout Widget `/v1/transactions` - [here](#New-subscription)**

    > **Note:** To make the above request, you need `location_id`. This `location_id` comes from the Purs system and how to get the `location_id` for a merchant is explained [**here**](#location-id-of-merchant).

    > **Note:** The above request should be made from your backend, not directly from your frontend. This approach ensures that the tokens and their corresponding merchant mappings, which are stored in your backend, remain secure. Your frontend should make an API call to your backend with the `amount` and `location_id` as parameters. Your backend will then handle the call to the Purs API (`/v1/transactions`) using the valid tokens stored in your system.

### 🟩 PursCheckoutWidget method

- Below is code sample to integrate the Purs checkout widget in your website.

**Step 1**

Add the Purs checkout CDN into your script tag

```html
<script src="https://purs-sandbox-cdn.s3.us-west-2.amazonaws.com/checkout/v1/index.min.js"></script>
```

**Step 2**

Add a "Pay with Purs button" on your page.

```html
<button id="purs-checkout-button">
    Pay with Purs
</button>
```

> 👍 **Recommended**: Add the Purs logo to this button. [Link](https://purs-sandbox-cdn.s3.us-west-2.amazonaws.com/checkout/v1/connect-with-purs.png)

**Step 3**

Implement the logic to call a function (`initateCheckout`) which initiates the checkout flow on a button click.

```javascript
const button = document.getElementById('purs-checkout-button');
button.addEventListener('click', initiateCheckout);
```

**Step 4**

Implement the logic to call the `PursCheckoutWidget.init` method with `url` and `onPaymentComplete` as parameters.

- the `url` takes the value of checkout url and steps to get this url are mentioned in the 🟧 [**green section**](#-purs-checkout-widget-url).
- the `onPaymentComplete` expects a callback function (`updateUI`) defined on your end.

```javascript
const initiateCheckout = async () => {
    try {
        const amount = 2000 // amount value in cents
        const response = await createPaymentRequest(amount, 'purs-location-id');
        const checkoutUrl = response

        //this is the main method to initiate the Purs Checkout Widget
        PursCheckoutWidget.init({
            url: checkoutUrl&email=`user-email-id`, // email is an optional query param passed so user doesn't have to again enter their email in Purs checkout widget
            onPaymentComplete: (paymentData) => {
                const subscription_token = paymentData?.subscriptionToken; // the subscriptionToken is an optional field returned if you pass `create_subscription`: true

                updateUI(); // Update UI is a function that you can implement which is called when the payment is completed by the checkout widget to update any UI changes on your end.
			}
        });
    }
    catch (error) {
        console.log(error);
    }
}
```

**Step 5**

Implement the logic to get the checkout url in a function. (`createPaymentRequest`)

- As mentioned [**here**](#-purs-checkout-widget-url), your frontend should make a request to your backend which in turn requests the Purs backend for the checkout url.

```javascript
const createPaymentRequest = async (amount, locationid) => {
    const response = await fetch('www.your-backend-api.com', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json'
        },
        body: JSON.stringify({
            amount: amount,
            location_id: locationid
        })
    });
    if (!response.ok) {
        throw new Error('failed to create payment')
    }
    return response.json()
}
```

**Step 6**

Implement the logic for a callback function (`updateUI`) to handle any UI changes after a successful payment.

```javascript
const updateUI = () => {
    const button = document.getElementById('purs-checkout-button');
    // Disable the button
    button.disabled = true;
    // make necessary UI changes according to your needs
};
```

> ⚠️ **Important**: Both the parameters for `PursCheckoutWidget.init` i.e. `url` and `onPaymentComplete` are required.

- Everything combined

**HTML**

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Cann X Website</title>
</head>
<body>
    <div>Cann X Website</div>
    <!-- add this button on your checkout page  -->
    <button id="purs-checkout-button">Pay with Purs</button>

    <script src="https://purs-sandbox-cdn.s3.us-west-2.amazonaws.com/checkout/v1/index.min.js"></script>
    <script src="./index.js" type="module"></script>

</body>
</html>
```

**JavaScript**

```javascript
const updateUI = () => {
    const button = document.getElementById('purs-checkout-button');
    // Disable the button
    button.disabled = true;
    // make necessary UI changes according to your needs


};

const createPaymentRequest = async (amount, locationid) => {
    const response = await fetch('www.your-backend-api.com', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json'
        },
        body: JSON.stringify({
            amount: amount,
            location_id: locationid
        })
    });
    if (!response.ok) {
        throw new Error('failed to create payment')
    }
    return response.json()
}

const initiateCheckout = async () => {
    try {
        const amount = 2000 // amount value in cents
        const response = await createPaymentRequest(amount, 'purs-location-id');
        const checkoutUrl = response

        //this is the main method to initiate the Purs Checkout Widget
        PursCheckoutWidget.init({
            url: checkoutUrl&email=`user-email-id`, // email is an optional query param passed so user doesn't have to again enter their email in Purs checkout widget
            onPaymentComplete: (paymentData) => {
                console.log('Payment completed!', paymentData);
                const subscription_token = paymentData?.subscriptionToken; // the subscriptionToken is a optional field returned if you pass `create_subscription`: true

                updateUI(); // Update UI is a function that you can implement which is called when the payment is completed by the checkout widget to update any UI changes on your end.
			}
        });
    }
    catch (error) {
        console.log(error);
    }
}

const button = document.getElementById('purs-checkout-button');
button.addEventListener('click', initiateCheckout); // call the initiateCheckout function when the button is clicked
```

**Integration steps**

> **Note:** The naming of functions in the above code sample is for illustration purpose only. You can change it accordingly. Just make sure the core logic remains same and the `PursCheckoutWidget.init` method receives the `url` and `onPaymentComplete` parameters.


### Location ID of merchant

- In the Purs system, each "Merchant" can have multiple "Locations" (typically representing a physical retail location).
- During the onboarding process, when a merchant creates an account on the Purs Merchant Portal, they are required to add at least one location. Additional locations can also be added later through the portal.
- To retrieve all locations associated with a particular merchant, use the `/v1/merchant` endpoint. This allows you to present the available locations (and other details) related to the merchant on your platform, enabling them to choose the location where they want to receive payments from your users.
- Once the merchant selects a location, you will use the corresponding `location_id` in the request body as outlined in the previous section.
- **Endpoint details to get the locations `/v1/merchant` - [here](#Merchant)**

### Transaction Status

- This is an optional but recommended step where you can make an additional API call to Purs to get the transaction status for a particular transaction.
- The`transaction_id` received in the checkout URL [**response**](#post-v1transactions) will be used to retrieve the status of that transaction.
- **Endpoint details to get transaction status `/v1/transactions/{transactionId}/status` - [here](#Transaction-verification)**

</details>

## Demo

[![Demo](https://github.com/user-attachments/assets/adb28ec8-51bf-492b-8dab-4d9ec4231d84)](https://drive.google.com/file/d/1zMIE1NFHZO3gC3ZY_Y60Esy8aUI7mM2r/view?usp=sharing)

## API Endpoints

<details><summary><h3><b>OAuth2 Authorization</b></h3></summary>

- **URL**

```javascript
// Put this URI in the "Connect with Purs" button
GET https://{OAUTH_URL}/oauth2/authorize?
		response_type=code& // leave as is. Ie. "code"
		client_id={CLIENT_ID}&
		redirect_uri={REDIRECT_URL}& // this must be URL-encoded
		state=abcdefg& // this is optional
		scope=openid+profile+email+phone+PURS_API/--- // add all required scopes
```

| **Field** | **Type** | **Description** |
| --- | --- | --- |
| **`response_type`** | `string` | **(Required)** This parameter indicates the type of response desired from the OAuth2 server. It must be included in every authorization request. Currently only "code" type supported. |
| **`client_id`** | `string` | **(Required)** The unique identifier assigned to your application by Purs. This ID is used to distinguish your application from others during the authentication process. |
| **`redirect_uri`** | `string` | **(Required)** The URI where the user will be sent after authorization. This URI must be one of the pre-registered redirect URIs for your client ID. |
| **`state`** | `string` | **(Optional)** The state parameter is an optional but highly recommended CSRF token to safeguard against Cross-Site Request Forgery attacks. It should be a unique, random string generated by your platform. The state value is passed a query param along with the **REDIRECT_URL** |
| **`scope`** | `string` | **(Required)** The scope parameter requires a space-separated list of permissions, including standard scopes (`openid`, `email`, `phone`) and custom merchant-specific scopes |

- **PURS_API/MERCHANT_READ** - Enables your application to get all locations and other related merchant information
- **PURS_API/TRANSACTIONS_READ** - Allows read access to view transaction history
- **PURS_API/TRANSACTIONS_WRITE** - Grants permission to create a single transactions
- **PURS_API/SUBSCRIPTION_READ** - Provides read access to view subscription and transaction
- **PURS_API/SUBSCRIPTION_WRITE** - Grants permission to create, cancel a subscription and related reccuring transactions

#### **Response**

- Positive response

```
HTTP/1.1 302 Found
Location: https://{REDIRECT_URL}?code=a1b2c3d4-5678-90ab-cdef-EXAMPLE11111&state=abcdefg
```

- Negative responses

```
// The following is the response to an example request with incorrect formatting.
HTTP 1.1 302 Found Location: https://{REDIRECT_URL}?error=invalid_request
```

```
// If the client requests code in response_type, but doesn't have permission for these requests.
HTTP 1.1 302 Found Location: https://{REDIRECT_URL}?error=unauthorized_client
```

```
// If the requested scopes are unknown, malformed, or not valid.
HTTP 1.1 302 Found Location: https://{REDIRECT_URL}?error=invalid_scope
```
</details>

<details><summary><h3><b>Get new tokens</b></h3></summary>

- **URL**

```
POST https://{OAUTH_URL}/oauth2/token
```

- **Headers**

```javascript
{
    "Content-Type": "application/x-www-form-urlencoded",
    "Authorization": "Basic <Base64Encode(<CLIENT_ID>:<CLIENT_SECRET>)>"
}
```

- **Body (form-urlencoded, not JSON)**

```
grant_type=authorization_code& // leave as is. Ie. "authorization_code"
client_id=<CLIENT_ID>&
code=<code>& // from query parameter
redirect_uri=<REDIRECT_URL> // this needs to URL encoded
```

- **Base64Encode example**

```javascript
const CLIENT_ID = 'dummy-client-id#1234'
const CLIENT_SECRET = 'dummy-client-secret#4321'
const authToken = `${CLIENT_ID}:${CLIENT_SECRET}`
const base64Encode = btoa(authToken); // use this value in the Authorization header
```

#### **Response**

- Positive response

```javascript
{
  "access_token": "eyJra1example",
  "id_token": "eyJra2example",
  "refresh_token": "eyJj3example",
  "expires_in": 86400 // expiry of access_token and id token, value in seconds
}
```

- Negative responses

```javascript
{
  "error":"invalid_request|invalid_client|invalid_grant|unauthorized_client|unsupported_grant_type"
}
```

> Make sure to store these tokens securely in you system and note that every set of tokens are unique for a particular merchant.

> **Note**: The `access_token` and `id_token` has expiry duration of 1 day (86400 secs) and the `refresh_token` has expiry duration of 10 years.

> **Note**: Since the `access_token` and `id_token` expire, you can either run a daily cron job to refresh the tokens **OR** during a request call, check the expiry of tokens with the help of `expires_in` key and if the tokens are expired, refresh them before making the request call.
</details>

<details><summary><h3><b>Refresh tokens</b></h3></summary>

- **URL**

```
POST https://{OAUTH_URL}/oauth2/token
```

- **Headers**

```javascript
{
    "Content-Type": "application/x-www-form-urlencoded",
    "Authorization": "Basic <Base64Encode(<CLIENT_ID>:<CLIENT_SECRET>)>"
}
```

- **Body (form-urlencoded, not JSON)**

```
grant_type=refresh_token& // leave as is. Ie. "refresh_token"
refresh_token=<refresh_token>
```

#### Response

- Positive response

```javascript
{
  "access_token": "new1example",
  "id_token": "new2example",
  "expires_in": 86400 // expiry of access_token and id token, value in seconds
}
```

- Negative response

```javascript
{
  "error":"invalid_request"
}
```
</details>

<details><summary><h3><b>Revoke tokens</b></h3></summary>

- **URL**

```
POST https://{OAUTH_URL}/oauth2/revoke
```

- **Headers**

```javascript
{
    "Content-Type": "application/x-www-form-urlencoded",
    "Authorization": "Basic <Base64Encode(<CLIENT_ID>:<CLIENT_SECRET>)>"
}
```

- **Body (form-urlencoded, not JSON)**

```
token=<refresh_token> // the refresh token of the merchant you want to revoke all the tokens.
```

#### Response

- Positive response

```
A successful response contains an empty body
```

- Negative response

```javascript
{
  "error":"invalid_request|unsupported_token_type|invalid_client"
}
```
</details>


<details><summary><h3><b>New subscription</b></h3></summary>

#### Request for creating a new recurrent payment subscription

- **URL**

```
POST https://{BASE_URL}/v1/transactions
```

- **Headers**

```javascript
{
    "x-access-token": "<access_token>", // access_token obtained in the OAuth2 flow unique for every merchant
    "Authorization": "Bearer <id_token>" // id_token obtained in the OAuth2 flow unique for every merchant
}
```

- **Body (JSON)**

```javascript
{
    "amount": amount, // The amount in cents to be immediately paid by the user (could be 0)  (Integer)
    "location_id": <purs_location_id>, // The ID of the merchant location where the subscription will be created (String)
    "create_subscription": true // Subscription flag
}
```

#### Response

- Positive response

```javascript
{
    "url": "https://{CHECKOUT_URL}?tid=abcd1234",
    "transaction_id": "abcd1234",
}
```

- Negative responses

| `status code` | `message` |
| --- | --- |
| 400 | The amount value is not an integer, less than 0, or greater than 100000. |
| 401 | The bearer token is not valid. |
| 404 | Location not found. |
| 500 | Internal server error |

```javascript
{
    "status_code": "<status_code>",
    "message": "<message>"
}
```
</details>

<details><summary><h3><b>Subscription verification</b></h3></summary>

#### Check user's subscription and account balance

- **URL**

```
POST https://{BASE_URL}/v1/transactions/subscription-check
```

- **Headers**

```javascript
{
    "x-access-token": "<access_token>", // access_token obtained in the OAuth2 flow unique for every merchant
    "Authorization": "Bearer <id_token>" // id_token obtained in the OAuth2 flow unique for every merchant
    "x-subscription-token": "<subscription_token>" // Token returned by Purs during user subscription process to confirm recurrent payment
}
```

- **Body (JSON)**

```javascript
{
    "amount": amount, // Optional. Verifies that the user has at least this amount in their bank account
}
```

#### Response

- Positive response

```javascript
{
    "created_at_datetime": "2024-05-05T11:00:00.000Z",
    "account_nickname": "User's account",
    "account_last_four": "1234",
    "amount_verified": true/false      // If the amount was passed in the request
}
```

- Negative responses

| `status code` | `message` |
| --- | --- |
| 401 | The bearer token is not valid. |
| 404 | Subscription not found or canceled. |
| 404 | User does not have active bank account. |
| 500 | Internal server error |

```javascript
{
    "status_code": "<status_code>",
    "message": "<message>"
}
```
</details>

<details><summary><h3><b>New recurring payment</b></h3></summary>

#### Request for creating recurrent payment

- **URL**

```
POST https://{BASE_URL}/v1/transactions/auto-approve
```

- **Headers**

```javascript
{
    "x-access-token": "<access_token>", // access_token obtained in the OAuth2 flow unique for every merchant
    "Authorization": "Bearer <id_token>" // id_token obtained in the OAuth2 flow unique for every merchant
    "x-subscription-token": "<subscription_token>" // Token returned by Purs during user subscription process to confirm recurrent payment
}
```

- **Body (JSON)**

```javascript
{
    "amount": amount, // The amount in cents to be paid by the user (non-zero)  (Integer)
}
```

#### Response

- Positive response

```javascript
{
    "transaction_id": "abcd1234"
}
```

- Negative responses

| `status code` | `message` |
| --- | --- |
| 400 | The amount value is not an integer, non positive, or greater than 100000. |
| 401 | The bearer token is not valid. |
| 404 | Location not found. |
| 500 | Internal server error |

```javascript
{
    "status_code": "<status_code>",
    "message": "<message>"
}
```
</details>

<details><summary><h3><b>Merchant </b></h3></summary>

### **`GET /v1/merchant`**

#### Request for retrieving merchant's data

- **URL**

```
GET https://{BASE_URL}/v1/merchant
```

- **Headers**

```javascript
{
    "x-access-token": "<access_token>", // access_token obtained in the OAuth2 flow unique for every merchant
    "Authorization": "Bearer <id_token>" // id_token obtained in the OAuth2 flow unique for every merchant
}
```

#### Response

- Positive response

```javascript
{
    merchant: [...],
    bank_accounts: [...],
    merchant_users: [...],
    locations: [
        {
          location_name: 'Test location',
          purs_location_id: 'qwertyabcd',
          ...
        },
        {
          location_name: 'Prod location',
          purs_location_id: 'abcdqwerty',
          ...
        }
    ]
}
```

- Negative responses

| `status code` | `message` |
| --- | --- |
| 401 | The bearer token is not valid. |
| 500 | Internal server error |

```javascript
{
    "status_code": "<status_code>",
    "message": "<message>"
}
```
</details>

<details><summary><h3><b>Transaction verification</b></h3></summary>

#### Request for retrieving transaction's status

- **URL**

```
GET https://{BASE_URL}/v1/transactions/{transactionId}/status
```

- **Headers**

```javascript
{
    "x-access-token": "<access_token>", // access_token obtained in the OAuth2 flow unique for every merchant
    "Authorization": "Bearer <id_token>" // id_token obtained in the OAuth2 flow unique for every merchant
}
```

#### Response

- Positive response

```javascript
{
    "transaction_id": "<transaction_id>",
    "status": "COMPLETED" | "PENDING" | "CANCELLED" | "REVERSED"
}
```

- Negative responses

| `status code` | `message` |
| --- | --- |
| 403 | This transaction belongs to a different merchant. |
| 401 | The bearer token is not valid. |
| 404 | Transaction could not be not found. |
| 500 | Internal server error |

```javascript
{
    "status_code": "<status_code>",
    "message": "<message>"
}
```
</details>

<details><summary><h3><b>Location's transactions</b></h3></summary>

#### Request for retrieving all location's transactions

- **URL**

```
GET https://{BASE_URL}/v1/transactions?location_id={locationId}&page_key={page_key}&limit={limit}
```
#### Note - `page_key` and `limit` are optional query params. For the first request there won't be a `page_key` and if `limit` is not passed then the default value is 25.

- **Headers**

```javascript
{
    "x-access-token": "<access_token>", // access_token obtained in the OAuth2 flow unique for every merchant
    "Authorization": "Bearer <id_token>" // id_token obtained in the OAuth2 flow unique for every merchant
}
```

#### Response

- Positive response

```javascript
{
  transactions: [{
      transaction_id: "<transaction_id>",
      status: "COMPLETED" | "PENDING" | "CANCELLED" | "REVERSED",
      ...
     }, {
	...
    }],
  next_page_key: "<page_key>"
}
```

- Negative responses

| `status code` | `message` |
| --- | --- |
| 403 | Forbidden |
| 401 | The bearer token is not valid. |
| 500 | Internal server error |

```javascript
{
    "status_code": "<status_code>",
    "message": "<message>"
}
```
</details>

<details><summary><h3><b>Cancel subscription</b></h3></summary>

#### Cancel user's subscription

- **URL**

```
DELETE https://{BASE_URL}/v1/transactions/subscription-cancel
```

- **Headers**

```javascript
{
    "x-access-token": "<access_token>", // access_token obtained in the OAuth2 flow unique for every merchant
    "Authorization": "Bearer <id_token>" // id_token obtained in the OAuth2 flow unique for every merchant
    "x-subscription-token": "<subscription_token>" // Token returned by Purs during user subscription process to confirm recurrent payment
}
```

#### Response

- Positive response

```javascript
{
    "created_at_datetime": "2024-05-05T11:00:00.000Z",
    "canceled_at_datetime": "2024-06-06T00:00:00.000Z"
}
```

- Negative responses

| `status code` | `message` |
| --- | --- |
| 401 | The bearer token is not valid. |
| 404 | Subscription not found or already canceled. |
| 500 | Internal server error |

```javascript
{
    "status_code": "<status_code>",
    "message": "<message>"
}
```
</details>

<details><summary><h3><b>Void transaction</b></h3></summary>

#### Void transaction created by mistake.

- **URL**

```
DELETE https://{BASE_URL}/v1//transactions/{transactionId}
```

- **Headers**

```javascript
{
    "x-access-token": "<access_token>", // access_token obtained in the OAuth2 flow unique for every merchant
    "Authorization": "Bearer <id_token>" // id_token obtained in the OAuth2 flow unique for every merchant
}
```

#### Response

- Positive response

```javascript
{
    "transaction_id": "<transaction_id>",
    "status": "REVERSED"
}
```

- Negative responses

| `status code` | `message` |
| --- | --- |
| 401 | The bearer token is not valid. |
| 404 | Transaction not found or already voided. |
| 403 | Transaction can not be voided. |
| 500 | Internal server error |

```javascript
{
    "status_code": "<status_code>",
    "message": "<message>"
}
```
</details>

---

## Upcoming APIs

...
