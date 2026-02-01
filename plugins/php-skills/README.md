# PHP Skills Plugin

Modern PHP guidelines as focused Claude Code skills, plus a review command.

## What's Included

**Skills** (auto-activate based on context):

| Skill | Focus |
|-------|-------|
| `php-guidelines` | Type system, strict types, OOP (classes, interfaces, traits, enums), modern PHP 8.x patterns (match, readonly, property hooks, pipe operator, attributes), functions, generators, fibers, namespaces, error handling, performance, version migration (8.0-8.5), testing, build/deploy, anti-patterns |
| `php-security` | SQL injection prevention, XSS, CSRF, input validation, password hashing, file upload security, session management, filesystem security, cryptography, HTTP headers, common vulnerability patterns |

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
- PHP 8.0-8.5 Migration Guides
