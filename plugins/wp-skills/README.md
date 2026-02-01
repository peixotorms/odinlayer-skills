# WordPress Skills Plugin

Comprehensive WordPress guidelines as focused Claude Code skills, plus a review command.

## What's Included

**Skills** (auto-activate based on context):

| Skill | Focus |
|-------|-------|
| `wp-guidelines` | Coding standards, naming conventions (functions, variables, classes, files, hooks, post types), global prefix rule, Yoda conditions, strict comparisons, WordPress API alternatives, i18n, enqueuing assets, hooks, deprecated functions, formatting |
| `wp-security` | Output escaping (XSS), nonce verification (CSRF), input sanitization, SQL injection prevention with `$wpdb->prepare()`, capabilities and authorization, file uploads, safe redirects, AJAX security, REST API permissions, common vulnerability patterns |
| `wp-performance` | WP_Query optimization, caching layers (page, object, transients), race conditions, cache stampede prevention, hooks anti-patterns, AJAX and external requests, asset loading, WP-Cron, uncached functions, profiling, platform-specific guidance, 30 anti-patterns |
| `wp-blocks` | Gutenberg block development (block.json, static/dynamic rendering, InnerBlocks, deprecations), block themes (theme.json, templates, patterns, style variations), Interactivity API (directives, SSR, stores), WordPress 6.9 features, tooling |
| `wp-rest-api` | Route registration, WP_REST_Controller, JSON Schema validation, permission callbacks, authentication, response shaping, field registration, CPT/taxonomy REST exposure, Abilities API (WordPress 6.9+), error handling |
| `wp-plugins` | Plugin architecture, lifecycle hooks (activation/deactivation/uninstall), Settings API, data storage, custom tables with dbDelta, WP-CLI commands and operations, PHPStan configuration, PHPCS setup, testing, build and deploy |
| `wp-apis` | WordPress core APIs for plugin/theme development — admin menus, shortcodes, meta boxes, custom post types, taxonomies, HTTP API, WP-Cron scheduling, dashboard widgets, users and roles, privacy (GDPR), theme mods and Customizer, Site Health, global variables, responsive images, advanced hooks |

**Command:**

- `/wp-review` — Review WordPress code against coding standards, security, and performance guidelines

## Installation

```bash
claude plugin marketplace add peixotorms/odinlayer-skills
claude plugin install wp-skills
```

## Usage

Skills activate automatically when Claude detects WordPress-related work. You can also explicitly review code:

```
/wp-review wp-content/plugins/my-plugin/
/wp-review wp-content/themes/my-theme/functions.php
```

## Sources

- [WordPress Coding Standards (WPCS)](https://github.com/WordPress/WordPress-Coding-Standards) — 58 PHP_CodeSniffer sniffs
- [WordPress Agent Skills](https://github.com/WordPress/agent-skills) — Official WordPress Foundation AI skills
- [elvis/claude-wordpress-skills](https://github.com/elvismdev/claude-wordpress-skills) — Performance review patterns
- [WordPress Developer Resources](https://developer.wordpress.org/)
