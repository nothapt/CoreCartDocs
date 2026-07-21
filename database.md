# Database

CoreCart uses MySQL with InnoDB and `utf8mb4`.

The default table prefix is:

```text
cc_
```

The schema is stored in:

```text
install/database.sql
```

## Main Relationships

```text
Language
 ├── Product Descriptions
 └── Category Descriptions

Customer
 ├── Addresses
 ├── Cart Items
 └── Orders
      └── Order Products

Product
 ├── Product Descriptions
 ├── Product-to-Category
 ├── Cart Items
 └── Order Products

Category
 ├── Category Descriptions
 └── Product-to-Category
```

## Tables

### `cc_language`

Stores supported languages.

Important fields:

- `language_id`
- `code`
- `name`
- `locale`

Product and category descriptions reference a language.

### `cc_admin_user`

Stores administrator accounts.

Important fields:

- `admin_id`
- `username`
- `email`
- `password`
- `status`
- `last_login`
- `last_ip`
- `date_added`

Passwords contain hashes, never plaintext values.

### `cc_login_attempt`

Stores authentication attempts used by rate limiting.

Important fields:

- `attempt_id`
- `ip_address`
- `login`
- `success`
- `date_added`

Old records should be removed periodically.

### `cc_product`

Stores language-independent product data.

Important fields:

- `product_id`
- `model`
- `sku`
- `price`
- `quantity`
- `image`
- `status`
- `date_added`

Prices use `DECIMAL`, not floating-point values.

### `cc_product_description`

Stores localized product content.

The primary key is:

```text
(product_id, language_id)
```

Important fields:

- `name`
- `description`

Deleting a product deletes its descriptions.

### `cc_category`

Stores category hierarchy and status.

Important fields:

- `category_id`
- `parent_id`
- `status`

`parent_id = 0` represents a root category.

Application logic must prevent self-parenting and cyclic category relationships.

### `cc_category_description`

Stores localized category names.

The primary key is:

```text
(category_id, language_id)
```

### `cc_product_to_category`

Implements the many-to-many relationship between products and categories.

The primary key is:

```text
(product_id, category_id)
```

### `cc_customer`

Stores customer accounts.

Important fields:

- `customer_id`
- `username`
- `email`
- `password`
- `status`
- `date_added`

Username and email are unique.

### `cc_address`

Stores customer addresses.

Important fields:

- `address_id`
- `customer_id`
- `firstname`
- `lastname`
- `address_1`
- `address_2`
- `city`
- `postcode`
- `country`
- `zone`
- `default`

An address belongs to one customer.

Changing the default address should be transactional so that the customer does not end up with inconsistent default flags.

### `cc_order`

Stores orders and order snapshots.

Important fields:

- `order_id`
- `customer_id`
- `status`
- `total`
- `comment`
- customer email and phone snapshot;
- shipping address snapshot;
- currency code and value;
- creation and modification timestamps.

`customer_id` may be null for guest orders or after customer deletion.

The contact, shipping, and currency fields preserve purchase-time values even if the customer later edits their profile.

### `cc_order_product`

Stores the products included in an order.

Important fields:

- `order_product_id`
- `order_id`
- `product_id`
- `name`
- `quantity`
- `price`

The product name and price are snapshots. Historical orders must not change when catalog data changes.

### `cc_cart`

Stores guest and customer cart items.

Important fields:

- `cart_id`
- `customer_id`
- `session_id`
- `product_id`
- `quantity`
- `date_added`

A guest row uses `session_id` and a null `customer_id`.

A customer row uses `customer_id`.

Unique constraints prevent duplicate product rows for the same guest session or customer:

```text
uq_session_product(session_id, product_id)
uq_customer_product(customer_id, product_id)
```

Application logic must still handle concurrent inserts and merges safely.

### `cc_setting`

Stores application settings.

Important fields:

- `setting_id`
- `setting_key`
- `setting_value`
- `setting_group`

Sensitive secrets should not be stored in general settings unless encrypted and access-controlled.

## Monetary Values

Money is stored using fixed-precision decimal columns:

```sql
DECIMAL(15,4)
```

Application calculations should use BCMath or another decimal-safe approach.

Do not convert monetary values to binary floating point for totals, taxes, prices, or refunds.

## Order Status

Current status values are:

| Value | Status |
|---:|---|
| `0` | Pending |
| `1` | Processing |
| `2` | Shipped |
| `3` | Delivered |
| `9` | Cancelled |

Status changes should use explicit transition rules.

Queries must not assume that a numerically higher status always represents revenue. Cancelled orders use value `9` and must be excluded explicitly.

## Checkout Transaction

Order creation should be one atomic transaction:

```text
Begin transaction
→ Lock cart rows
→ Read and validate products
→ Validate stock
→ Calculate total
→ Insert order snapshot
→ Insert order products
→ Decrease stock
→ Clear cart
→ Commit
```

Any failure must roll back the complete operation.

Relevant product rows should be locked or updated with a quantity condition to prevent overselling.

## Foreign Keys

Foreign keys protect important relationships.

Typical behaviors include:

- product deletion cascades to descriptions;
- category deletion cascades to descriptions;
- customer deletion cascades to addresses and cart rows;
- customer deletion sets order `customer_id` to null;
- order deletion cascades to order products;
- product deletion is restricted when referenced by an order product.

Historical order data should remain available even when related catalog or customer data changes.

## Transactions Beyond Checkout

Transactions are also recommended for:

- product and product-description creation or update;
- category and category-description update;
- default-address changes;
- cart merge operations;
- multi-record administration actions.

## Migrations

The current schema is distributed as a complete installation file.

As the project evolves, production upgrades should use ordered migrations instead of reimporting the full schema.

A migration system should provide:

- a unique migration identifier;
- ordered execution;
- a migration history table;
- safe failure handling;
- rollback guidance where practical;
- separate schema and data migrations.

## Installation

A development database may be initialized with:

```bash
mysql   --host=127.0.0.1   --user=root   --password   corecart < install/database.sql
```

The CLI installer may also create the database, import the schema, create an administrator, and write configuration.

Do not use default administrator credentials in production.

## Backups

A production backup plan should include:

- regular database dumps or managed snapshots;
- encrypted backup storage;
- retention rules;
- restore testing;
- application files and uploaded media;
- documented recovery procedures.

A backup is not considered reliable until restoration has been tested.
