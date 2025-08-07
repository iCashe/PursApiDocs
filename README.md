# Purs Developer Documentation

## Introduction

- This document provides a comprehensive guide for onboarding your merchants and connecting their accounts with **Purs** and start accepting payments.
- Before a merchant can link their account with **Purs** via your platform, a few prerequisites must be completed.

## Prerequisites

### **CLIENT_ID** and **CLIENT_SECRET**

- Purs will provide you with a **CLIENT_ID** and **CLIENT_SECRET**.

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

- **Endpoint details under [OAuth APIs section](#oauth)**
- To connect a seller's Purs Merchant Account with their Cann-X seller account, Cann-X will need to have a "Connect with Purs" button (likely somew in the admin portal for your merchants).
- When clicked, this button should navigate to the `https://{OAUTH_URL}/oauth2/authorize` URL with the appropriate query parameters.
- The merchant will be prompted to enter their Purs Merchant Portal login credentials.
- Once they authenticate, they will be redirected to the **REDIRECT_URL** Cann-X provided Purs. An extra query parameter will be present when the seller is redirected — a query parameter called `code`.
- See the next section to understand what to do with the `code` that is provided by Purs as a query parameter attached to your **REDIRECT_URL**.

### Retrieve and Store Tokens

- Extract the value of this `code` query parameter and make a `POST` request to Purs to exchange this short-lived `code` for OAuth tokens.
- You will need the **CLIENT_ID** and **CLIENT_SECRET** which Purs has provided you.
- Make sure to make this request from your backend where the **CLIENT_ID** and **CLIENT_SECRET** are stored securely.

### Refresh Tokens

- Since the `access_token` and `id_token` expire, you should refresh them with `refresh_token` to make valid requests.

### Revoke Tokens

- This is to revoke the tokens for a particular merchant.

</details>

<details><summary><h1><b>Checkout Flow</b></h1></summary>

There are 2 steps in this process  in the sequence diagram below.

- 🟧 - Getting the Purs Checkout Widget URL.
- 🟩 - Calling the `PursCheckoutWidget` method with the URL received from the above step.

<img width="775" height="1081" alt="Subscription" src="https://github.com/user-attachments/assets/19bc047b-7dbd-4ac1-8b99-462e6dfac29a" />



### 🟧 Purs Checkout Widget URL

- Purs checkout widget is a way for Cann-X customers to make payments.
- **Endpoint details to get the Purs Checkout Widget `/v1/transactions` under [Escrow Payment APIs](#escrow-payment)**

    > **Note**: The above request should be made from your backend, not directly from your frontend. This approach ensures that the tokens and their corresponding merchant mappings, which are stored in your backend, remain secure. Your frontend should make an API call to your backend with the required parameters. Your backend will then handle the call to the Purs API (`/v1/transactions`) using the valid tokens stored in your system.

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
> ⚠️ **Important**: Both the parameters for `PursCheckoutWidget.init` i.e. `url` and `onPaymentComplete` are required.

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
            // other necessary parameters
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
            // other necessary parameters
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

> **Note**: The naming of functions in the above code sample is for illustration purpose only. You can change it accordingly. Just make sure the core logic remains same and the `PursCheckoutWidget.init` method receives the `url` and `onPaymentComplete` parameters.


### Location ID of merchant (Only for subscription and one-time payments)

- In the Purs system, each "Merchant" can have multiple "Locations" (typically representing a physical retail location).
- During the onboarding process, when a merchant creates an account on the Purs Merchant Portal, they are required to add at least one location. Additional locations can also be added later through the portal.
- To retrieve all locations associated with a particular merchant, use the `/v1/merchant` endpoint. This allows you to present the available locations (and other details) related to the merchant on your platform, enabling them to choose the location where they want to receive payments from your users.
- Once the merchant selects a location, you will use the corresponding `location_id` in the request body as outlined in the previous section.
- **Endpoint details to get the locations `/v1/merchant`under [Merchant API](#merchant-api)**

### Transaction Status

- This is an optional but recommended step where you can make an additional API call to Purs to get the transaction status for a particular transaction.
- The`transaction_id` received in the checkout URL [response](#post-v1transactions) will be used to retrieve the status of that transaction.
- **Endpoint details to get transaction status `/v1/transactions/{transactionId}/status` under [Common Transaction APIs](#transaction-apis)**

</details>

<details><summary><h1><b>Onboarding Flow</b></h1></summary>

<img width="599" height="615" alt="Onboarding" src="https://github.com/user-attachments/assets/4c89f654-a388-45b4-bd04-e9e0e61f18a6" />

### Purs Onboarding URL

- Purs onboarding widget is a way for Cann-X to onboard their entitites to the Purs system.
- Onboarding URL to connect a merchant to Purs is as follows:
```html
https://sandbox-users.purs.digital/onboarding${userType}?email=${DEVELOPER_EMAIL}
```
- Currently `userType` can be one of the following:
    - `seller`
    - `buyer`
    - `transporter`
- `DEVELOPER_EMAIL` is optional parameter passed so user doesn't have to again enter their email in Purs onboarding widget


### PursCheckoutWidget method for onboarding

- Below is code sample to integrate the Purs checkout widget onboading flow in your website.

**Step 1**

Add the Purs checkout CDN into your script tag

```html
<script src="https://purs-sandbox-cdn.s3.us-west-2.amazonaws.com/checkout/v1/index.min.js"></script>
```

**Step 2**

Add a "Onboarding with Purs" button on your page.

```html
<button id="onboarding-btn">
    Onboard with Purs
</button>
```

> 👍 **Recommended**: Add the Purs logo to this button. [Link](https://purs-sandbox-cdn.s3.us-west-2.amazonaws.com/checkout/v1/connect-with-purs.png)

**Step 3**

Implement the logic to call a function (`initiateOnboarding`) which initiates the onboarding flow on a button click.

```javascript
const button = document.getElementById('onboarding-btn');
button.addEventListener('click', initiateOnboarding);
```

**Step 4**

Implement the logic to call the `PursCheckoutWidget.init` method with `url` and `onOnboardingComplete` as parameters.

- the `url` is `https://sandbox-users.purs.digital/onboarding${userType}?email=${DEVELOPER_EMAIL}`
- the `onOnboardingComplete` expects a callback function (`updateOnboardingUI`) defined on your end.

```javascript
const initiateOnboarding = async () => {
	try {
	
		// upto you how to pass this URL,`userType` and optional `DEVELOPER_EMAIL`
		const onboardingUrl = `https://sandbox-users.purs.digital/onboarding/${userType}?email=${DEVELOPER_EMAIL}`;

		PursCheckoutWidget.init({
			url: onboardingUrl,
			onOnboardingComplete: (onboardingData) => {
				console.log('Onboarding completed!', onboardingData);

				// Show user ID if available
				const userId = onboardingData?.userId; // a userId returned for you to store it securely 
				if (userId) {
					console.log(`User ID: ${userId}`);
				}

				// Update Onboarding UI
				updateOnboardingUI(); // Update UI is a function that you can implement which is called when the onboarding is completed.
			}
		});
	} catch (error) {
		console.error('Onboarding initiation failed:', error);
	}
};
```
> ⚠️ **Important**: Both the parameters for `PursCheckoutWidget.init` i.e. `url` and `onOnboardingComplete` are required.

**Step 5**

Implement the logic for a callback function (`updateOnboardingUI`) to handle any UI changes after a successful payment.

```javascript
const updateOnboardingUI = () => {
    const button = document.getElementById('onboarding-btn');
    // Disable the button
    button.disabled = true;
    // make necessary UI changes according to your needs
};
```



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
    <button id="onboarding-btn">Onboard with Purs</button>

    <script src="https://purs-sandbox-cdn.s3.us-west-2.amazonaws.com/checkout/v1/index.min.js"></script>
    <script src="./index.js" type="module"></script>

</body>
</html>
```

**JavaScript**

```javascript
const updateOnboardingUI = () => {
    const button = document.getElementById('purs-checkout-button');
    // Disable the button
    button.disabled = true;
    // make necessary UI changes according to your needs


};

const initiateOnboarding = async () => {
	try {
	
		// upto you how to pass this URL,`userType` and optional `DEVELOPER_EMAIL`
		const onboardingUrl = `https://sandbox-users.purs.digital/onboarding/${userType}?email=${DEVELOPER_EMAIL}`;

		PursCheckoutWidget.init({
			url: onboardingUrl,
			onOnboardingComplete: (onboardingData) => {
				console.log('Onboarding completed!', onboardingData);

				// Show user ID if available
				const userId = onboardingData?.userId; // a userId returned for you to store it securely 
				if (userId) {
					console.log(`User ID: ${userId}`);
				}

				// Update Onboarding UI
				updateOnboardingUI(); // Update UI is a function that you can implement which is called when the onboarding is completed.
			}
		});
	} catch (error) {
		console.error('Onboarding initiation failed:', error);
	}
};

const button = document.getElementById('onboarding-btn');
button.addEventListener('click', initiateOnboarding); // call the initiateOnboarding function when the button is clicked
```

**Integration steps**

> **Note:** The naming of functions in the above code sample is for illustration purpose only. You can change it accordingly. Just make sure the core logic remains same and the `PursCheckoutWidget.init` method receives the `url` and `onOnboardingComplete` parameters.

</details>



## Demo
<table>
<tr>
<td width="50%">

### Checkout Flow
[![Demo](https://github.com/user-attachments/assets/adb28ec8-51bf-492b-8dab-4d9ec4231d84)](https://drive.google.com/file/d/1zMIE1NFHZO3gC3ZY_Y60Esy8aUI7mM2r/view?usp=sharing)

</td>
<td width="50%">

### Onboarding Flow
[![Demo](https://github.com/user-attachments/assets/160eae7b-f16a-47cd-b1d8-7ba46acb55d9)](https://drive.google.com/file/d/1wXR2-bxM--znOldrUc7F4phYu-my3S0M/view?usp=sharing)

</td>
</tr>
</table>

## API Endpoints

<details id="oauth">

<summary><strong>OAuth APIs</strong></summary><br>

---
* <strong>Connect to Purs using OAuth2 Authorization</strong>

    * URL

        ```javascript
        // Put this URI in the "Connect with Purs" button
        GET https://{OAUTH_URL}/oauth2/authorize?
                response_type=code& // leave as is. Ie. "code"
                client_id={CLIENT_ID}&
                redirect_uri={REDIRECT_URL}& // this must be URL-encoded
                state=abcdefg& // this is optional
                scope=openid+profile+email+phone+PURS_API/--- // add all required scopes
        ```

        | Field | Type | Description |
        | --- | --- | --- |
        | `response_type` | `string` | (Required) This parameter indicates the type of response desired from the OAuth2 server. It must be included in every authorization request. Currently only "code" type supported. |
        | `client_id` | `string` | (Required) The unique identifier assigned to your application by Purs. This ID is used to distinguish your application from others during the authentication process. |
        | `redirect_uri` | `string` | (Required) The URI where the user will be sent after authorization. This URI must be one of the pre-registered redirect URIs for your client ID. |
        | `state` | `string` | (Optional) The state parameter is an optional but highly recommended CSRF token to safeguard against Cross-Site Request Forgery attacks. It should be a unique, random string generated by your platform. The state value is passed a query param along with the REDIRECT_URL |
        | `scope` | `string` | (Required) The scope parameter requires a space-separated list of permissions, including standard scopes (`openid`, `email`, `phone`) and custom merchant-specific scopes |


        #### Custom Merchant Scopes
        | Scope | Description |
        |-------|-------------|
        | `PURS_API/MERCHANT_READ` | Enables your application to get merchant information |
        | `PURS_API/TRANSACTIONS_READ` | Allows read-only access to all transactions data |
        | `PURS_API/TRANSACTIONS_WRITE` | Grants permission to create / update transactions |


    * Success Response

        ```
        HTTP/1.1 302 Found
        Location: https://{REDIRECT_URL}?code=a1b2c3d4-5678-90ab-cdef-EXAMPLE11111&state=abcdefg
        ```

    * Error Responses

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
---
* <strong>Get new set of tokens</strong>

    * URL

        ```
        POST https://{OAUTH_URL}/oauth2/token
        ```

    * Headers

        ```javascript
        {
            "Content-Type": "application/x-www-form-urlencoded",
            "Authorization": "Basic <Base64Encode(<CLIENT_ID>:<CLIENT_SECRET>)>"
        }
        ```

    * Body (form-urlencoded, not JSON)

        ```
        grant_type=authorization_code& // leave as is. Ie. "authorization_code"
        client_id=<CLIENT_ID>&
        code=<code>& // from query parameter
        redirect_uri=<REDIRECT_URL> // this needs to URL encoded
        ```

    * Base64Encode example

        ```javascript
        const CLIENT_ID = 'dummy-client-id#1234'
        const CLIENT_SECRET = 'dummy-client-secret#4321'
        const authToken = `${CLIENT_ID}:${CLIENT_SECRET}`
        const base64Encode = btoa(authToken); // use this value in the Authorization header
        ```

    * Success Response

        ```javascript
        {
            "access_token": "eyJra1example",
            "id_token": "eyJra2example",
            "refresh_token": "eyJj3example",
            "expires_in": 86400 // expiry of access_token and id token, value in seconds
        }
        ```

    * Error Responses

        ```javascript
        {
            "error":"invalid_request|invalid_client|invalid_grant|unauthorized_client|unsupported_grant_type"
        }
        ```

    > Make sure to store these tokens securely in you system and note that every set of tokens are unique for a particular merchant.

    > Note: The `access_token` and `id_token` has expiry duration of 1 day (86400 secs) and the `refresh_token` has expiry duration of 10 years.

    > Note: Since the `access_token` and `id_token` expire, you can either run a daily cron job to refresh the tokens OR during a request call, check the expiry of tokens with the help of `expires_in` key and if the tokens are expired, refresh them before making the request call.

---
* <strong>Refresh tokens</strong>

    * URL

        ```
        POST https://{OAUTH_URL}/oauth2/token
        ```

    * Headers

        ```javascript
        {
            "Content-Type": "application/x-www-form-urlencoded",
            "Authorization": "Basic <Base64Encode(<CLIENT_ID>:<CLIENT_SECRET>)>"
        }
        ```

    * Body (form-urlencoded, not JSON)

        ```
        grant_type=refresh_token& // leave as is. Ie. "refresh_token"
        refresh_token=<refresh_token>
        ```

    * Success Response

        ```javascript
        {
            "access_token": "new1example",
            "id_token": "new2example",
            "expires_in": 86400 // expiry of access_token and id token, value in seconds
        }
        ```

    * Error responses

        ```javascript
        {
            "error":"invalid_request"
        }
        ```
---   
* <strong>Revoke tokens</strong>

    * URL

        ```
        POST https://{OAUTH_URL}/oauth2/revoke
        ```

    * Headers

        ```javascript
        {
            "Content-Type": "application/x-www-form-urlencoded",
            "Authorization": "Basic <Base64Encode(<CLIENT_ID>:<CLIENT_SECRET>)>"
        }
        ```

    * Body (form-urlencoded, not JSON)

        ```
        token=<refresh_token> // the refresh token of the merchant you want to revoke all the tokens.
        ```

    * Success Response

        ```
        A successful response contains an empty body
        ```

    - Negative response

        ```javascript
        {
            "error":"invalid_request|unsupported_token_type|invalid_client"
        }
        ```
---
</details>


<details id="merchant-api">

<summary><strong>Merchant API</strong></summary><br>

---
* <strong>Get merchant info</strong>

    * URL

        ```
        GET https://{BASE_URL}/v1/merchant
        ```

    * Headers

        ```javascript
        {
            "x-access-token": "<access_token>", // access_token obtained in the OAuth2 flow unique for every merchant
            "Authorization": "Bearer <id_token>" // id_token obtained in the OAuth2 flow unique for every merchant
        }
        ```

    * Success Response

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

    * Error Responses

        | `status code` | `message` |
        | --- | --- |
        | 401 | The bearer token is not valid. |
        | 500 | Internal server error |
---
</details>

<details id="one-time-payment">

<summary><strong>One-Time Transaction API</strong></summary><br>

---
* <strong>Create a one-time payment request</strong>
    * URL

        ```
        POST https://{BASE_URL}/v1/transactions
        ```

    * Headers

        ```javascript
        {
            "x-access-token": "<access_token>", // access_token obtained in the OAuth2 flow unique for every merchant
            "Authorization": "Bearer <id_token>" // id_token obtained in the OAuth2 flow unique for every merchant
        }
        ```

    * Body (JSON)

        ```javascript
        {
            "amount": amount, // The amount in cents to be immediately paid by the user (could be 0)  (Integer)
            "location_id": <purs_location_id>, // The ID of the merchant location where the subscription will be created (String)
        }
        ```

    * Success Response

        ```javascript
        {
            "url": "https://{CHECKOUT_URL}?tid=<transaction_id>",
            "transaction_id": "<transaction_id>",
        }
        ```

    * Error Responses

        | `status code` | `message` |
        | --- | --- |
        | 400 | The amount value is not an integer, less than 0, or greater than 100000. |
        | 401 | The bearer token is not valid. |
        | 404 | Location not found. |
        | 500 | Internal server error |
---
</details>

<details id="subscription-payment">

<summary><strong>Subscription Payment APIs</strong></summary><br>

---
* <strong>Create new subscription</strong>

    * URL

        ```
        POST https://{BASE_URL}/v1/transactions
        ```

    * Headers

        ```javascript
        {
            "x-access-token": "<access_token>", // access_token obtained in the OAuth2 flow unique for every merchant
            "Authorization": "Bearer <id_token>" // id_token obtained in the OAuth2 flow unique for every merchant
        }
        ```

    * Body (JSON)

        ```javascript
        {
            "amount": amount, // The amount in cents to be immediately paid by the user (could be 0)  (Integer)
            "location_id": <purs_location_id>, // The ID of the merchant location where the subscription will be created (String)
            "create_subscription": true // Subscription flag
        }
        ```

    * Success Response

        ```javascript
        {
            "url": "https://{CHECKOUT_URL}?tid=<transaction_id>",
            "transaction_id": "<transaction_id>",
        }
        ```

    * Error Responses

        | `status code` | `message` |
        | --- | --- |
        | 400 | The amount value is not an integer, less than 0, or greater than 100000. |
        | 401 | The bearer token is not valid. |
        | 404 | Location not found. |
        | 500 | Internal server error |

---
* <strong>Verify existing subscription</strong>

    * URL

        ```
        POST https://{BASE_URL}/v1/transactions/subscription-check
        ```

    * Headers

        ```javascript
        {
            "x-access-token": "<access_token>", // access_token obtained in the OAuth2 flow unique for every merchant
            "Authorization": "Bearer <id_token>" // id_token obtained in the OAuth2 flow unique for every merchant
            "x-subscription-token": "<subscription_token>" // Token returned by Purs during user subscription process to confirm recurrent payment
        }
        ```

    * Body (JSON)

        ```javascript
        {
            "amount": amount, // Optional. Verifies that the user has at least this amount in their bank account
        }
        ```

    * Success Response

        ```javascript
        {
            "created_at_datetime": "2024-05-05T11:00:00.000Z",
            "account_nickname": "User's account",
            "account_last_four": "1234",
            "amount_verified": true/false      // If the amount was passed in the request
        }
        ```

    * Error Responses

        | Status | Error Message |
        |--------|---------------|
        | `401` | The bearer token is not valid |
        | `404` | Subscription not found or canceled |
        | `404` | User does not have active bank account |
        | `500` | Internal server error |

---
* <strong>Record a recurring payment for an existing subscription</strong>

    * URL

        ```
        POST https://{BASE_URL}/v1/transactions/auto-approve
        ```

    * Headers

        ```javascript
        {
            "x-access-token": "<access_token>", // access_token obtained in the OAuth2 flow unique for every merchant
            "Authorization": "Bearer <id_token>" // id_token obtained in the OAuth2 flow unique for every merchant
            "x-subscription-token": "<subscription_token>" // Token returned by Purs during user subscription process to confirm recurrent payment
        }
        ```

    * Body (JSON)

        ```javascript
        {
            "amount": amount, // The amount in cents to be paid by the user (non-zero)  (Integer)
        }
        ```

    * Success Response

        ```javascript
        {
            "transaction_id": "<transaction_id>"
        }
        ```

    * Error Responses

        | `status code` | `message` |
        | --- | --- |
        | 400 | The amount value is not an integer, non positive, or greater than 100000. |
        | 401 | The bearer token is not valid. |
        | 404 | Location not found. |
        | 500 | Internal server error |

---
* <strong>Cancel an existing subscription</strong>

    * URL

        ```
        DELETE https://{BASE_URL}/v1/transactions/subscription-cancel
        ```

    * Headers

        ```javascript
        {
            "x-access-token": "<access_token>", // access_token obtained in the OAuth2 flow unique for every merchant
            "Authorization": "Bearer <id_token>" // id_token obtained in the OAuth2 flow unique for every merchant
            "x-subscription-token": "<subscription_token>" // Token returned by Purs during user subscription process to confirm recurrent payment
        }
        ```

    * Success Response

        ```javascript
        {
            "created_at_datetime": "2024-05-05T11:00:00.000Z",
            "canceled_at_datetime": "2024-06-06T00:00:00.000Z"
        }
        ```

    * Error Responses

        | `status code` | `message` |
        | --- | --- |
        | 401 | The bearer token is not valid. |
        | 404 | Subscription not found or already canceled. |
        | 500 | Internal server error |

---
</details>

<details id="escrow-payment">

<summary><strong>Escrow Payment APIs</strong></summary><br>

---
* <strong>Create new escrow transaction</strong>

    * URL

        ```
        POST https://{BASE_URL}/v1/transactions
        ```

    * Headers

        ```javascript
        {
            "x-access-token": "<access_token>", // access_token obtained in the OAuth2 flow unique for every merchant
            "Authorization": "Bearer <id_token>" // id_token obtained in the OAuth2 flow unique for every merchant
        }
        ```

    * Body (JSON)

        ```javascript
        {
            "amount": amount, // The amount in cents to be moved from buyer to escrow account  (Integer)
            "create_escrow": true, // Escrow flag
            "seller_id": <seller_id> // userId generated with the onboarding flow for external merchant payee
        }
        ```

    * Success Response

        ```javascript
        {
            "url": "https://{CHECKOUT_URL}?tid=<transaction_id>",
            "transaction_id": "<transaction_id>",
        }
        ```

    * Error Responses

        | `status code` | `message` |
        | --- | --- |
        | 400 | The amount value is not an integer, less than or equal to 0, or greater than 100000. |
        | 401 | The bearer token is not valid. |
        | 404 | Location not found. |
        | 500 | Internal server error |

---

* <strong>Top-up balance for an escrow transaction</strong>

    * URL

        ```
        POST https://{BASE_URL}/v1/transactions/escrow-toptup
        ```

    * Headers

        ```javascript
        {
            "x-access-token": "<access_token>", // access_token obtained in the OAuth2 flow unique for every merchant
            "Authorization": "Bearer <id_token>" // id_token obtained in the OAuth2 flow unique for every merchant
            "x-escrow-token": "<escrow_token>" // Token returned by Purs when user confirms the escrow transaction first time
        }
        ```

    * Body (JSON)

        ```javascript
        {
            "amount": amount, // The additional amount in cents to be moved from buyer to escrow account  (Integer)
        }
        ```

    * Success Response

        ```javascript
        {
            "transaction_id": "<transaction_id>",
            "status": "COMPLETED"
        }
        ```

    * Error Responses

        | Status | Error Message |
        |--------|---------------|
        | `401` | The bearer token is not valid |
        | `404` | Escrow transaction not found |
        | `404` | User does not have active bank account |
        | `500` | Internal server error |

---

* <strong>Release funds and settle escrow transaction</strong>

    * URL

        ```
        POST https://{BASE_URL}/v1/transactions/escrow-release
        ```

    * Headers

        ```javascript
        {
            "x-access-token": "<access_token>", // access_token obtained in the OAuth2 flow unique for every merchant
            "Authorization": "Bearer <id_token>" // id_token obtained in the OAuth2 flow unique for every merchant
            "x-escrow-token": "<escrow_token>" // Token returned by Purs when user confirms the escrow transaction first time
        }
        ```

    * Body (JSON)

        ```javascript
        {
            "auto_refund_excess": true|false, // if true, additional escrow balance will be returned to buyer, else balance must settle to 0
            "payments": [
                // provide a list of payor + payee+ amount records to be processed
                // default payor is escrow if not specified
                // buyer_id need not be specified for refund of remaining amount, only set auto_refund_excess to true
                // all ID values mentioned below are returned by Purs during onboarding of the respective entities
                {
                    // payment to seller
                    "payee_id": <seller_id>,
                    "amount": 500
                },
                {
                    // payment to transporter
                    "payee_id": <transporter_id>,
                    "amount": 100
                },
                {
                    // seller to transporter transfer
                    "payor_id": <seller_id>,
                    "payee_id": <transporter_id>,
                    "amount": 127
                }
            ]
        }
        ```

    * Success Response

        ```javascript
        {
            "message": "Success"
        }
        ```

    * Error Responses

        | Status | Error Message |
        |--------|---------------|
        | `401` | The bearer token is not valid |
        | `404` | Escrow transaction not found |
        | `404` | User does not have active bank account |
        | `500` | Internal server error |

---


* <strong>Check existing escrow transaction status</strong>

    * URL

        ```
        GET https://{BASE_URL}/v1/transactions/escrow-check
        ```

    * Headers

        ```javascript
        {
            "x-access-token": "<access_token>", // access_token obtained in the OAuth2 flow unique for every merchant
            "Authorization": "Bearer <id_token>" // id_token obtained in the OAuth2 flow unique for every merchant
            "x-escrow-token": "<escrow_token>" // Token returned by Purs when user confirms the escrow transaction first time
        }
        ```

    * Success Response

        ```javascript
        {
            "status": "HOLD|READY|CLOSED",
            "updated_at_datetime": "2025-08-07T18:41:42.525Z",
            "escrow_available": 0, // Balance already available in escrow account for settlement
            "escrow_pending": 0 // Balance amount in transit from buyer to escrow account for the ACH to-up done
        }
        ```

    * Error Responses

        | Status | Error Message |
        |--------|---------------|
        | `401` | The bearer token is not valid |
        | `404` | Escrow transaction not found |
        | `404` | User does not have active bank account |
        | `500` | Internal server error |

---

* <strong>Confirm an escrow transaction (update from HOLD to READY - only for non-production)</strong>

    * URL

        ```
        POST https://{BASE_URL}/v1/transactions/escrow-confirm
        ```

    * Headers

        ```javascript
        {
            "x-access-token": "<access_token>", // access_token obtained in the OAuth2 flow unique for every merchant
            "Authorization": "Bearer <id_token>" // id_token obtained in the OAuth2 flow unique for every merchant
            "x-escrow-token": "<escrow_token>" // Token returned by Purs when user confirms the escrow transaction first time
        }
        ```

    * Success Response

        ```javascript
        {
            "status": "READY",
            "escrow_available": 3000, // Balance already available in escrow account for settlement
            "escrow_pending": 0 // Balance amount in transit from buyer to escrow account for the ACH to-up done
        }
        ```

    * Error Responses

        | Status | Error Message |
        |--------|---------------|
        | `401` | The bearer token is not valid |
        | `404` | Escrow transaction not found |
        | `404` | User does not have active bank account |
        | `500` | Internal server error |

---

</details>

<details id="transaction-apis">

<summary><strong>Common Transaction APIs</strong></summary><br>

---
* <strong>Get all transactions for the merchant</strong>

    * URL

        ```
        GET https://{BASE_URL}/v1/transactions?page_key={page_key}&limit={limit}
        ```

        > Note - `page_key` and `limit` are optional query params. For the first request there won't be a `page_key` and if `limit` is not passed then the default value is 25.

    * Headers

        ```javascript
        {
            "x-access-token": "<access_token>", // access_token obtained in the OAuth2 flow unique for every merchant
            "Authorization": "Bearer <id_token>" // id_token obtained in the OAuth2 flow unique for every merchant
        }
        ```

    * Success Response

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

    * Error Responses

        | `status code` | `message` |
        | --- | --- |
        | 403 | Forbidden |
        | 401 | The bearer token is not valid. |
        | 500 | Internal server error |

---
* <strong>Get status of a single transaction</strong>

    * URL

        ```
        GET https://{BASE_URL}/v1/transactions/{transactionId}/status
        ```

    * Headers

        ```javascript
        {
            "x-access-token": "<access_token>", // access_token obtained in the OAuth2 flow unique for every merchant
            "Authorization": "Bearer <id_token>" // id_token obtained in the OAuth2 flow unique for every merchant
        }
        ```

    * Success Response

        ```javascript
        {
            "transaction_id": "<transaction_id>",
            "status": "COMPLETED" | "PENDING" | "CANCELLED" | "REVERSED"
        }
        ```

    * Error Responses

        | `status code` | `message` |
        | --- | --- |
        | 403 | This transaction belongs to a different merchant. |
        | 401 | The bearer token is not valid. |
        | 404 | Transaction could not be not found. |
        | 500 | Internal server error |

---
* <strong>Void (reverse) a transaction</strong>

    * URL

        ```
        DELETE https://{BASE_URL}/v1/transactions/{transactionId}
        ```

    * Headers

        ```javascript
        {
            "x-access-token": "<access_token>", // access_token obtained in the OAuth2 flow unique for every merchant
            "Authorization": "Bearer <id_token>" // id_token obtained in the OAuth2 flow unique for every merchant
        }
        ```

    * Success Response

        ```javascript
        {
            "transaction_id": "<transaction_id>",
            "status": "REVERSED"
        }
        ```

    * Error Responses

        | `status code` | `message` |
        | --- | --- |
        | 401 | The bearer token is not valid. |
        | 404 | Transaction not found or already voided. |
        | 403 | Transaction can not be voided. |
        | 500 | Internal server error |
---
</details>
