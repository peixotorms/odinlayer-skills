---
name: woocommerce-extensions
description: Use when building WooCommerce extensions or integrating with WooCommerce. Covers WC_Payment_Gateway, WC_Product (wc_get_product), WC_Order (wc_get_order), WC_Data CRUD methods, HPOS (High-Performance Order Storage, OrdersTableDataStore, wc_get_orders, custom_order_tables), Store API endpoints, WooCommerce REST API (/wc/v3/), block checkout (AbstractPaymentMethodType), woocommerce_checkout_process and add_action woocommerce_* hooks, order lifecycle hooks, product hooks, settings pages, custom emails, and FeaturesUtil compatibility declarations.
---

# WooCommerce Extension Development

Reference for building WooCommerce extensions: payment gateways, order and product
CRUD, HPOS, Store API, REST API, block checkout integration, settings, and hooks.

---

## 1. Extension Architecture

### Check WooCommerce Is Active

Always gate your plugin on WooCommerce availability:

```php
add_action( 'plugins_loaded', 'mce_init' );

function mce_init(): void {
    if ( ! class_exists( 'WooCommerce' ) ) {
        add_action( 'admin_notices', function (): void {
            echo '<div class="error"><p>' .
                esc_html__( 'My Extension requires WooCommerce.', 'my-extension' ) .
                '</p></div>';
        } );
        return;
    }
    // Safe to use WooCommerce APIs from here.
}
```

### Plugin Header

```php
/**
 * Plugin Name: My WooCommerce Extension
 * Plugin URI:  https://example.com/my-extension
 * Description: Extends WooCommerce with custom features.
 * Version:     1.0.0
 * Author:      Your Name
 * Requires Plugins: woocommerce
 * WC requires at least: 8.0
 * WC tested up to: 10.4
 * Requires PHP: 7.4
 */
```

The `Requires Plugins: woocommerce` header (WordPress 6.5+) enforces the dependency
automatically. `WC requires at least` and `WC tested up to` are read by WooCommerce
to show compatibility notices.

### Declaring Feature Compatibility

```php
add_action( 'before_woocommerce_init', function (): void {
    if ( class_exists( '\Automattic\WooCommerce\Utilities\FeaturesUtil' ) ) {
        // Declare HPOS compatibility.
        \Automattic\WooCommerce\Utilities\FeaturesUtil::declare_compatibility(
            'custom_order_tables',
            __FILE__,
            true
        );
        // Declare Block Checkout compatibility.
        \Automattic\WooCommerce\Utilities\FeaturesUtil::declare_compatibility(
            'cart_checkout_blocks',
            __FILE__,
            true
        );
    }
} );
```

---

## 2. Modern WooCommerce Architecture

### DI Container

WooCommerce `src/` classes use dependency injection. Access singletons with:

```php
$instance = wc_get_container()->get( \Automattic\WooCommerce\Internal\SomeClass::class );
```

Extensions can access public WC services through the container. Internal classes
(`src/Internal/`) are private API and may change without notice — only rely on
classes in `src/` root or documented APIs.

### Hook Callback Naming Convention

Name hook callbacks `handle_{hook_name}` with `@internal` annotation:

```php
/**
 * Handle the woocommerce_order_status_changed hook.
 *
 * @internal
 */
public function handle_woocommerce_order_status_changed( int $order_id, string $old, string $new ): void {
    // ...
}
```

### Interactivity API Stores Are Private

All WooCommerce Interactivity API stores use `lock: true` — they are **not
extensible**. Removing or changing store state/selectors is not a breaking
change. Do not depend on WC store internals in your extension.

### Data Integrity

Always verify entity state before deletion or modification:

```php
// Verify order status before deletion.
$order = wc_get_order( $order_id );
if ( ! $order || ! in_array( $order->get_status(), array( 'draft', 'checkout-draft' ), true ) ) {
    return false;  // Don't delete non-draft orders.
}
$order->delete( true );

// Verify ownership in batch operations.
foreach ( $order_ids as $id ) {
    $order = wc_get_order( $id );
    if ( $order && (int) $order->get_customer_id() === get_current_user_id() ) {
        $order->delete( true );
    }
}
```

**Checklist:** entity exists -> correct state -> correct owner -> race-condition safe
-> check return value of delete/save.

---

## 3. Payment Gateway API

Register a gateway by adding its class to the `woocommerce_payment_gateways` filter:

```php
add_filter( 'woocommerce_payment_gateways', function ( array $gateways ): array {
    $gateways[] = 'MCE_Gateway';
    return $gateways;
} );
```

Extend `WC_Payment_Gateway` and implement `process_payment()` and optionally
`process_refund()`. See `resources/payment-gateway.md` for the full class skeleton,
tokenization details, and field validation.

### Supported Features

| Feature            | Description                              |
|--------------------|------------------------------------------|
| `products`         | Standard product payments                |
| `refunds`          | Admin refund processing                  |
| `subscriptions`    | WooCommerce Subscriptions support        |
| `tokenization`     | Saved payment methods                    |
| `pre-orders`       | Pre-order charge scheduling              |

---

## 4. HPOS (High-Performance Order Storage)

Since WooCommerce 8.2, HPOS is the default order storage for new installs.
Orders are stored in custom tables, not `wp_postmeta`.

### Accessing Order Data (HPOS-Compatible)

```php
// Always use CRUD methods — never direct SQL or get_post_meta().
$order = wc_get_order( $order_id );

// Read.
$total    = $order->get_total();
$status   = $order->get_status();
$billing  = $order->get_billing_email();
$meta     = $order->get_meta( '_mce_transaction_id' );

// Write.
$order->set_status( 'completed' );
$order->update_meta_data( '_mce_transaction_id', 'txn_123' );
$order->save();
```

### Querying Orders

```php
// Use wc_get_orders() — works with both HPOS and legacy storage.
$orders = wc_get_orders( array(
    'status'     => 'processing',
    'limit'      => 50,
    'orderby'    => 'date',
    'order'      => 'DESC',
    'meta_query' => array(
        array(
            'key'   => '_mce_synced',
            'value' => 'no',
        ),
    ),
) );
```

### What NOT to Do

| Deprecated Pattern                           | HPOS-Compatible Alternative                   |
|----------------------------------------------|-----------------------------------------------|
| `get_post_meta( $order_id, '_key', true )`   | `$order->get_meta( '_key' )`                  |
| `update_post_meta( $order_id, '_key', $v )`  | `$order->update_meta_data( '_key', $v ); $order->save();` |
| `new WP_Query( ['post_type'=>'shop_order'] )`| `wc_get_orders( [...] )`                      |
| Direct SQL on `wp_posts`/`wp_postmeta`       | Use `OrdersTableDataStore` or CRUD methods    |
| `get_post( $order_id )`                      | `wc_get_order( $order_id )`                   |

---

## 5. Product & Customer CRUD

### Products

```php
// Read.
$product = wc_get_product( $product_id );
$name    = $product->get_name();
$price   = $product->get_price();
$sku     = $product->get_sku();
$stock   = $product->get_stock_quantity();
$type    = $product->get_type();            // simple, variable, grouped, external.
$meta    = $product->get_meta( '_mce_custom' );

// Write.
$product->set_regular_price( '29.99' );
$product->update_meta_data( '_mce_custom', 'value' );
$product->save();
```

### Product Types

| Class                    | Type       | Use Case                        |
|--------------------------|------------|---------------------------------|
| `WC_Product_Simple`      | `simple`   | Standard product                |
| `WC_Product_Variable`    | `variable` | Product with variations         |
| `WC_Product_Variation`   | `variation`| Single variation of a variable  |
| `WC_Product_Grouped`     | `grouped`  | Collection of simple products   |
| `WC_Product_External`    | `external` | Affiliate / external product    |

### Customers

```php
$customer = new WC_Customer( $user_id );
$email    = $customer->get_email();
$orders   = $customer->get_order_count();
$spent    = $customer->get_total_spent();

$customer->update_meta_data( '_mce_loyalty', 'gold' );
$customer->save();
```

---

## 6. Store API (Cart & Checkout)

The Store API is an **unauthenticated** public REST API for customer-facing
cart, checkout, and product functionality.

### Key Endpoints

| Method   | Endpoint                     | Purpose                   |
|----------|------------------------------|---------------------------|
| `GET`    | `/wc/store/v1/products`      | List products             |
| `GET`    | `/wc/store/v1/cart`          | Get current cart           |
| `POST`   | `/wc/store/v1/cart/add-item` | Add item to cart          |
| `POST`   | `/wc/store/v1/checkout`      | Place order               |
| `PUT`    | `/wc/store/v1/checkout`      | Update order (9.8+)       |

Use `ExtendSchema` to add custom data to Store API responses. See
`resources/block-checkout-settings-emails.md` for full `ExtendSchema` code.

---

## 7. WooCommerce REST API

The REST API is **authenticated** (consumer key + secret) for admin/back-office operations.

### Authentication

```bash
# Generate keys at WooCommerce > Settings > Advanced > REST API.
curl https://example.com/wp-json/wc/v3/products \
  -u ck_xxx:cs_xxx
```

### Key Namespaces

| Namespace       | Purpose                                |
|-----------------|----------------------------------------|
| `wc/v3`         | Core CRUD (products, orders, customers)|
| `wc/store/v1`   | Store API (unauthenticated, cart/checkout) |
| `wc-admin`      | Admin analytics and reports            |
| `wc-analytics`  | Analytics data                         |

### Common Endpoints

| Endpoint            | Methods          | Description          |
|---------------------|------------------|----------------------|
| `/wc/v3/products`   | GET, POST        | Products CRUD        |
| `/wc/v3/orders`     | GET, POST        | Orders CRUD          |
| `/wc/v3/customers`  | GET, POST        | Customers CRUD       |
| `/wc/v3/coupons`    | GET, POST        | Coupons CRUD         |
| `/wc/v3/reports`    | GET              | Sales reports        |
| `/wc/v3/webhooks`   | GET, POST        | Webhook management   |

---

## 8. Block Checkout Integration

The Block Checkout (default since WooCommerce 8.3) requires separate registration
from the classic `WC_Payment_Gateway`. Register an `AbstractPaymentMethodType`
subclass server-side and a client-side JS payment method. See
`resources/block-checkout-settings-emails.md` for full server and client code.

---

## 9. Settings Page Integration

Add a WooCommerce settings tab with `woocommerce_settings_tabs_array` and render
fields with `woocommerce_admin_fields()`. See `resources/block-checkout-settings-emails.md`
for full settings page code.

### Settings Field Types

| Type         | Description                          |
|--------------|--------------------------------------|
| `text`       | Text input                           |
| `textarea`   | Multi-line text                      |
| `checkbox`   | Enable/disable toggle                |
| `select`     | Dropdown with `options` array        |
| `multiselect`| Multiple selection                   |
| `radio`      | Radio buttons                        |
| `number`     | Numeric input                        |
| `password`   | Password field                       |
| `color`      | Color picker                         |
| `title`      | Section heading (no input)           |
| `sectionend` | Closes a section                     |

---

## 10. Key Hooks Reference

### Order Lifecycle

| Hook (Action)                            | When                                     |
|------------------------------------------|------------------------------------------|
| `woocommerce_new_order`                  | Order first created                      |
| `woocommerce_checkout_order_processed`   | After checkout places order              |
| `woocommerce_payment_complete`           | Payment marked complete                  |
| `woocommerce_order_status_changed`       | Any status transition ($old, $new, $order)|
| `woocommerce_order_status_{status}`      | Transitioned to specific status          |
| `woocommerce_order_refunded`             | Refund processed                         |

### Cart & Checkout

| Hook                                      | Type   | Purpose                              |
|-------------------------------------------|--------|--------------------------------------|
| `woocommerce_add_to_cart`                 | Action | Item added to cart                   |
| `woocommerce_cart_calculate_fees`         | Action | Add custom fees                      |
| `woocommerce_before_calculate_totals`     | Action | Modify cart items before totals      |
| `woocommerce_cart_item_price`             | Filter | Modify displayed cart item price     |
| `woocommerce_checkout_fields`             | Filter | Add/modify checkout fields           |
| `woocommerce_checkout_process`            | Action | Validate checkout before processing  |

### Product

| Hook                                      | Type   | Purpose                              |
|-------------------------------------------|--------|--------------------------------------|
| `woocommerce_product_options_general_product_data` | Action | Add fields to General tab    |
| `woocommerce_process_product_meta`        | Action | Save custom product fields           |
| `woocommerce_single_product_summary`      | Action | Output on single product page        |
| `woocommerce_product_get_price`           | Filter | Modify product price dynamically     |
| `woocommerce_is_purchasable`              | Filter | Control whether product can be bought|

### Email

| Hook                                      | Purpose                                  |
|-------------------------------------------|------------------------------------------|
| `woocommerce_email_classes`               | Register custom email classes            |
| `woocommerce_email_order_details`         | Add content to order emails              |
| `woocommerce_email_before_order_table`    | Content before order table in emails     |
| `woocommerce_email_after_order_table`     | Content after order table in emails      |

### Admin

| Hook                                      | Purpose                                  |
|-------------------------------------------|------------------------------------------|
| `woocommerce_admin_order_data_after_billing_address` | Add fields after billing    |
| `woocommerce_admin_order_data_after_shipping_address`| Add fields after shipping   |
| `manage_edit-shop_order_columns`          | Add columns to orders list               |
| `woocommerce_product_data_tabs`           | Add product data tabs                    |
| `woocommerce_product_data_panels`         | Render product data tab panels           |

---

## 11. Custom Emails

Register custom WC email classes via the `woocommerce_email_classes` filter.
See `resources/block-checkout-settings-emails.md` for the full `MCE_Custom_Email`
class implementation.

---

## 12. WP-CLI for WooCommerce

See `resources/testing-cli.md` for WP-CLI commands and custom command examples.

---

## 13. Testing WooCommerce Extensions

See `resources/testing-cli.md` for the base test class pattern, naming conventions,
and WC logger mocking.

---

## 14. Common Mistakes

| Mistake | Fix |
|---------|-----|
| Using `get_post_meta()` for order data | Use `$order->get_meta()` — required for HPOS |
| `new WP_Query(['post_type'=>'shop_order'])` | Use `wc_get_orders()` — works with HPOS and legacy |
| Not declaring HPOS compatibility | Add `FeaturesUtil::declare_compatibility('custom_order_tables', ...)` |
| Not declaring Block Checkout compatibility | Add `FeaturesUtil::declare_compatibility('cart_checkout_blocks', ...)` |
| Only implementing classic checkout | Register `AbstractPaymentMethodType` for block checkout too |
| Calling `$order->set_status()` without `$order->save()` | CRUD changes are only persisted after `save()` |
| Using `$woocommerce` global instead of `WC()` | Use the `WC()` function — it's the singleton accessor |
| Modifying cart totals with direct property access | Use `woocommerce_before_calculate_totals` hook |
| Direct SQL queries on order tables | Use WC CRUD or `wc_get_orders()` |
| Not checking if WooCommerce is active | Wrap init in `class_exists('WooCommerce')` check |
| Standalone functions outside classes | Always use class methods — standalone functions are hard to mock in tests |
| Using `new` for DI-managed WC classes | Use `wc_get_container()->get( ClassName::class )` for `src/` singletons |
| `call_user_func_array` with associative keys | Always use positional (numerically indexed) arrays — keys are silently ignored |
| Deleting orders without verifying status | Check `$order->get_status()` before `$order->delete()` — could delete completed orders |
| Depending on WC Interactivity API stores | All WC stores are `lock: true` (private) — they can change without notice |
| Batch operations without per-item validation | Verify each item exists, has correct state, and correct owner before modifying |
