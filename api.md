# HTTP API

CoreCart exposes JSON-based HTTP endpoints for the storefront, customer account, administration area, and health checks.

> The HTTP interface may change before the first stable release.

## Base Behavior

Requests and responses use UTF-8.

JSON responses use:

```http
Content-Type: application/json; charset=utf-8
```

A successful response generally follows this format:

```json
{
  "status": "success",
  "message": "ok",
  "data": {}
}
```

An error response generally follows this format:

```json
{
  "status": "error",
  "code": "ERROR",
  "message": "Something went wrong"
}
```

Validation errors may include field-specific messages.

## Request Data

Query parameters are used for filtering, pagination, and resource identifiers.

JSON request bodies should use:

```http
Content-Type: application/json
```

Form requests may use:

```http
Content-Type: application/x-www-form-urlencoded
```

State-changing requests may require a CSRF token:

```http
X-CSRF-Token: TOKEN
```

or:

```json
{
  "_csrf_token": "TOKEN"
}
```

## Authentication

CoreCart uses session-based authentication.

There are two separate user contexts:

- customer authentication;
- administrator authentication.

Protected customer endpoints require a valid customer session.

Protected administration endpoints require a valid administrator session.

Cart and checkout endpoints may support both guests and authenticated customers. When a customer session is available, the request should operate on the customer cart.

## Catalog

### List products

```http
GET /catalog/product
GET /api/product
```

Returns active products with pagination.

Common query parameters:

| Parameter | Description |
|---|---|
| `page` | Page number |
| `limit` | Requested page size when supported |

### View a product

```http
GET /catalog/product/view?id=1
GET /api/product/view?id=1
```

Returns one product.

### Search products

```http
GET /catalog/product/search?q=phone
GET /api/product/search?q=phone
```

Returns active products matching the search term.

### List categories

```http
GET /catalog/category
GET /api/category
```

Returns active categories.

### View a category

```http
GET /catalog/category/view?id=1
GET /api/category/view?id=1
```

Returns the category and its products when supported.

## Cart

Cart endpoints operate on the current guest session or authenticated customer.

### View the cart

```http
GET /cart
GET /api/cart
```

Example response data:

```json
{
  "items": [],
  "total": "0.0000",
  "count": 0
}
```

### Add an item

```http
POST /cart/add
POST /api/cart/add
```

Example body:

```json
{
  "product_id": 10,
  "quantity": 2,
  "_csrf_token": "TOKEN"
}
```

The requested total quantity must not exceed available stock.

### Update an item

```http
POST /cart/update
POST /api/cart/update
```

Example body:

```json
{
  "cart_id": 15,
  "quantity": 3,
  "_csrf_token": "TOKEN"
}
```

A quantity of zero may remove the item.

### Remove an item

```http
POST /cart/remove
POST /api/cart/remove
```

Example body:

```json
{
  "cart_id": 15,
  "_csrf_token": "TOKEN"
}
```

### Clear the cart

```http
POST /cart/clear
POST /api/cart/clear
```

### Get cart item count

```http
GET /cart/count
GET /api/cart/count
```

## Checkout

Checkout creates an order from the current cart.

### View checkout data

```http
GET /checkout
GET /api/checkout
```

Returns the current cart and total.

### Confirm checkout

```http
POST /checkout/confirm
POST /api/checkout/confirm
```

Example body:

```json
{
  "customer_email": "customer@example.com",
  "customer_phone": "+10000000000",
  "shipping_firstname": "John",
  "shipping_lastname": "Doe",
  "shipping_address_1": "123 Main Street",
  "shipping_address_2": null,
  "shipping_city": "New York",
  "shipping_postcode": "10001",
  "shipping_country": "US",
  "shipping_zone": "NY",
  "comment": "",
  "_csrf_token": "TOKEN"
}
```

For authenticated customers, a saved address identifier may be supported:

```json
{
  "address_id": 4,
  "_csrf_token": "TOKEN"
}
```

Order creation must validate stock and complete atomically.

Successful response:

```json
{
  "status": "success",
  "message": "Order placed successfully",
  "data": {
    "order_id": 100
  }
}
```

### Checkout success

```http
GET /checkout/success?order_id=100
GET /api/checkout/success?order_id=100
```

The order must match the order stored for the current session. Authenticated customers must only access their own orders.

## Customer Authentication

### Login

```http
GET /account/login
POST /account/login
```

Example POST body:

```json
{
  "email": "customer@example.com",
  "password": "secret",
  "_csrf_token": "TOKEN"
}
```

A successful login regenerates the session identifier and merges the guest cart into the customer cart.

### Register

```http
GET /account/register
POST /account/register
```

Example body:

```json
{
  "username": "john",
  "email": "john@example.com",
  "password": "secret123",
  "_csrf_token": "TOKEN"
}
```

### Logout

```http
POST /account/logout
```

## Customer Account

The following endpoints require customer authentication.

### Profile

```http
GET /account/profile
GET /api/account/profile
```

### Change password

```http
POST /account/password
POST /api/account/password
```

### Addresses

```http
GET  /account/address
POST /account/address/create
POST /account/address/edit?id=4
POST /account/address/delete
```

API equivalents may use the `/api/account/` prefix.

### Orders

```http
GET /account/order
GET /account/order/view?id=100
```

A customer must only be able to access orders owned by that customer.

## Administration

Administration endpoints require administrator authentication unless explicitly documented as public.

### Authentication

```http
GET  /admin/auth/login
POST /admin/auth/loginPost
POST /admin/auth/logout
```

### CSRF token

```http
GET /admin/csrf-token
```

### Dashboard

```http
GET /admin/dashboard
```

### Products

```http
GET  /admin/product/index
POST /admin/product/create
POST /admin/product/update
POST /admin/product/delete
```

### Categories

```http
GET  /admin/category/index
POST /admin/category/create
POST /admin/category/update
POST /admin/category/delete
```

### Orders

```http
GET  /admin/order/index
GET  /admin/order/view?id=100
POST /admin/order/updateStatus
```

Valid order states are:

| Value | State |
|---:|---|
| `0` | Pending |
| `1` | Processing |
| `2` | Shipped |
| `3` | Delivered |
| `9` | Cancelled |

Only valid state transitions are accepted.

### Customers

```http
GET /admin/customer/index
GET /admin/customer/view?id=1
```

## Health Checks

### Liveness

```http
GET /health/live
```

Confirms that the PHP application process can respond.

Example:

```json
{
  "status": "ok"
}
```

### Readiness

```http
GET /health/ready
```

Checks required application dependencies such as the database.

A ready application returns `200`.

A degraded or unavailable application should return `503` without exposing credentials, SQL statements, stack traces, or raw exception messages.

## Status Codes

| Status | Meaning |
|---:|---|
| `200` | Request completed |
| `201` | Resource created |
| `302` | Redirect |
| `400` | Invalid request |
| `401` | Authentication required |
| `403` | Forbidden or invalid CSRF token |
| `404` | Resource not found |
| `405` | HTTP method not allowed |
| `409` | Resource conflict |
| `413` | Request body too large |
| `415` | Unsupported content type |
| `422` | Validation failed |
| `429` | Too many requests |
| `500` | Internal server error |
| `503` | Application not ready |
