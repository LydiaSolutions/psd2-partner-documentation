# Lydia Solutions PSD2 documentation for third party providers

This repository helps third party providers integrate with Lydia Solutions’ Open Banking API. It documents access to customer account information, funds availability checks and payment initiation, together with authorization requirements and monthly service monitoring reports.

## Onboarding and usage

Contact psd2@sumeria.eu to be enrolled and access the Open Banking API. Integration requires a registered provider identity, client credentials, registered redirect URIs for customer authorization, and mutual TLS for the protected interfaces.

After enrollment, start with the guide for your service below. Use the [authorization and API reference](./doc/assets/openapi.yaml) to prepare requests and interpret responses. It describes customer authorization through the Lydia app, token exchange and renewal, service scopes, request formats and error responses. Obtain the connection details required for your integration during onboarding.

## Service documentation

- [Account information services (AIS)](./doc/ais.md): retrieve customer identity, accounts, balances and transaction history.
- [Confirmation of funds (CBPII)](./doc/cbpii.md): check whether an account has sufficient funds for a specified amount.
- [Payment initiation services (PIS)](./doc/pis.md): create payment requests, direct customers to consent approval, retrieve request status and confirm pending requests.

The guides provide the service-specific entry points; the shared API reference describes their interfaces and authorization mechanisms.

## Service monitoring

[Monthly monitoring reports](./monitoring/) record daily availability of the Lydia app and third party provider interface, alongside response times for payment initiation, account information and funds confirmation. They also include corresponding payment and account balance response times in the Lydia app.
