# Allegro Manager — API Reference & Project Architecture

> Comprehensive reference for building a product/sales management platform on top of **Allegro REST API** with **Odoo ERP** integration.

---

## Table of Contents

1. [Allegro REST API Overview](#1-allegro-rest-api-overview)
2. [Allegro API Endpoint Reference](#2-allegro-api-endpoint-reference)
3. [Odoo Integration Reference](#3-odoo-integration-reference)
4. [Project Architecture](#4-project-architecture)

---

## 1. Allegro REST API Overview

### Base URLs

| Environment | API Base URL | Auth URL | App Registration |
|---|---|---|---|
| **Production** | `https://api.allegro.pl` | `https://allegro.pl/auth/oauth/` | `https://apps.developer.allegro.pl/` |
| **Sandbox** | `https://api.allegro.pl.allegrosandbox.pl` | `https://allegro.pl.allegrosandbox.pl/auth/oauth/` | `https://apps.developer.allegro.pl.allegrosandbox.pl/` |

Official documentation: [developer.allegro.pl/documentation](https://developer.allegro.pl/documentation)
OpenAPI spec: [developer.allegro.pl/swagger.yaml](https://developer.allegro.pl/swagger.yaml)
Postman collection: [Allegro REST API on Postman](https://www.postman.com/allegro-rest-api/allegro-rest-api/collection/cztgnvw/allegro-rest-api)
Status page: [api.allegrostatus.pl](https://api.allegrostatus.pl/)

### Authentication (OAuth2)

Allegro uses OAuth2. All API requests require a Bearer token in the `Authorization` header.

#### Authorization Code Flow (for web apps)

```
1. Redirect user to:
   GET https://allegro.pl/auth/oauth/authorize
     ?response_type=code
     &client_id={CLIENT_ID}
     &redirect_uri={REDIRECT_URI}

2. User logs in, grants access → redirected to your redirect_uri with ?code=XXX

3. Exchange code for tokens:
   POST https://allegro.pl/auth/oauth/token
     ?grant_type=authorization_code
     &code={CODE}
     &redirect_uri={REDIRECT_URI}
   Authorization: Basic base64(CLIENT_ID:CLIENT_SECRET)

4. Response:
   {
     "access_token": "...",
     "token_type": "bearer",
     "refresh_token": "...",
     "expires_in": 43199,
     "scope": "allegro:api:sale:offers:read ...",
     "jti": "..."
   }
```

#### Device Flow (for CLI / server apps without browser)

```
1. POST https://allegro.pl/auth/oauth/device
     ?client_id={CLIENT_ID}
   Authorization: Basic base64(CLIENT_ID:CLIENT_SECRET)

2. Response includes verification_uri_complete — user visits this URL to authorize.

3. Poll for token:
   POST https://allegro.pl/auth/oauth/token
     ?grant_type=urn:ietf:params:oauth:grant-type:device_code
     &device_code={DEVICE_CODE}
   Authorization: Basic base64(CLIENT_ID:CLIENT_SECRET)
```

#### Token Refresh

```
POST https://allegro.pl/auth/oauth/token
  ?grant_type=refresh_token
  &refresh_token={REFRESH_TOKEN}
Authorization: Basic base64(CLIENT_ID:CLIENT_SECRET)
```

Tokens expire after ~12 hours. Refresh tokens are long-lived but single-use — each refresh returns a new refresh token.

### Content Types & Versioning

All requests must include versioned Accept/Content-Type headers:

```
Accept: application/vnd.allegro.public.v1+json
Content-Type: application/vnd.allegro.public.v1+json
```

Beta endpoints use:
```
Accept: application/vnd.allegro.beta.v1+json
```

DELETE requests and OAuth endpoints are not versioned.

### Language

Send `Accept-Language` header with each request:
- `pl-PL` (Polish — default)
- `en-US` (English)
- Also supported: `cs-CZ`, `sk-SK`, `hu-HU`, `uk-UA`

### Rate Limits

| Limit Type | Value | Scope |
|---|---|---|
| **Global** | 9,000 requests/minute | Per Client ID |
| **Per-endpoint** | Varies (documented per resource) | Per Client ID |
| **Leaky bucket** | Varies | Per User ID |

Exceeding limits returns `429 Too Many Requests` and blocks the Client ID for 1 minute.

### Pagination

Two styles used across the API:

**Offset-based:**
```
GET /sale/offers?limit=100&offset=0
GET /sale/offers?limit=100&offset=100
```

**Cursor-based:**
```
GET /sale/products?page.id=MjAyNC0wOS0xMFQwNjo0OTowMi40NTBa
```

### Error Handling

**Standard error format (4xx/5xx):**
```json
{
  "errors": [
    {
      "code": "NotAcceptableException",
      "message": "An error has occurred",
      "details": null,
      "path": "category.id",
      "userMessage": "Request contains invalid data.",
      "metadata": {}
    }
  ]
}
```

**OAuth error format (401):**
```json
{
  "error": "unauthorized",
  "error_description": "Full authentication is required to access this resource"
}
```

Every response includes a `Trace-Id` header for debugging/support tickets.

### Date/Time & Currency

- All dates in ISO 8601: `YYYY-MM-DDTHH:MM:SSZ` (UTC)
- Duration fields use ISO 8601 duration: `P3D` (3 days), `PT24H` (24 hours)
- Prices are always `{ "amount": "123.45", "currency": "PLN" }` (string amounts)
- Resource IDs are strings (usually UUID format)

---

## 2. Allegro API Endpoint Reference

All paths are relative to the base URL (`https://api.allegro.pl`).

### 2.1 Offers — User's Offer Information

| Method | Path | Description |
|---|---|---|
| `GET` | `/sale/offer-events` | Get events about the seller's offers |
| `GET` | `/sale/offers/{offerId}/smart` | Get Smart! classification report for an offer |
| `GET` | `/sale/offers` | Get seller's offers (with filters, sorting, pagination) |
| `GET` | `/sale/product-offers/{offerId}/parts` | Get selected data of a product-offer (fast, partial) |
| `GET` | `/sale/product-offers/{offerId}` | Get all data of a product-offer |

### 2.2 Offers — Offer Management

| Method | Path | Description |
|---|---|---|
| `POST` | `/sale/product-offers` | Create offer based on product |
| `PATCH` | `/sale/product-offers/{offerId}` | Edit an offer |
| `DELETE` | `/sale/product-offers/{offerId}` | Delete a draft offer |
| `GET` | `/sale/product-offers/{offerId}/operations/{operationId}` | Check processing status of POST/PATCH |
| `PUT` | `/sale/offer-price-change-commands/{commandId}` | Modify the Buy Now price |
| `PUT` | `/sale/offer-publish-commands/{commandId}` | Batch offer publish / unpublish |
| `GET` | `/sale/offer-publish-commands/{commandId}` | Publish command summary |
| `GET` | `/sale/offer-publish-commands/{commandId}/tasks` | Publish command detailed report |
| `GET` | `/sale/offers/{offerId}/promo-options` | Get promo options for an offer |
| `GET` | `/sale/offers/promo-options` | Get all available offer promotion packages |
| `GET` | `/sale/offers/promo-options-commands/{commandId}` | Get offer promotion packages |
| `POST` | `/sale/offers/promo-options-commands` | Modify offer promotion packages |
| `PUT` | `/sale/offer-promotion-commands/{commandId}` | Batch offer promotion package modification |
| `GET` | `/sale/offers/unfilled-parameters` | Get offers with missing parameters |

### 2.3 Offers — Translations

| Method | Path | Description |
|---|---|---|
| `GET` | `/sale/offers/{offerId}/translations` | Get offer translations |
| `PATCH` | `/sale/offers/{offerId}/translations/{language}` | Update offer translation |
| `DELETE` | `/sale/offers/{offerId}/translations/{language}` | Delete offer translation |

### 2.4 Offers — Batch Modification

| Method | Path | Description |
|---|---|---|
| `PUT` | `/sale/offer-modification-commands/{commandId}` | Batch offer modification |
| `GET` | `/sale/offer-modification-commands/{commandId}` | Modification command summary |
| `GET` | `/sale/offer-modification-commands/{commandId}/tasks` | Modification command detailed report |
| `PUT` | `/sale/offer-price-change-commands/{commandId}` | Batch offer price modification |
| `GET` | `/sale/offer-price-change-commands/{commandId}` | Change price command summary |
| `GET` | `/sale/offer-price-change-commands/{commandId}/tasks` | Change price command detailed report |
| `PUT` | `/sale/offer-quantity-change-commands/{commandId}` | Batch offer quantity modification |
| `GET` | `/sale/offer-quantity-change-commands/{commandId}` | Change quantity command summary |
| `GET` | `/sale/offer-quantity-change-commands/{commandId}/tasks` | Change quantity command detailed report |
| `POST` | `/sale/offer-price-automation-commands` | Batch automatic pricing rules modification |
| `GET` | `/sale/offer-price-automation-commands/{commandId}` | Automatic pricing command summary |
| `GET` | `/sale/offer-price-automation-commands/{commandId}/tasks` | Automatic pricing command detailed report |

### 2.5 Offers — Automatic Pricing

| Method | Path | Description |
|---|---|---|
| `GET` | `/sale/price-automation/rules` | Get automatic pricing rules |
| `POST` | `/sale/price-automation/rules` | Create automatic pricing rule |
| `GET` | `/sale/price-automation/rules/{ruleId}` | Get rule by ID |
| `PUT` | `/sale/price-automation/rules/{ruleId}` | Edit automatic pricing rule |
| `DELETE` | `/sale/price-automation/rules/{ruleId}` | Delete automatic pricing rule |
| `GET` | `/sale/price-automation/offers/{offerId}/rules` | Get rules assigned to an offer |

### 2.6 Offers — Variants

| Method | Path | Description |
|---|---|---|
| `GET` | `/sale/offer-variants` | Get the user's variant sets |
| `POST` | `/sale/offer-variants` | Create variant set |
| `GET` | `/sale/offer-variants/{setId}` | Get a variant set |
| `PUT` | `/sale/offer-variants/{setId}` | Update variant set |
| `DELETE` | `/sale/offer-variants/{setId}` | Delete a variant set |

### 2.7 Offers — Tags

| Method | Path | Description |
|---|---|---|
| `GET` | `/sale/offer-tags` | Get the user's tags |
| `POST` | `/sale/offer-tags` | Create a tag |
| `PUT` | `/sale/offer-tags/{tagId}` | Modify a tag |
| `DELETE` | `/sale/offer-tags/{tagId}` | Delete a tag |
| `GET` | `/sale/offers/{offerId}/tags` | Get tags assigned to an offer |
| `POST` | `/sale/offers/{offerId}/tags` | Assign tags to an offer |

### 2.8 Offers — Compatibility List

| Method | Path | Description |
|---|---|---|
| `GET` | `/sale/compatible-products/groups` | Get list of compatible product groups |
| `GET` | `/sale/compatible-products` | Get list of compatible products |
| `GET` | `/sale/compatible-products/suggested` | Get suggested compatibility list |
| `GET` | `/sale/compatibility-list/supported-categories` | Get categories where compatibility list is supported |

### 2.9 Offers — Tax Settings & Deposits

| Method | Path | Description |
|---|---|---|
| `GET` | `/sale/tax-settings` | Get all tax settings for category |
| `GET` | `/sale/deposit-types` | Get deposit types |

### 2.10 Offers — Rating

| Method | Path | Description |
|---|---|---|
| `GET` | `/sale/offers/{offerId}/rating` | Get offer rating |

### 2.11 Images & Attachments

| Method | Path | Description |
|---|---|---|
| `POST` | `/sale/images` | Upload an offer image |
| `POST` | `/sale/offer-attachments` | Create an offer attachment |
| `PUT` | `/sale/offer-attachments/{attachmentId}` | Upload an offer attachment |
| `GET` | `/sale/offer-attachments/{attachmentId}` | Get offer attachment details |

### 2.12 Products

| Method | Path | Description |
|---|---|---|
| `GET` | `/sale/products` | Search products |
| `GET` | `/sale/products/{productId}` | Get all data of a product |
| `POST` | `/sale/products` | Propose a product |
| `POST` | `/sale/products/{productId}/change-proposals` | Propose changes to a product |
| `GET` | `/sale/products/{productId}/change-proposals/{changeProposalId}` | Get product change proposal |
| `GET` | `/sale/products/parameters` | Get product parameters available in given category |

### 2.13 Categories & Parameters

| Method | Path | Description |
|---|---|---|
| `GET` | `/sale/categories` | Get IDs of Allegro categories |
| `GET` | `/sale/categories/{categoryId}` | Get a category by ID |
| `GET` | `/sale/categories/{categoryId}/parameters` | Get parameters supported by a category |
| `GET` | `/sale/categories/suggestions` | Get categories suggestions |
| `GET` | `/sale/categories/changes` | Get changes in categories |
| `GET` | `/sale/categories/{categoryId}/parameters-changes` | Get planned changes in category parameters |

### 2.14 Rebates & Promotions

| Method | Path | Description |
|---|---|---|
| `GET` | `/sale/loyalty/promotions` | Get the user's list of promotions |
| `POST` | `/sale/loyalty/promotions` | Create a new promotion |
| `GET` | `/sale/loyalty/promotions/{promotionId}` | Get promotion data by ID |
| `PUT` | `/sale/loyalty/promotions/{promotionId}` | Modify a promotion |
| `DELETE` | `/sale/loyalty/promotions/{promotionId}` | Deactivate a promotion |
| `GET` | `/sale/loyalty/turnover-discounts` | Get the list of turnover discounts |
| `PUT` | `/sale/loyalty/turnover-discounts/{marketplaceId}` | Create/modify turnover discount |
| `PUT` | `/sale/loyalty/turnover-discounts/{marketplaceId}/deactivate` | Deactivate turnover discount |

### 2.15 Offer Bundles

| Method | Path | Description |
|---|---|---|
| `GET` | `/sale/bundles` | List seller's bundles |
| `POST` | `/sale/bundles` | Create a new offer bundle |
| `GET` | `/sale/bundles/{bundleId}` | Get bundle by ID |
| `DELETE` | `/sale/bundles/{bundleId}` | Delete bundle |
| `PUT` | `/sale/bundles/{bundleId}/discount` | Update discount associated with bundle |

### 2.16 Badge Campaigns

| Method | Path | Description |
|---|---|---|
| `GET` | `/sale/badge-campaigns` | Get available badge campaigns |
| `GET` | `/sale/badges` | Get a list of badges |
| `POST` | `/sale/badges` | Apply for badge in selected offer |
| `GET` | `/sale/badges/{badgeApplicationId}` | Get badge application details |
| `GET` | `/sale/badge-applications` | Get a list of badge applications |
| `PATCH` | `/sale/badges/offers/{offerId}/campaigns/{campaignId}` | Update campaign badge for offer |
| `GET` | `/sale/badge-operations/{operationId}` | Get badge operation details |

### 2.17 Allegro Prices

| Method | Path | Description |
|---|---|---|
| `GET` | `/sale/allegro-prices-account-eligibility` | Get eligibility info for account |
| `GET` | `/sale/allegro-prices-account-participation` | Get account participation status |
| `PATCH` | `/sale/allegro-prices-account-participation` | Update account participation |
| `PUT` | `/sale/allegro-prices-account-consents` | Update consents for account |
| `GET` | `/sale/allegro-prices-offer-consents/{offerId}` | Get consents for an offer |
| `PUT` | `/sale/allegro-prices-offer-consents/{offerId}` | Update consents for an offer |
| `POST` | `/sale/allegro-prices-offers-submit-command` | Submit offers command |
| `GET` | `/sale/allegro-prices-offers-submit-command/{commandId}` | Get submit command status |
| `POST` | `/sale/allegro-prices-offers-exclude-command` | Exclude offers command |
| `GET` | `/sale/allegro-prices-offers-exclude-command/{commandId}` | Get exclude command status |
| `POST` | `/sale/allegro-prices-offers-status` | Query Allegro Prices offers status |

### 2.18 AlleDiscount

| Method | Path | Description |
|---|---|---|
| `GET` | `/sale/alle-discount/campaigns` | List AlleDiscount campaigns |
| `GET` | `/sale/alle-discount/eligible-offers` | List eligible offers |
| `GET` | `/sale/alle-discount/offer-participations` | List offer participations |
| `POST` | `/sale/alle-discount/offer-submit-commands` | Submit offer to campaign |
| `GET` | `/sale/alle-discount/offer-submit-commands/{commandId}` | Get submission status |
| `POST` | `/sale/alle-discount/offer-withdrawal-commands` | Withdraw offer from campaign |
| `GET` | `/sale/alle-discount/offer-withdrawal-commands/{commandId}` | Get withdrawal status |

### 2.19 Classifieds

| Method | Path | Description |
|---|---|---|
| `GET` | `/sale/offer-classifieds-packages/{offerId}` | Get classified packages assigned to offer |
| `PUT` | `/sale/offer-classifieds-packages/{offerId}` | Assign packages to a classified |
| `GET` | `/sale/classifieds-packages` | Get configurations of packages |
| `GET` | `/sale/classifieds-packages/{packageId}` | Get configuration of a package |
| `GET` | `/sale/offer-classifieds-statistics/{offerId}` | Get advertisement daily statistics |
| `GET` | `/sale/classifieds-statistics` | Get seller's advertisement daily statistics |

### 2.20 Pricing (Fees & Commissions)

| Method | Path | Description |
|---|---|---|
| `GET` | `/pricing/offer-quotes` | Get the user's current offer quotes |
| `POST` | `/pricing/offer-fee-preview` | Calculate fee and commission for an offer |

---

### 2.21 Orders — Order Management

| Method | Path | Description |
|---|---|---|
| `GET` | `/order/events` | Get order events |
| `GET` | `/order/event-stats` | Get order events statistics |
| `GET` | `/order/checkout-forms` | Get the user's orders |
| `GET` | `/order/checkout-forms/{checkoutFormId}` | Get an order's details |
| `PUT` | `/order/checkout-forms/{checkoutFormId}/fulfillment` | Set seller order status |
| `GET` | `/order/checkout-forms/{checkoutFormId}/shipments` | Get list of parcel tracking numbers |
| `POST` | `/order/checkout-forms/{checkoutFormId}/shipments` | Add a parcel tracking number |
| `GET` | `/order/carriers` | Get list of available shipping carriers |
| `GET` | `/order/carriers/{carrierId}/tracking` | Get carrier parcel tracking history |
| `GET` | `/order/checkout-forms/{checkoutFormId}/invoices` | Get order invoice details |
| `POST` | `/order/checkout-forms/{checkoutFormId}/invoices` | Post new invoice |
| `PUT` | `/order/checkout-forms/{checkoutFormId}/invoices/{invoiceId}/file` | Upload invoice file |
| `POST` | `/order/checkout-forms/{checkoutFormId}/invoices/url` | Upload URL to billing documents |
| `GET` | `/order/pickup-drop-off-points` | Get Allegro pickup drop off points |

### 2.22 Payments

| Method | Path | Description |
|---|---|---|
| `GET` | `/payments/payment-operations` | Payment operations history |
| `GET` | `/payments/refunds` | Get a list of refunded payments |
| `POST` | `/payments/refunds` | Initiate a refund of a payment |

### 2.23 Post Purchase Issues (Disputes & Claims)

| Method | Path | Description |
|---|---|---|
| `GET` | `/sale/disputes` | Get the user's post purchase issues |
| `GET` | `/sale/disputes/{disputeId}` | Get a single dispute or claim |
| `GET` | `/sale/disputes/{disputeId}/messages` | Get messages/state changes in an issue |
| `POST` | `/sale/disputes/{disputeId}/messages` | Add a message to an issue |
| `POST` | `/sale/disputes/{disputeId}/status` | Change status of a claim |
| `POST` | `/sale/dispute-attachments` | Create an attachment declaration |
| `PUT` | `/sale/dispute-attachments/{attachmentId}` | Upload an attachment |
| `GET` | `/sale/dispute-attachments/{attachmentId}` | Get an attachment |

### 2.24 Shipment Management

| Method | Path | Description |
|---|---|---|
| `GET` | `/shipment-management/delivery-services` | Get available delivery services |
| `POST` | `/shipment-management/shipments/create-commands` | Create new shipment |
| `GET` | `/shipment-management/shipments/create-commands/{commandId}` | Get shipment creation status |
| `GET` | `/shipment-management/shipments/{shipmentId}` | Get shipment details |
| `POST` | `/shipment-management/shipments/{shipmentId}/cancel-commands` | Cancel shipment |
| `GET` | `/shipment-management/shipments/{shipmentId}/cancel-commands/{commandId}` | Get cancellation status |
| `POST` | `/shipment-management/shipments/labels` | Get shipments labels |
| `POST` | `/shipment-management/shipments/protocol` | Get shipments protocol |
| `POST` | `/shipment-management/pickups/proposals` | Get pickup proposals |
| `POST` | `/shipment-management/pickups/create-commands` | Request shipments pickup |
| `GET` | `/shipment-management/pickups/create-commands/{commandId}` | Create pickup command status |
| `GET` | `/shipment-management/pickups/{pickupId}` | Get pickup details |

### 2.25 Customer Returns (BETA)

| Method | Path | Description |
|---|---|---|
| `GET` | `/order/customer-returns` | Get customer returns by query |
| `GET` | `/order/customer-returns/{customerReturnId}` | Get customer return by ID |
| `POST` | `/order/customer-returns/{customerReturnId}/rejection` | Reject customer return refund |

### 2.26 Commission Refunds

| Method | Path | Description |
|---|---|---|
| `GET` | `/order/refund-claims` | Get list of refund applications |
| `POST` | `/order/refund-claims` | Create a refund application |
| `GET` | `/order/refund-claims/{claimId}` | Get refund application details |
| `DELETE` | `/order/refund-claims/{claimId}` | Cancel a refund application |

---

### 2.27 Sale Settings — After-Sale Services

| Method | Path | Description |
|---|---|---|
| `GET` | `/after-sales-service-conditions/return-policies` | Get user's return policies |
| `POST` | `/after-sales-service-conditions/return-policies` | Create new return policy |
| `GET` | `/after-sales-service-conditions/return-policies/{returnPolicyId}` | Get user's return policy |
| `PUT` | `/after-sales-service-conditions/return-policies/{returnPolicyId}` | Change return policy |
| `DELETE` | `/after-sales-service-conditions/return-policies/{returnPolicyId}` | Delete return policy |
| `GET` | `/after-sales-service-conditions/implied-warranties` | Get user's implied warranties |
| `POST` | `/after-sales-service-conditions/implied-warranties` | Create implied warranty |
| `GET` | `/after-sales-service-conditions/implied-warranties/{impliedWarrantyId}` | Get implied warranty |
| `PUT` | `/after-sales-service-conditions/implied-warranties/{impliedWarrantyId}` | Change implied warranty |
| `GET` | `/after-sales-service-conditions/warranties` | Get user's warranties |
| `POST` | `/after-sales-service-conditions/warranties` | Create warranty |
| `GET` | `/after-sales-service-conditions/warranties/{warrantyId}` | Get warranty |
| `PUT` | `/after-sales-service-conditions/warranties/{warrantyId}` | Change warranty |
| `POST` | `/after-sales-service-conditions/warranties/{warrantyId}/attachments` | Create warranty attachment metadata |
| `PUT` | `/after-sales-service-conditions/warranties/{warrantyId}/attachments/{attachmentId}` | Upload warranty attachment |

### 2.28 Delivery Settings

| Method | Path | Description |
|---|---|---|
| `GET` | `/sale/delivery-methods` | Get list of delivery methods |
| `GET` | `/sale/delivery-settings` | Get user's delivery settings |
| `PUT` | `/sale/delivery-settings` | Modify delivery settings |
| `GET` | `/sale/shipping-rates` | Get user's shipping rates |
| `POST` | `/sale/shipping-rates` | Create a new shipping rates set |
| `GET` | `/sale/shipping-rates/{shippingRatesSetId}` | Get shipping rates set details |
| `PUT` | `/sale/shipping-rates/{shippingRatesSetId}` | Edit shipping rates set |

### 2.29 Additional Services

| Method | Path | Description |
|---|---|---|
| `GET` | `/sale/offer-additional-services/categories` | Get definitions by categories |
| `GET` | `/sale/offer-additional-services/groups` | Get user's additional services groups |
| `POST` | `/sale/offer-additional-services/groups` | Create additional services group |
| `GET` | `/sale/offer-additional-services/groups/{groupId}` | Get group details |
| `PUT` | `/sale/offer-additional-services/groups/{groupId}` | Modify group |
| `GET` | `/sale/offer-additional-services/groups/{groupId}/translations` | Get translations for group |
| `PATCH` | `/sale/offer-additional-services/groups/{groupId}/translations/{language}` | Create/update translation |
| `DELETE` | `/sale/offer-additional-services/groups/{groupId}/translations/{language}` | Delete translation |

### 2.30 Size Tables

| Method | Path | Description |
|---|---|---|
| `GET` | `/sale/size-tables` | Get user's size tables |
| `POST` | `/sale/size-tables` | Create a size table |
| `GET` | `/sale/size-tables/{tableId}` | Get a size table |
| `PUT` | `/sale/size-tables/{tableId}` | Update a size table |
| `GET` | `/sale/size-tables-templates` | Get size table templates |

### 2.31 Points of Service

| Method | Path | Description |
|---|---|---|
| `GET` | `/points-of-service` | Get user's points of service |
| `POST` | `/points-of-service` | Create a point of service |
| `GET` | `/points-of-service/{pointOfServiceId}` | Get point of service details |
| `PUT` | `/points-of-service/{pointOfServiceId}` | Modify point of service |
| `DELETE` | `/points-of-service/{pointOfServiceId}` | Delete point of service |

### 2.32 Contacts

| Method | Path | Description |
|---|---|---|
| `GET` | `/sale/offer-contacts` | Get user's contacts |
| `POST` | `/sale/offer-contacts` | Create a new contact |
| `GET` | `/sale/offer-contacts/{contactId}` | Get contact details |
| `PUT` | `/sale/offer-contacts/{contactId}` | Modify contact details |

### 2.33 Responsible Persons & Producers

| Method | Path | Description |
|---|---|---|
| `GET` | `/sale/responsible-persons` | Get list of responsible persons |
| `POST` | `/sale/responsible-persons` | Create responsible person |
| `PUT` | `/sale/responsible-persons/{personId}` | Update responsible person |
| `GET` | `/sale/responsible-producers` | Get list of responsible producers |
| `POST` | `/sale/responsible-producers` | Create responsible producer |
| `GET` | `/sale/responsible-producers/{producerId}` | Get responsible producer |
| `PUT` | `/sale/responsible-producers/{producerId}` | Update responsible producer |

---

### 2.34 One Fulfillment — Advance Ship Notices

| Method | Path | Description |
|---|---|---|
| `GET` | `/fulfillment/advance-ship-notices` | Get list of ASNs |
| `POST` | `/fulfillment/advance-ship-notices` | Create an ASN |
| `GET` | `/fulfillment/advance-ship-notices/{asnId}` | Get single ASN |
| `PUT` | `/fulfillment/advance-ship-notices/{asnId}` | Update ASN |
| `DELETE` | `/fulfillment/advance-ship-notices/{asnId}` | Delete ASN |
| `PUT` | `/fulfillment/advance-ship-notices/{asnId}/cancel` | Cancel ASN |
| `PUT` | `/fulfillment/advance-ship-notices/{asnId}/submit` | Submit the ASN |
| `GET` | `/fulfillment/advance-ship-notices/{asnId}/submit-status` | Get submit status |
| `PUT` | `/fulfillment/advance-ship-notices/{asnId}/submitted` | Update submitted ASN |
| `GET` | `/fulfillment/advance-ship-notices/{asnId}/receiving-state` | Check receiving state |
| `GET` | `/fulfillment/advance-ship-notices/{asnId}/labels` | Get labels for ASN |

### 2.35 One Fulfillment — Stock, Parcels, Products, Removal

| Method | Path | Description |
|---|---|---|
| `GET` | `/fulfillment/stock` | Get available stock |
| `GET` | `/fulfillment/parcels` | Get list of shipped parcels |
| `GET` | `/fulfillment/products` | Get list of available products |
| `GET` | `/fulfillment/removal-preferences` | Get current removal preference |
| `PUT` | `/fulfillment/removal-preferences` | Create new removal preference |

### 2.36 Tax Identification Number

| Method | Path | Description |
|---|---|---|
| `GET` | `/tax-identification-numbers` | Get tax identification number |
| `POST` | `/tax-identification-numbers` | Add tax identification number |
| `PUT` | `/tax-identification-numbers/{taxIdentificationNumberId}` | Update tax identification number |

---

### 2.37 User Information

| Method | Path | Description |
|---|---|---|
| `GET` | `/me` | Get basic information about user |
| `GET` | `/me/sales-quality` | Get sales quality |
| `GET` | `/me/smart-seller-classification-report` | Get Smart! seller classification report |
| `GET` | `/me/additional-emails` | Get user's additional emails |
| `POST` | `/me/additional-emails` | Add new additional email |
| `GET` | `/me/additional-emails/{emailId}` | Get additional email info |
| `DELETE` | `/me/additional-emails/{emailId}` | Delete additional email |
| `GET` | `/users/{userId}/ratings` | Get user's ratings |
| `GET` | `/users/{userId}/ratings/{ratingId}` | Get rating by ID |
| `PUT` | `/users/{userId}/ratings/{ratingId}/answer` | Answer for rating |
| `PUT` | `/users/{userId}/ratings/{ratingId}/removal` | Request removal of rating |
| `GET` | `/users/{userId}/ratings-summary` | Get any user's ratings summary |

### 2.38 Information About Marketplaces

| Method | Path | Description |
|---|---|---|
| `GET` | `/marketplaces` | Get details for all marketplaces |

### 2.39 Message Center

| Method | Path | Description |
|---|---|---|
| `GET` | `/messaging/threads` | List user threads |
| `GET` | `/messaging/threads/{threadId}` | Get user thread |
| `PUT` | `/messaging/threads/{threadId}/read` | Mark thread as read |
| `GET` | `/messaging/threads/{threadId}/messages` | List messages in thread |
| `POST` | `/messaging/threads/{threadId}/messages` | Write a new message in thread |
| `POST` | `/messaging/messages` | Write a new message (new thread) |
| `GET` | `/messaging/messages/{messageId}` | Get single message |
| `DELETE` | `/messaging/messages/{messageId}` | Delete single message |
| `POST` | `/messaging/message-attachments` | Add attachment declaration |
| `PUT` | `/messaging/message-attachments/{attachmentId}` | Upload attachment binary data |
| `GET` | `/messaging/message-attachments/{attachmentId}` | Download attachment |

### 2.40 Billing

| Method | Path | Description |
|---|---|---|
| `GET` | `/billing/billing-entries` | Get list of billing entries |
| `GET` | `/billing/billing-types` | Get list of billing types |

### 2.41 Auctions & Bidding

| Method | Path | Description |
|---|---|---|
| `GET` | `/bidding/offers/{offerId}/bid` | Get current user's bid information |
| `PUT` | `/bidding/offers/{offerId}/bid` | Place a bid in an auction |

### 2.42 Charity & Affiliate

| Method | Path | Description |
|---|---|---|
| `GET` | `/charity/fundraising-campaigns` | Search fundraising campaigns |
| `GET` | `/sale/affiliate/conversions` | [BETA] List CPS conversions |

---

## 3. Odoo Integration Reference

### 3.1 Odoo External API Options

Odoo provides three external API protocols. The choice depends on which Odoo version you're running.

| Protocol | Endpoint | Odoo Versions | Status |
|---|---|---|---|
| **XML-RPC** | `/xmlrpc/2/` | 8.0 – 19.0 | Deprecated, removed in Odoo 20 (fall 2026) |
| **JSON-RPC** | `/jsonrpc` | 8.0 – 19.0 | Deprecated, removed in Odoo 20 (fall 2026) |
| **JSON-2 API** | `/json/2/` | 19.0+ | Current recommended API |

> **Requirement:** API access requires Odoo **Custom pricing plan** (not Free or Standard).

### 3.2 Authentication

**JSON-2 API (Odoo 19+):**
```
Authorization: Bearer {API_KEY}
Content-Type: application/json
```

**XML-RPC (legacy):**
```python
# Authenticate
common = xmlrpc.client.ServerProxy('{odoo_url}/xmlrpc/2/common')
uid = common.authenticate(db, username, password, {})

# Call methods
models = xmlrpc.client.ServerProxy('{odoo_url}/xmlrpc/2/object')
models.execute_kw(db, uid, password, 'res.partner', 'search', [[['is_company', '=', True]]])
```

API keys can be generated in Odoo: **User Preferences → Account Security → API Keys**.

### 3.3 JSON-2 API Request Format (Odoo 19+)

```
POST /json/2/{model}/{method}
Authorization: Bearer {API_KEY}
Content-Type: application/json

{
  "params": {
    "args": [...],
    "kwargs": {...}
  }
}
```

### 3.4 Key Odoo Models for Integration

| Odoo Model | Purpose | Sync Direction |
|---|---|---|
| `product.template` | Master product definitions | Odoo → Allegro |
| `product.product` | Product variants (with attributes) | Odoo → Allegro |
| `sale.order` | Sale orders | Allegro → Odoo |
| `sale.order.line` | Order line items | Allegro → Odoo |
| `stock.quant` | Warehouse stock levels | Odoo → Allegro |
| `stock.picking` | Delivery/shipping orders | Bidirectional |
| `account.move` | Invoices & billing entries | Allegro → Odoo |
| `account.move.line` | Invoice line items | Allegro → Odoo |
| `res.partner` | Customers / contacts | Allegro → Odoo |
| `purchase.order` | Purchase orders (restock) | Internal trigger |

### 3.5 Common Odoo API Operations

```
# Search records
POST /json/2/product.product/search_read
{ "params": { "kwargs": { "domain": [["type", "=", "product"]], "fields": ["name", "default_code", "qty_available"], "limit": 100 } } }

# Create record
POST /json/2/sale.order/create
{ "params": { "args": [{ "partner_id": 42, "order_line": [[0, 0, { "product_id": 7, "product_uom_qty": 2 }]] }] } }

# Update record
POST /json/2/product.product/write
{ "params": { "args": [[product_id], { "qty_available": 50 }] } }
```

---

## 4. Project Architecture

### 4.1 High-Level Architecture

```
┌─────────────────┐       ┌──────────────────────────┐       ┌─────────────────┐
│                 │       │    Allegro Manager API    │       │                 │
│   Allegro API   │◄─────►│       (NestJS)            │◄─────►│   Odoo ERP      │
│   api.allegro.pl│       │                          │       │   (JSON-2 API)  │
│                 │       │  ┌─────────┐  ┌────────┐ │       │                 │
└─────────────────┘       │  │ Prisma  │  │ Redis  │ │       └─────────────────┘
                          │  │ (ORM)   │  │ (Queue)│ │
                          │  └────┬────┘  └────────┘ │
                          │       │                  │
                          └───────┼──────────────────┘
                                  │
                          ┌───────┴────────┐
                          │  PostgreSQL    │
                          │  (Data Store)  │
                          └───────┬────────┘
                                  │
                          ┌───────┴────────┐
                          │  Next.js       │
                          │  Dashboard     │
                          │  (Frontend)    │
                          └────────────────┘
```

### 4.2 Monorepo Structure (Turborepo)

```
allegro-manager/
├── apps/
│   ├── api/                          # NestJS backend
│   │   ├── src/
│   │   │   ├── modules/
│   │   │   │   ├── auth/             # Allegro OAuth2 + JWT session
│   │   │   │   ├── allegro/          # Allegro API client wrapper
│   │   │   │   ├── offers/           # Offer CRUD, batch ops, pricing
│   │   │   │   ├── orders/           # Order sync, fulfillment
│   │   │   │   ├── products/         # Product catalog, categories
│   │   │   │   ├── odoo/             # Odoo API client (JSON-2 / XML-RPC)
│   │   │   │   ├── sync/             # Sync engine (queues, jobs, scheduling)
│   │   │   │   ├── billing/          # Allegro fees, Odoo accounting
│   │   │   │   ├── shipping/         # Shipment mgmt, tracking
│   │   │   │   ├── webhooks/         # Allegro event handlers
│   │   │   │   └── health/           # Health check endpoint
│   │   │   ├── common/
│   │   │   │   ├── decorators/
│   │   │   │   ├── guards/
│   │   │   │   ├── interceptors/
│   │   │   │   ├── filters/
│   │   │   │   └── pipes/
│   │   │   ├── prisma/
│   │   │   │   ├── schema.prisma
│   │   │   │   └── migrations/
│   │   │   ├── app.module.ts
│   │   │   └── main.ts
│   │   ├── test/
│   │   └── package.json
│   │
│   └── web/                          # Next.js frontend
│       ├── src/
│       │   ├── app/                  # App Router pages
│       │   │   ├── (auth)/           # Login, OAuth callback
│       │   │   ├── dashboard/        # Main dashboard
│       │   │   ├── offers/           # Offer management UI
│       │   │   ├── orders/           # Order management UI
│       │   │   ├── products/         # Product catalog UI
│       │   │   ├── billing/          # Fees & financial reports
│       │   │   ├── settings/         # Account, Odoo connection
│       │   │   └── layout.tsx
│       │   ├── components/
│       │   │   ├── ui/               # shadcn/ui components
│       │   │   └── shared/           # App-specific shared components
│       │   └── lib/
│       │       ├── api-client.ts     # Typed API client (NestJS backend)
│       │       └── utils.ts
│       └── package.json
│
├── packages/
│   ├── shared/                       # Shared types, constants, utils
│   │   ├── src/
│   │   │   ├── types/
│   │   │   │   ├── allegro.ts        # Allegro API response types
│   │   │   │   ├── odoo.ts           # Odoo model types
│   │   │   │   └── index.ts
│   │   │   └── constants/
│   │   │       ├── allegro-endpoints.ts
│   │   │       └── index.ts
│   │   └── package.json
│   │
│   ├── eslint-config/                # Shared ESLint config
│   └── tsconfig/                     # Shared TypeScript configs
│
├── docker-compose.yml                # PostgreSQL + Redis
├── turbo.json
├── package.json
├── pnpm-workspace.yaml
├── .env.example
└── README.md
```

### 4.3 NestJS Module Breakdown

| Module | Responsibility | Key Dependencies |
|---|---|---|
| **AuthModule** | Allegro OAuth2 flow, token storage/refresh, JWT session for dashboard | `passport`, `@nestjs/jwt` |
| **AllegroModule** | Low-level HTTP client for Allegro API, auto token injection, rate limit handling | `axios` or `got` |
| **OffersModule** | CRUD offers, batch price/quantity changes, automatic pricing, promotions | AllegroModule |
| **OrdersModule** | Fetch & process orders, fulfillment status, invoice uploads | AllegroModule |
| **ProductsModule** | Product catalog, category tree, parameter lookup, product matching | AllegroModule |
| **OdooModule** | Odoo JSON-2/XML-RPC client, model CRUD operations | `xmlrpc` (for legacy) |
| **SyncModule** | Scheduled jobs, queue-based sync engine, conflict resolution | `@nestjs/bull`, `@nestjs/schedule` |
| **BillingModule** | Import billing entries from Allegro, push to Odoo accounting | AllegroModule, OdooModule |
| **ShippingModule** | Shipment creation, label generation, tracking sync | AllegroModule |
| **WebhooksModule** | Receive Allegro order/offer events, dispatch to relevant modules | — |
| **HealthModule** | Liveness/readiness checks | `@nestjs/terminus` |

### 4.4 Data Models (Prisma Schema — Overview)

```prisma
// Allegro seller account + OAuth tokens
model AllegroAccount {
  id            String   @id @default(cuid())
  name          String
  allegroUserId String   @unique
  clientId      String
  clientSecret  String
  accessToken   String
  refreshToken  String
  tokenExpiresAt DateTime
  sandboxMode   Boolean  @default(false)
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt

  offers        Offer[]
  orders        Order[]
  syncJobs      SyncJob[]
}

// Internal product representation (master catalog)
model Product {
  id              String   @id @default(cuid())
  name            String
  sku             String   @unique
  ean             String?
  description     String?
  categoryId      String?
  price           Decimal
  currency        String   @default("PLN")
  stockQuantity   Int      @default(0)
  odooProductId   Int?     // FK to Odoo product.product
  odooTemplateId  Int?     // FK to Odoo product.template
  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt

  offers          Offer[]
  orderItems      OrderItem[]
}

// Allegro offer linked to internal product
model Offer {
  id              String   @id @default(cuid())
  allegroOfferId  String   @unique
  productId       String
  accountId       String
  title           String
  status          String   // ACTIVE, INACTIVE, ENDED, ACTIVATING
  price           Decimal
  currency        String   @default("PLN")
  stockAvailable  Int
  stockSold       Int      @default(0)
  externalId      String?  // external.id in Allegro (for mapping)
  lastSyncedAt    DateTime?
  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt

  product         Product        @relation(fields: [productId], references: [id])
  account         AllegroAccount @relation(fields: [accountId], references: [id])
}

// Allegro order synced to local DB
model Order {
  id                String   @id @default(cuid())
  allegroOrderId    String   @unique
  accountId         String
  buyerEmail        String?
  buyerLogin        String?
  totalAmount       Decimal
  currency          String   @default("PLN")
  status            String   // READY_FOR_PROCESSING, SENT, DELIVERED, etc.
  paymentStatus     String?
  shippingCarrier   String?
  trackingNumber    String?
  odooSaleOrderId   Int?     // FK to Odoo sale.order
  orderDate         DateTime
  lastSyncedAt      DateTime?
  createdAt         DateTime @default(now())
  updatedAt         DateTime @updatedAt

  account           AllegroAccount @relation(fields: [accountId], references: [id])
  items             OrderItem[]
}

model OrderItem {
  id              String   @id @default(cuid())
  orderId         String
  productId       String?
  allegroOfferId  String
  title           String
  quantity        Int
  unitPrice       Decimal
  currency        String   @default("PLN")

  order           Order    @relation(fields: [orderId], references: [id])
  product         Product? @relation(fields: [productId], references: [id])
}

// Sync job tracking
model SyncJob {
  id            String   @id @default(cuid())
  accountId     String
  type          String   // ORDERS, OFFERS, STOCK, BILLING
  direction     String   // ALLEGRO_TO_APP, APP_TO_ALLEGRO, APP_TO_ODOO, ODOO_TO_APP
  status        String   // PENDING, RUNNING, COMPLETED, FAILED
  startedAt     DateTime?
  completedAt   DateTime?
  itemsProcessed Int     @default(0)
  errorMessage  String?
  createdAt     DateTime @default(now())

  account       AllegroAccount @relation(fields: [accountId], references: [id])
}

// Generic ID mapping between systems
model ExternalMapping {
  id            String   @id @default(cuid())
  entityType    String   // PRODUCT, ORDER, CUSTOMER
  internalId    String
  allegroId     String?
  odooId        String?
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt

  @@unique([entityType, internalId])
  @@index([entityType, allegroId])
  @@index([entityType, odooId])
}
```

### 4.5 Core Sync Flows

**Order Sync (Allegro → App → Odoo):**
```
1. Scheduled job polls GET /order/events every 2 minutes
2. For each new/changed order event:
   a. Fetch full order via GET /order/checkout-forms/{id}
   b. Upsert Order + OrderItems in local DB
   c. Queue job: create/update Odoo sale.order via JSON-2 API
   d. Queue job: create res.partner if new buyer
3. On fulfillment (Odoo stock.picking validated):
   a. Push tracking number → POST /order/checkout-forms/{id}/shipments
   b. Update order status → PUT /order/checkout-forms/{id}/fulfillment
```

**Stock Sync (Odoo → App → Allegro):**
```
1. Scheduled job polls Odoo stock.quant every 5 minutes
2. Compare stock levels with local Product.stockQuantity
3. For changed products:
   a. Update local DB
   b. Queue batch command: PUT /sale/offer-quantity-change-commands/{commandId}
   c. Poll command status until complete
```

**Product Sync (Odoo → App → Allegro):**
```
1. On demand or scheduled: fetch product.template from Odoo
2. Map Odoo product fields → Allegro offer fields
3. For new products: POST /sale/product-offers
4. For existing: PATCH /sale/product-offers/{offerId}
5. Store Allegro offer ID in ExternalMapping
```

### 4.6 Key Design Decisions

| Decision | Choice | Rationale |
|---|---|---|
| **Queue system** | Bull (Redis-backed) | Handles rate limits, retries, delayed jobs; built-in NestJS integration |
| **Token storage** | Encrypted in PostgreSQL | Refresh tokens are sensitive; encrypt at rest with AES-256 |
| **Multi-account** | AllegroAccount per seller | Support multiple Allegro seller accounts in one instance |
| **Sync strategy** | Event-driven + polling hybrid | Allegro has order events but no push webhooks; poll events endpoint |
| **Odoo API version** | JSON-2 with XML-RPC fallback | Future-proof (JSON-2 is the new standard), fallback for older Odoo |
| **Monorepo** | Turborepo | Shared types between FE/BE, single CI pipeline, atomic deployments |
| **Rate limit handling** | Per-account token bucket in Redis | Track usage per Client ID, delay requests when approaching 9000/min |

---

## Appendix: Useful Links

- [Allegro REST API Documentation](https://developer.allegro.pl/documentation)
- [Allegro API Tutorials](https://developer.allegro.pl/tutorials)
- [Allegro API News / Changelog](https://developer.allegro.pl/news)
- [Allegro API GitHub Issues](https://github.com/allegro/allegro-api/issues)
- [Allegro Sandbox](https://allegro.pl.allegrosandbox.pl/)
- [Allegro App Registration](https://apps.developer.allegro.pl/)
- [Allegro Postman Collection](https://www.postman.com/allegro-rest-api/allegro-rest-api/collection/cztgnvw/allegro-rest-api)
- [Odoo External JSON-2 API Docs](https://www.odoo.com/documentation/19.0/developer/reference/external_api.html)
- [Odoo External RPC API Docs (legacy)](https://www.odoo.com/documentation/19.0/developer/reference/external_rpc_api.html)
