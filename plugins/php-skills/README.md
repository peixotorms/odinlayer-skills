# PHP Skills Plugin

Modern PHP guidelines as focused Claude Code skills, plus a review command.

## What's Included

**Skills** (auto-activate based on context):

| Skill | Focus |
|-------|-------|
| `php-guidelines` | Type system, strict types, OOP (classes, interfaces, traits, enums), modern PHP 8.x patterns (match, readonly, property hooks, pipe operator), DI & SOLID, PSR standards, Composer, JSON handling, functions, generators, fibers, namespaces, error handling, attributes, arrays, strings, version migration (8.0-8.5), testing, build/deploy, anti-patterns |
| `php-security` | SQL injection, XSS, CSRF, input validation (filter_var, FILTER_VALIDATE_*, FILTER_SANITIZE_*), output escaping, password hashing, file uploads, session management, serialization security, process execution security, filesystem security, cryptography, HTTP headers, php.ini hardening (OWASP), common vulnerability patterns |
| `php-performance` | OPcache (configuration, preloading, JIT, monitoring), database (PDO), caching (Redis, Memcached), SPL data structures, micro-optimizations (function choices, memory management, loops, casting), profiling |
| `php-networking` | cURL & HTTP requests, multi-curl (parallel), regex (PCRE, modifiers, named captures, ReDoS prevention), parallel & async (pcntl_fork, stream_select, parallel extension, fibers) |
| `php-intl` | UTF-8 everywhere (mbstring), encoding detection & conversion, database & JSON encoding, intl extension (Collator, NumberFormatter, IntlDateFormatter, MessageFormatter, Transliterator, Normalizer), HTTP charset pipeline |

**Command:**

- `/php-review` — Review PHP code against the guidelines

## Installation

```bash
claude plugin marketplace add peixotorms/odinlayer-skills
claude plugin install php-skills
```

## Usage

Skills activate automatically when Claude detects PHP-related work. You can also explicitly review code:

```
/php-review src/Controller/UserController.php
/php-review app/Models/User.php
```

## Sources

- [PHP Manual](https://www.php.net/manual/en/)
- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/)
- [PHP The Right Way](https://phptherightway.com/)
- PHP 8.0-8.5 Migration Guides
