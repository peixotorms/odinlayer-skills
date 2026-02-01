---
name: php-review
description: Review PHP code against the guidelines
user_invocable: true
arguments:
  - name: file
    description: File or directory to review
    required: false
---

Review the specified PHP file (or current file) against the php-guidelines and php-security skills.

Check for:
1. **Strict types** — `declare(strict_types=1)` present, type declarations on params/returns/properties
2. **Type safety** — no loose `==`, no type juggling risks, proper nullable handling
3. **OOP** — proper visibility, readonly where appropriate, enums over constants, final where needed
4. **Modern syntax** — match over switch, nullsafe, named args, constructor promotion, pipe operator
5. **Error handling** — proper try-catch, no `@` suppression, \Throwable where needed
6. **Security** — prepared statements, htmlspecialchars output, CSRF tokens, password_hash, no user input in paths/SQL/shell
7. **Performance** — generators for large data, proper autoloading, no string concat in loops
8. **Deprecated patterns** — flag anything deprecated in PHP 8.x

For each issue found, cite the specific guideline and suggest a fix with code.
