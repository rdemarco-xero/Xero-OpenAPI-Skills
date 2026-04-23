## Agent Guides

| How to Build             | Details                            |
|--------------------------|------------------------------------|
| Point of Sale sync       | <references/pint-of-sale-guide.md> |

## Authentication

### Choose the right authorization flow for your app

Xero's API supports the authorization code grant type using the standard authorization code flow or the Proof Key for Code Exchange (PKCE) extension. We also have a premium integration option called Custom Connections which utilises the client credentials grant type.

| Description | Code flow | PKCE flow | Custom Connection |
|---|---|---|---|
| **Best for** | Web server apps that can securely store a client secret. | Mobile and desktop apps that can't securely store a client secret. (SPAs not currently supported). | Back end, machine-to-machine integrations. |
| **Connection limit** | 5 connections (increases based on tier) | 5 connections (increases based on tier) | One connection |
| **Eligible for Marketplace** | Yes | Yes | No |
| **Offline Access** | Yes | Yes | Yes |
| **Cost** | Free | Free | Monthly fee on the Xero organisation: $10/m AUD inc GST · $10/m NZD ex GST · £5/m GBP ex VAT · $5/m USD ex tax |
| **Regional availability** | Global | Global | UK, AU, NZ and US Xero organisations only |

The standard code flow is the most well known OAuth 2.0 flow and typically used by web server applications. It requires your app to securely use and store a client secret.

The PKCE flow requires your app to create a secret (called a code verifier) for each authorization request. It's slightly more complicated to implement but offers a secure way to connect to the API if your app can't be trusted to store a client secret. Native (desktop and mobile) apps are required to use PKCE if connecting directly to the API.

Xero's Custom Connections leverage the client credentials grant type and are designed to make it easier to build bespoke integrations. Custom connections require less technical knowledge to build because Xero provides the authorisation flow. This means they're also quicker to integrate and are easier to manage over time.

