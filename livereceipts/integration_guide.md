# Live Receipts Integration Guide

The live receipts product allows you to send receipts to customers via facebook messenger or email and store them at One Market. Once a user has opted in to receive receipts you can use One Market API's to manage their receipt communication preferences.

## Prerequisites

- The online version of this feature uses the [Facebook Messenger Checkbox Plug-in][4] to get the users consent. Familiarize yourself with it.
- A secure server where you run your application to integrate with the `/v1/retailer/orders` API.
- Choose the delivery channel(s) you would like your receipts to be delivered to. Available options are Facebook Messenger and Email.

## Integration Steps

1. [Onboarding and Obtaining Credentials](#onboarding-and-obtaining-credentials)
1. [Get an Access Token](#retrieving-access-tokens)
1. Online Integration
    - [Facebook messenger opt-in widget for your HTML checkout pages](#online-integration)
1. In-Store Integration
    - [Implicit opt-in using phone number or email on the `/v1/retailer/orders` API](#in-store-integration)
1. Creating an Order
    - [Choosing the delivery channel for your receipt](#receipt-delivery-options-for-live-receipts)
    - [Create an order via POST on the `/v1/retailer/orders` API when you want to send a receipt to a customer](#create-an-order)
1. [Let users view their receipt preferences and opt-out of receiving receipts at a later date](#retrieve-user-preferences-and-allow-the-user-to-modify-them)

## Onboarding and Obtaining Credentials

You will need to provide the following for us to onboard your account:

- Your merchant name. (As you want them to appear on receipts)
- A list of unique retailer identifiers (yours included). You will use these to uniquely identify retailers in your system and send them to us via a `Retailer-Ref` HTTP header in your requests.
- **Important:** If you are doing the [Online Integration](#online-integration) with the HTML Facebook widget also provide the domain you will be hosting your sandbox and production environments on. As well as facebook usernames (not email addresses) for internal users you will want to test with.

You will receive from us:

- A `client_id`, `client_secret` and a list of `scopes` for each environment you have access to.

**Important:** Protect the `client_id` and `client_secret` the same way you would protect any other credentials in your backend system.

## Retrieving Access Tokens

The live receipts feature uses a [standard OAuth 2.0 client_credentials grant][1] to provide your application access to our Retailer APIs.

Request below (**Note:** the -u parameter below for HTTP Basic Authentication of the `client_id` and `client_secret`):

```bash
curl -X POST \
-u "{{client_id}}:{{client_secret}}" \
-H "content-type: application/x-www-form-urlencoded" \
-d "grant_type=client_credentials" \
-d "scope=retailer.orders.write retailer.orders.read retailer.users.write retailer.users.read" \
https://api.sandbox.onemarketnetwork.com/v1/oauth2/token
```

Response:

```json
{
    "access_token": "m-MuvKra4jC1vCYaz8NAjMBegtAFrXiV2eTQ8Jos3Lk.fuRHYe5IPnVUW-RFZFK1kGtjRAylsmlUI1GbDUIasdk",
    "expires_in": 299,
    "scope": "retailer.orders.write retailer.orders.read retailer.users.write retailer.users.read",
    "token_type": "bearer"
}
```

The access token is valid for 300 seconds. You will pass it in an HTTP Authorization header with the `access_token` appended to the string `Bearer` separated by a space.

Example:

`Authorization: Bearer m-MuvKra4jC1vCYaz8NAjMBegtAFrXiV2eTQ8Jos3Lk.fuRHYe5IPnVUW-RFZFK1kGtjRAylsmlUI1GbDUIasdk`

Example request to get the status of an order:

```bash
curl -X GET \
-H "Authorization: Bearer m-MuvKra4jC1vCYaz8NAjMBegtAFrXiV2eTQ8Jos3Lk.fuRHYe5IPnVUW-RFZFK1kGtjRAylsmlUI1GbDUIasdk" \
-H "Retailer-Ref: {{retailer_ref}}" \
https://api.sandbox.onemarketnetwork.com/v1/retailer/orders/123456700000-236
```

Response:

```json
{
    "meta": {
        "deprecationInformation": {}
    },
    "data": {},
    "errors": {
        "resource": [
            "123456700000-236"
        ]
    }
}
```

### Access Token Considerations

Calling an API with an `access_token` after it has expired will result in an HTTP status code `401:(UNAUTHORIZED)`. At this point your application must repeat the client_credentials API call to get a new access token.

*Note:* Many languages and frameworks contain the necessary http clients to make this happen automatically.

## Online Integration

In this case the user chooses to opt-in to receive their receipts via Facebook Messenger on an HTML checkout page controlled by you. You will then use the [`/v1/retailer/orders`](#create-an-order) to send receipts to the customer after they have opted-in.

![alt text](images/fbm.png "The Facebook Messenger Checkbox Plugin")

### Integrate Facebook Messenger Widget for your HTML pages

1. Prerequisites of your page
    - The hosting web page must be delivered over `https`.
    - The domain where you host the widget must be whitelisted by the [Facebook messenger checkbox plug-in][4].
    - Users that have accepted the invitation to test the app on https://developer.facebook.com.
    - *Optionally:* The [Google Chrome Disable Content Security Plugin](https://chrome.google.com/webstore/detail/disable-content-security/ieelmcmcagommplceebfedjlakkhpden?hl=en). This is used so the Facebook checkbox widget can be rendered on your website incase you can not get your domain whitelisted quickly.

1. Instructions
    - Insert the following javascript into the `head` section of your html page:
        ```html
        <script class="wrstag" src="//content.sandbox.onemarketnetwork.com/wrsjstag/wrs-sdk.min.js"></script>`
        ```
    - Insert a facebook checkbox `div` into your html page:
        ```html
        <div id="wrs-todo-box">
            <div id="messenger-ckbox"></div>
        </div>
        ```
    - Call the initialization method:
        ```html
        <script>
            WF.init();
        </script>
        ```
    - Set the data you will use to initialize the facebook messenger bot.
        ```html
        <script>

        // In this example we do this client side. You can also generate this on your server.
        var orderDate = new Date().getTime();

        // In the case of a known user you should set this to false and set RETAILER_USER_REF to a unique id of the user in your system.
        // In the case of a guest user you should set this to true and set RETAILER_USER_REF to a unique id you can track.
        var isGuest = false;

        // The orderRef is a unique id you generate. You will use this to reference the order later if you want to send or update the receipt for this transaction.
        var orderRef = "<<UNIQUE ORDER REFERENCE YOU WILL STORE>>";

        // The retailerUserRef is a unique id you generate. You will use this later to send new and updated receipts to this customer and allow
        // them to manage their preferences.
        var retailerUserRef = "<<UNIQUE USER REF YOU WILL STORE>>";

        // The retailerRef code you received during onboarding. You will have one for yourself or multiple if you are acting on
        // behalf of other retailers. Choose the correct one for the retailer.
        var retailerRef = "<<YOUR RETAILER_REF FOR THIS RETAILER>>";

        var optInInfo = {
            ORDER_DATE: orderDate,
            RETAILER_REF: retailerRef,
            IS_GUEST: isGuest,
            ORDER_REF: orderRef,
            RETAILER_USER_REF: retailerUserRef,
            FEATURE: "RECEIPTS"
        }
        </script>
        ```
    - Once you have the information data prepared and the user has opted-in, you can send the user a request through facebook messenger.
        ```html
        <script>
            WF.confirmUserOptIn(optInInfo);
        </script>
        ```

## In-Store Integration

This feature exclusively uses the `/v1/retailer/orders` API to opt a customer into receiving receipts via email or Facebook Messenger.

### Phone Number

In the case where you use a phone number entered by a customer on the terminal. You [create an order](#create-an-order) with a phone number and Facebook optin setting and we automatically try to send them a receipt via Facebook Messenger. If the user has a Facebook Messenger account tied to their phone number they will receive the receipt automatically. If they do not have an account they will be sent a text message with directions to sign up and get their receipt.

- See the [Facebook Messenger via phone number](#facebook-messenger-via-phone-number) section of the document for the required parameters for the `/v1/retailer/orders` API.

### Email

In the case where you use an email address entered by a customer on the terminal. You [create an order](#create-an-order) with an email optin setting and provide the customers email address. We will send the receipt to the email provided.

- See the [Email](#receipt-via-email) section of this document for the required parameters for the `/v1/retailer/orders` API.

## Create an Order

API documentation in [Swagger format][3] is available.

1. Your backend will need to [retrieve an access token](#retrieving-access-tokens) in order to call the APIs.

1. Determine what parameters you need to enable the [channel you will use to deliver the receipt.](#receipt-delivery-options-for-live-receipts)

1. Ensure you fill out all required fields defined in `/v1/retailer/orders` [swagger documentation][3]. To view the required fields view the Models section of the request in the documentation.

    ![alt text](images/swagger_models.png "Swagger models")

1. Add an HTTP header named `Retailer-Ref` to your API call.

    ```bash
    curl -X POST \
    -H "Authorization: Bearer m-MuvKra4jC1vCYaz8NAjMBegtAFrXiV2eTQ8Jos3Lk.fuRHYe5IPnVUW-RFZFK1kGtjRAylsmlUI1GbDUIasdk" \
    -H "Retailer-Ref: {{retailer_ref}}" \
    https://api.sandbox.onemarketnetwork.com/v1/retailer/orders
    ```

1. Your backend service will [POST to the `/v1/retailer/orders`][2] API to submit orders.
    ```json
    {
        "retailerOrder": {
            "orderRef": "<<YOUR UNIQUE IDENTIFIER FOR THIS ORDER>>",
            "userIdentification": {
                "retailerUserRef": "<<THE RETAILER_USER_REF USED IN THE JAVASCRIPT CODE ABOVE>>",
                "isGuest": false
            }
        },
        "optins": [ {
            "feature": "RECEIPTS",
            "channel": "FACEBOOK_MESSENGER"
        } ],
        ...
    }
    ```

## Retrieve user preferences and allow the user to modify them

Typically you will create an interface where you will allow users to manage their receipt communication preferences.  In order to support managing users preferences you will have to use the following APIs to retrieve and update their preferences.

1. Your backend will need to [retrieve an access token](#retrieving-access-tokens) in order to call the APIs. 
1. You will need to add an HTTP header named `Retailer-Ref` to your API call.

    ```bash
    curl -X POST \
    -H "Authorization: Bearer m-MuvKra4jC1vCYaz8NAjMBegtAFrXiV2eTQ8Jos3Lk.fuRHYe5IPnVUW-RFZFK1kGtjRAylsmlUI1GbDUIasdk" \
    -H "Retailer-Ref: {{retailer_ref}}" \
    https://api.sandbox.onemarketnetwork.com/v1/retailer/orders
    ```

1. Your backend service will issue a GET on the [`/v1/retailer/users/{{retailer_user_ref}}/preferences`][5] API to retrieve all the current preferences of the user. The `retailer_user_ref` will be the identifier you used in the javascript example above. You can also issue a GET on specific features and channels. An example for Facebook Messenger receipts: [`/v1/retailer/users/{{retailer_user_ref}}/preferences/FACEBOOK_MESSENGER/RECEIPTS`][5]

1. Your backend service will issue a PUT on the [`/v1/retailer/users/{{retailer_user_ref}}/preferences/{{channel}}/{{feature}}`][5] API with a update the preferences of the user. The `retailer_user_ref` will be the identifier you used in the javascript example above.

    Example:
    ```json
    {
        "userPreference": {
            "preference": {
                "feature": "RECEIPTS",
                "channel": "FACEBOOK_MESSENGER"
            },
            "setting": false
        }
    }
    ```

[1]: https://oauth2.thephpleague.com/authorization-server/client-credentials-grant/
[2]: https://retailerswagger.onemarketnetwork.com/#!/Order/post_retailer_retailer_code_order
[3]: https://retailerswagger.onemarketnetwork.com/#/
[4]: https://developers.facebook.com/docs/messenger-platform/discovery/checkbox-plugin
[5]: https://retailerswagger.onemarketnetwork.com/#!/User/get_retailer_retailer_code_user_retailer_user_ref_preference

## Receipt Delivery Options for Live Receipts

You have the option of sending receipts to a customer via email or Facebook Messenger. You will use the `optin` parameter of the `/v1/retailer/orders` API to choose your delivery mechanism. [See the complete API documentation.][2]

You have three delivery options:

- [Email](#receipt-via-email)
- [Facebook Messenger via user identifier](#facebook-messenger-via-user-identifier)
- [Facebook Messenger via phone number](#facebook-messenger-via-phone-number)

### Receipt via Email

To send a receipt via email you must provide an `optin` dictionary and provide the customer email address.

| Required Parameter | Value      | Required
|     ---            |  ---       | ---
| `orderType`       | `ONLINE or IN_STORE` | Y
| `optins`            | `[ { "feature": "RECEIPTS", "channel": "EMAIL", "enabled": true } ]` | Y
| `customer.email` | The customer email address | Y

```json
{
    ...
    "optins": [ {
        "feature": "RECEIPTS",
        "channel": "FACEBOOK_MESSENGER",
        "enabled": true
    } ] ,
    "customer": {
        "email": "testuser@testdomain.com"
    }
    ...
}
```

### Facebook Messenger via User Identifier

Given that a user has successfully opted-in to Facebook Messenger through the online integration and you have stored your own `retailer_user_ref` for the user you must provide an `optin` dictionary and set the `user_identification.retailer_user_ref` to your `retailer_user_ref` value.

| Required Parameter | Value      | Required
|     ---            |  ---       | ---
| `orderType`       | `ONLINE or IN_STORE` | Y
| `optins`            | `[ { "feature": "RECEIPTS", "channel": "FACEBOOK_MESSENGER", "enabled": true } ]` | Y
| `user_identification.retailerUserRef`            | A value you generated and stored to reference this user in your system. | Y

```json
{
    ...
    "optins": [ {
        "feature": "RECEIPTS",
        "channel": "FACEBOOK_MESSENGER",
        "enabled": true
    } ],
    "userIdentification": {
        "retailerUserRef": "<<THE RETAILER_USER_REF USED IN THE JAVASCRIPT CODE ABOVE>>",
        ...
    }
}
```

### Facebook Messenger via Phone Number

To send a receipt by phone number you must provide an `optin` dictionary, set `order_type` to `IN_STORE`, and provide a customer phone number (including country and area code).

| Required Parameter | Value      | Required
|     ---            |  ---       | ---
| `orderType`       | `IN_STORE` | Y
| `optins`            | `[ { "feature": "RECEIPTS", "channel": "FACEBOOK_MESSENGER", "enabled": true } ]` | Y
| `customer.phoneNumbers` | a list with a single customer phone number + area code (ex: +14158675309) | Y

An example:

```json
{
    ...
    "orderType": "IN_STORE",
    "optins": [ {
        "feature": "RECEIPTS",
        "channel": "FACEBOOK_MESSENGER",
        "enabled": true
    } ],
    "customer": {
        "phoneNumbers": ["+14158675309"]
    }
    ...
}
```