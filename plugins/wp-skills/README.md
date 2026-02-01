# WordPress Skills Plugin

Comprehensive WordPress, WooCommerce, and Elementor guidelines as focused Claude Code skills, plus review commands.

## What's Included

### WordPress Core (8 skills)

| Skill | Focus |
|-------|-------|
| `wp-guidelines` | Coding standards, naming conventions, Yoda conditions, strict comparisons, WordPress API alternatives, i18n, enqueuing assets, hooks, deprecated functions, formatting |
| `wp-security` | Output escaping (XSS), nonce verification (CSRF), input sanitization, SQL injection prevention, capabilities, file uploads, safe redirects, AJAX security, REST API permissions |
| `wp-performance` | WP_Query optimization, caching layers (page, object, transients), hooks anti-patterns, AJAX and external requests, asset loading, WP-Cron, profiling, 30 anti-patterns |
| `wp-blocks` | Gutenberg block development (block.json, static/dynamic rendering, InnerBlocks), block themes (theme.json, templates, patterns), Interactivity API, WordPress 6.9 features |
| `wp-rest-api` | Route registration, WP_REST_Controller, JSON Schema validation, permission callbacks, authentication, response shaping, Abilities API (WordPress 6.9+), error handling |
| `wp-plugins` | Plugin architecture, lifecycle hooks, Settings API, data storage, custom tables (dbDelta), WP-CLI, PHPStan, PHPCS, testing, build/deploy, WP 6.7–6.9 breaking changes |
| `wp-apis` | Core APIs — admin menus, shortcodes, meta boxes, CPTs, taxonomies, HTTP API, WP-Cron, dashboard widgets, users/roles, privacy (GDPR), theme mods, Site Health, responsive images |
| `wp-javascript` | Script enqueuing, wp_localize_script, wp_add_inline_script, AJAX handlers (wp_ajax_), Heartbeat API, wp.apiFetch, jQuery noConflict, wp.template, defer/async |

### WooCommerce (4 skills)

| Skill | Focus |
|-------|-------|
| `woocommerce-setup` | Extension architecture, plugin headers, WooCommerce availability check, FeaturesUtil compatibility declarations (HPOS, block checkout), DI container |
| `woocommerce-payments` | Payment gateways (WC_Payment_Gateway), process_payment, process_refund, tokenization, block checkout (AbstractPaymentMethodType, registerPaymentMethod) |
| `woocommerce-data` | HPOS order storage, wc_get_order, wc_get_orders, product/customer CRUD, Store API, REST API (/wc/v3/), ExtendSchema |
| `woocommerce-hooks` | Order lifecycle, cart/checkout, product, email, admin hooks, settings pages, custom WC_Email classes, WP-CLI, testing with WC_Unit_Test_Case |

### Elementor (5 skills)

| Skill | Focus |
|-------|-------|
| `elementor-development` | Addon structure, widget lifecycle, rendering, controls, dependencies, caching, inline editing, manager registration, scripts & styles |
| `elementor-controls` | 56+ control types (text, select, color, slider, media, repeater, etc.), group controls (typography, background, border, box-shadow), selectors, responsive controls, dynamic content |
| `elementor-hooks` | PHP action hooks, PHP filter hooks, JS hooks & commands, injecting controls into existing widgets |
| `elementor-forms` | Form actions, custom field types, validation, rendering, dependencies, content templates |
| `elementor-themes` | Theme builder locations, conditions, dynamic tags, Hello Elementor theme, Finder categories, context menu extensions |

### Commands

- `/wp-review` — Review WordPress code against coding standards, security, and performance guidelines
- `/elementor-review` — Review Elementor addon code against development guidelines

## Installation

```bash
claude plugin marketplace add peixotorms/odinlayer-skills
claude plugin install wp-skills
```

## Usage

Skills activate automatically based on context. You can also explicitly review code:

```
/wp-review wp-content/plugins/my-plugin/
/wp-review wp-content/themes/my-theme/functions.php
/elementor-review wp-content/plugins/my-addon/
```

## Sources

- [WordPress Coding Standards (WPCS)](https://github.com/WordPress/WordPress-Coding-Standards)
- [WordPress Agent Skills](https://github.com/WordPress/agent-skills)
- [WordPress Developer Resources](https://developer.wordpress.org/)
- [WooCommerce CLAUDE.md](https://github.com/woocommerce/woocommerce/blob/trunk/CLAUDE.md)
- [WooCommerce Developer Docs](https://developer.woocommerce.com/)
- [Elementor Developer Docs](https://developers.elementor.com/docs/)
