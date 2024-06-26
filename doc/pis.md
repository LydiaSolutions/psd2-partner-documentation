## USE LYDIA’S PSD2 PIS APIs

As part of the PSD2 implementation, Lydia provides a PIS API. Based on STET 1.6.2 it allows a TPP to initiate a new
transfer on behalf of a user.

## Table of content

 - [Prerequisites](#prerequisites)
 - [Creation flow](#creation-flow)
   - [Sequence](#sequence)
   - [Obtain a TPP token](#obtain-tpp-token-oauth---client-credentials)
   - [Create payment request](#create-payment-request)
   - [User approval](#user-approval)
   - [Obtain a PSU token](#obtain-psu-token-oauth---authorization-code)
   - [Confirm payment request](#confirm-payment-request)
 - [Follow up payment request](#follow-up-payment-request)
   - [Get payment request information](#get-payment-request-information)

## Prerequisites

To be able to exchange properly with the Lydia PIS API, you must have already contacted Lydia to be registered as a TPP.

You will then obtain your client ID and client secret to perform authorization.

## Creation flow

To initiate a new transfer the TPP must first identify itself and ask the user a confirmation.
This happens in several steps :

- [Obtain a TPP token](#obtain-tpp-token-oauth---client-credentials)
- [Create payment request](#create-payment-request)
- [User approval](#user-approval)
- [Obtain a PSU token](#obtain-psu-token-oauth---authorization-code)
- [Confirm payment request](#confirm-payment-request)

### Sequence

```mermaid
sequenceDiagram
    participant LydiaApp
    participant PSU
    participant TPP
    participant PIS

    PSU ->> TPP: Initiate the authorization flow
    TPP ->> PIS: POST /openbanking/oauth/v2/token
    PIS -->> TPP: HTTP-200 with TPP token
    
    TPP ->> PIS: POST /pisp/public/payment-requests
    PIS -->> TPP: HTTP-201 with PSU redirect URL

    TPP ->> PIS: GET {psu_redirection_url}
    PIS -->> LydiaApp: HTTP-302 redirect to Lydia app on consent page
    PSU ->> LydiaApp: Confirm the consent request
    LydiaApp ->> TPP: Redirect to TPP redirect URI with the authorization code as query parameter
    
    TPP ->> PIS: POST /openbanking/oauth/v2/token
    PIS -->> TPP: HTTP-200 with PSU token

    TPP ->> PIS: POST /pisp/public/payment-requests/{payment_request_id}/confirm
    PIS -->> TPP: HTTP 200 - transfer is sent
```

### Obtain TPP token (oauth - client credentials)

To retrieve what will be referred as a TPP token, the TPP should provide
its credentials and call the endpoint

**This route is using mTLS authentication. You must provide your QWAC certificate.** 

> POST https://openbanking.lydia-app.com:8082/openbanking/oauth/v2/token

Payload Example:

```json
{
  "client_id": "my_client_id",
  "client_secret": "my_secret_pass",
  "grant_type": "client_credentials",
  "scope": "pisp"
}
```

Response:

```json
{
  "access_token": "access.token.example...",
  "expires_in": 7200,
  "scope": "pisp",
  "token_type": "Bearer"
}
```

### Create payment request

**This route is using mTLS authentication. You must provide your QWAC certificate.**

> POST https://openbanking.lydia-app.com:8083/pisp/public/payment-requests

**Headers** :

| Name          | Example Value               | Description |
|---------------|-----------------------------|-------------|
| Authorization | Bearer access.token.example | TPP token   |

Payload Example:

```json
{
  "paymentInformationId": "7903209c-0aeb-4178-aa28-246b4d037aec",
  "creationDateTime": "2019-08-24T14:15:22Z",
  "numberOfTransactions": 1,
  "initiatingParty": {
    "name": "Mr Foo"
  },
  "paymentTypeInformation": {
    "serviceLevel": "SEPA",
    "categoryPurpose": "CASH",
    "localInstrument": "INST"
  },
  "debtor": {
    "name": "Mr Foo"
  },
  "debtorAccount": {
    "iban": "FR5817598000010000FOO000082"
  },
  "creditTransferTransaction": [
    {
      "paymentId": {
        "instructionId": "13ca1f9f-d1ea-442f-a91c-5b92057f63b7",
        "endToEndIdentification": "ZA90RH72"
      },
      "instructedAmount": {
        "amount": 13.37,
        "currency": "EUR"
      },
      "beneficiary": {
        "creditor": {
          "name": "Mr Bar"
        },
        "creditorAccount": {
          "iban": "FR8217598000010000BAR000026"
        }
      }
    }
  ],
  "supplementaryData": {
    "acceptedAuthenticationApproach": [
      "REDIRECT"
    ]
  }
}
```

Response:

```
{
    "appliedAuthenticationApproach": "REDIRECT",
    "_links": {
        "consentApproval": {
            "href": "https://openauth.lydia-app.com/lydia-login-url?client_id=my_client_id&consent_request_id=0a&payment_request_id=8db"
        }
    }
}
```

### User approval

To obtain the user approval, retrieve the link shared in the previous request (in `_links.consentApproval.href`).
You can parse from this URL the `payment_request_id`.

Add the `state` parameter, as well as your `redirect_uri` and make sure the mobile device of the user opens this link
with the parameter you've added.

It will redirect the user to the Lydia application where he will be able to approve the payment.

> GET https://openauth.lydia-app.com/lydia-login-url?client_id=...&redirect_uri=https://tpp.registered.redirect.uri&state=1

When the user accepts, it will redirect him to the previously specified `redirect_uri`, with an authorization code as
query parameter.

Ex:

> https://tpp.registered.redirect.uri?code=00temporary-authorization-code00

This `code` has a 30 seconds expiration time, and will be used to retrieve
a [PSU token](#obtain-psu-token-oauth---authorization-code)

### Obtain PSU token (oauth - authorization code)

**This route is using mTLS authentication. You must provide your QWAC certificate.**

> POST https://openbanking.lydia-app.com:8082/openbanking/oauth/v2/token

Payload Example:

```json
{
  "client_id": "my_client_id",
  "client_secret": "my_secret_pass",
  "grant_type": "authorization_code",
  "code": "00temporary-authorization-code00",
  "scope": "pisp",
  "redirect_uri": "https://tpp.registered.redirect.uri"
}
```

Response:

```json
{
  "access_token": "new-psu-token",
  "expires_in": 3600,
  "refresh_token": "new-refresh-token",
  "scope": "pisp",
  "token_type": "Bearer"
}
```

### Confirm payment request

**This route is using mTLS authentication. You must provide your QWAC certificate.**

> POST https://openbanking.lydia-app.com:8083/pisp/public/payment-requests/{payment_request_id}/confirm

Note: This should be the `payment_request_id` you've extracted at [Create payment request](#create-payment-request) step.

**Headers** :

| Name          | Example Value        | Description |
|---------------|----------------------|-------------|
| Authorization | Bearer new-psu-token | PSU token   |

Example response:
```json
{
    "paymentRequest": {
        "resourceId": "8db63567-1231-466e-b9b3-7d58c6109fab",
        "creationDateTime": "2019-08-24T14:15:22Z",
        "numberOfTransactions": 1,
        "paymentInformationStatus": "ACSP",
        "creditTransferTransaction": [
            {
                "paymentId": {
                  "instructionId": "13ca1f9f-d1ea-442f-a91c-5b92057f63b7",
                  "endToEndIdentification": "ZA90RH72"
                },
                "instructedAmount": {
                    "amount": 13.37,
                    "currency": "EUR"
                },
                "beneficiary": {
                    "creditor": {
                        "name": "Mr Bar"
                    },
                    "creditorAccount": {
                        "iban": "FR8217598000010000BAR000026"
                    }
                },
                "transactionStatus": "ACSP"
            }
        ]
    }
}
```

## Follow up payment request

Once the payment request is properly confirmed, the transfer is not settled yet.
To make sure that the payment ended up settled, you can use the API to retrieve the status of the request.

### Get payment request information

**This route is using mTLS authentication. You must provide your QWAC certificate.**

> GET https://openbanking.lydia-app.com:8083/pisp/public/payment-requests/{payment_request_id}

Note: This should be the `payment_request_id` you've extracted at [Create payment request](#create-payment-request) step.

**Headers** :

| Name          | Example Value               | Description |
|---------------|-----------------------------|-------------|
| Authorization | Bearer access.token.example | TPP token   |

Example response:
```json
{
  "paymentRequest": {
    "resourceId": "8db63567-1231-466e-b9b3-7d58c6109fab",
    "creationDateTime": "2019-08-24T14:15:22Z",
    "numberOfTransactions": 1,
    "paymentInformationStatus": "RJCT",
    "statusReasonInformation": "AM04",
    "creditTransferTransaction": {
      "paymentId": {
        "instructionId": "13ca1f9f-d1ea-442f-a91c-5b92057f63b7",
        "endToEndIdentification": "ZA90RH72"
      },
      "instructedAmount": {
        "amount": 13.37,
        "currency": "EUR"
      },
      "beneficiary": {
        "creditor": {
          "name": "Mr Bar"
        },
        "creditorAccount": {
          "iban": "FR8217598000010000BAR000026"
        }
      },
      "transactionStatus": "RJCT"
    }
  }
}
```
