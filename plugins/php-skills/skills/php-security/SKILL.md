---
name: php-security
description: Use when handling user input, database queries, file operations, authentication, sessions, or any security-sensitive PHP code. Covers SQL injection prevention, XSS, CSRF, input validation, password hashing, file upload security, session management, error exposure, and common vulnerability patterns.
---

# PHP Security

## Core Principle

**Never trust user input.** Validate everything from `$_GET`, `$_POST`, `$_COOKIE`, `$_FILES`, `$_SERVER`, and `$_REQUEST`. Defense in depth — layer multiple protections.

## SQL Injection Prevention

### Always Use Prepared Statements

```php
// GOOD: Parameterized query — safe
$stmt = $pdo->prepare('SELECT * FROM users WHERE email = ? AND status = ?');
$stmt->execute([$email, $status]);
$user = $stmt->fetch();

// GOOD: Named parameters
$stmt = $pdo->prepare('SELECT * FROM users WHERE id = :id');
$stmt->execute(['id' => $id]);

// BAD: String interpolation — SQL injection!
$pdo->query("SELECT * FROM users WHERE id = $id");
$pdo->query("SELECT * FROM users WHERE name = '$name'");
```

| Rule | Detail |
|------|--------|
| Parameters for data only | `WHERE`, `VALUES`, `SET` — not table/column names |
| Table/column names | Validate against whitelist, never user input directly |
| Least-privilege DB accounts | Don't use root; separate read/write accounts |
| Use PDO with exceptions | `$pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION)` |
| Disable emulated prepares | `$pdo->setAttribute(PDO::ATTR_EMULATE_PREPARES, false)` |

### Dynamic Table/Column Names

```php
// GOOD: Whitelist validation
$allowed = ['name', 'email', 'created_at'];
$column = in_array($column, $allowed, true) ? $column : 'name';
$stmt = $pdo->prepare("SELECT * FROM users ORDER BY {$column}");
```

## XSS Prevention

### Output Escaping

```php
// GOOD: Always escape user data in HTML
echo htmlspecialchars($userInput, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8');

// Helper function
function e(string $value): string {
    return htmlspecialchars($value, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8');
}

// In templates
<p>Hello, <?= e($user->name) ?></p>
<input value="<?= e($searchQuery) ?>">
```

| Context | Escaping |
|---------|----------|
| HTML body | `htmlspecialchars()` with `ENT_QUOTES` |
| HTML attribute | `htmlspecialchars()` with `ENT_QUOTES` |
| JavaScript | `json_encode()` with `JSON_HEX_TAG` |
| URL parameter | `urlencode()` |
| CSS | Avoid user input in CSS; whitelist if needed |

### Content Security Policy

```php
header("Content-Security-Policy: default-src 'self'; script-src 'self'");
header("X-Content-Type-Options: nosniff");
header("X-Frame-Options: DENY");
header("X-XSS-Protection: 0"); // Disable legacy XSS filter — CSP is better
```

## CSRF Protection

```php
// Generate token
session_start();
if (empty($_SESSION['csrf_token'])) {
    $_SESSION['csrf_token'] = bin2hex(random_bytes(32));
}

// In form
<input type="hidden" name="csrf_token" value="<?= e($_SESSION['csrf_token']) ?>">

// Validate on submit
if (!hash_equals($_SESSION['csrf_token'], $_POST['csrf_token'] ?? '')) {
    http_response_code(403);
    exit('Invalid CSRF token');
}
```

| Rule | Detail |
|------|--------|
| Token per session | Generate once, validate on every state-changing request |
| `hash_equals()` | Timing-safe comparison — prevents timing attacks |
| `SameSite=Lax` cookies | Additional protection layer |
| Not needed for GET | GET should never modify state |

## Password Security

```php
// Hash with bcrypt (default, recommended)
$hash = password_hash($password, PASSWORD_DEFAULT);

// Verify
if (password_verify($inputPassword, $storedHash)) {
    // Authenticated
}

// Check if rehash needed (algorithm/cost changes)
if (password_needs_rehash($storedHash, PASSWORD_DEFAULT)) {
    $newHash = password_hash($inputPassword, PASSWORD_DEFAULT);
    // Update stored hash
}
```

| Rule | Detail |
|------|--------|
| Never store plaintext | Always `password_hash()` |
| `PASSWORD_DEFAULT` | Auto-uses best available algorithm |
| `PASSWORD_ARGON2ID` | Stronger than bcrypt if available (PHP 7.3+) |
| Never use `md5()` / `sha1()` | Not designed for passwords — too fast |
| `password_verify()` only | Don't compare hashes manually |
| Rehash on login | `password_needs_rehash()` keeps hashes current |

## Input Validation & Sanitization

```php
// Filter functions
$email = filter_input(INPUT_POST, 'email', FILTER_VALIDATE_EMAIL);
$age = filter_input(INPUT_POST, 'age', FILTER_VALIDATE_INT, [
    'options' => ['min_range' => 0, 'max_range' => 150]
]);
$url = filter_input(INPUT_POST, 'url', FILTER_VALIDATE_URL);

// Manual validation
if (!is_string($name) || mb_strlen($name) > 255) {
    throw new ValidationException('Invalid name');
}

// Type casting (strict mode helps, but validate boundaries)
$id = (int) $_GET['id'];
if ($id <= 0) { throw new NotFoundException(); }
```

| Rule | Detail |
|------|--------|
| Validate type, format, range, length | Before any processing |
| Whitelist over blacklist | Accept known-good, reject everything else |
| Don't use `$_REQUEST` | Ambiguous source — specify `$_GET`, `$_POST`, `$_COOKIE` |
| `$_SERVER` values can be spoofed | `HTTP_*` headers are user-controlled |
| `filter_input()` for simple cases | Manual validation for complex rules |

## File Upload Security

```php
$file = $_FILES['upload'];

// Validate
if ($file['error'] !== UPLOAD_ERR_OK) {
    throw new UploadException('Upload failed');
}

// Check MIME type (don't trust $_FILES['type'] — user-controlled)
$finfo = new finfo(FILEINFO_MIME_TYPE);
$mime = $finfo->file($file['tmp_name']);
$allowed = ['image/jpeg', 'image/png', 'image/gif'];
if (!in_array($mime, $allowed, true)) {
    throw new ValidationException('Invalid file type');
}

// Check size
if ($file['size'] > 5 * 1024 * 1024) { // 5MB
    throw new ValidationException('File too large');
}

// Generate safe filename — never use original
$ext = match($mime) {
    'image/jpeg' => '.jpg',
    'image/png' => '.png',
    'image/gif' => '.gif',
};
$safeName = bin2hex(random_bytes(16)) . $ext;

// Move to non-web-accessible directory
move_uploaded_file($file['tmp_name'], '/var/uploads/' . $safeName);
```

| Rule | Detail |
|------|--------|
| Never trust `$_FILES['name']` | Generate random filename |
| Never trust `$_FILES['type']` | Check with `finfo` on actual file |
| Store outside web root | Serve through PHP controller |
| Validate file content | MIME check, size limits, dimension limits for images |
| No executable uploads | Block `.php`, `.phtml`, `.phar`, etc. |

## Session Security

```php
// Secure session configuration
ini_set('session.cookie_httponly', '1');     // No JavaScript access
ini_set('session.cookie_secure', '1');       // HTTPS only
ini_set('session.cookie_samesite', 'Lax');   // CSRF protection
ini_set('session.use_strict_mode', '1');     // Reject uninitialized IDs
ini_set('session.use_only_cookies', '1');    // No session ID in URL

session_start();

// Regenerate ID on privilege change (login)
session_regenerate_id(true);

// Destroy session properly
session_unset();
session_destroy();
setcookie(session_name(), '', time() - 3600, '/');
```

| Rule | Detail |
|------|--------|
| `httponly` cookies | Prevents XSS session hijacking |
| `secure` flag | Only send over HTTPS |
| Regenerate on login | Prevents session fixation |
| Regenerate on privilege change | Elevation of privilege attacks |
| Absolute timeout | Expire sessions after N hours regardless of activity |
| Idle timeout | Expire after N minutes of inactivity |

## Filesystem Security

```php
// BAD: Path traversal attack
$file = $_GET['file'];
include("/templates/$file");  // ../../etc/passwd

// GOOD: Validate and constrain
$allowed = ['header', 'footer', 'sidebar'];
$file = $_GET['file'];
if (!in_array($file, $allowed, true)) {
    throw new NotFoundException();
}
include("/templates/{$file}.php");

// GOOD: realpath validation
$basePath = realpath('/var/uploads');
$fullPath = realpath("/var/uploads/{$filename}");
if ($fullPath === false || !str_starts_with($fullPath, $basePath)) {
    throw new SecurityException('Path traversal detected');
}
```

| Rule | Detail |
|------|--------|
| Whitelist allowed paths | Never build paths from user input directly |
| `realpath()` + prefix check | Resolves symlinks, detects `..` traversal |
| `basename()` strips directories | But don't rely on it alone |
| Restrict PHP user permissions | Principle of least privilege |
| Log file operations | Audit trail for sensitive access |

## Cryptography

```php
// Random bytes for tokens, keys, nonces
$token = bin2hex(random_bytes(32));  // 64-char hex string
$apiKey = base64_encode(random_bytes(32));

// HMAC for data integrity
$signature = hash_hmac('sha256', $data, $secretKey);
if (!hash_equals($expected, $signature)) {
    throw new SecurityException('Invalid signature');
}

// Encryption with sodium (PHP 7.2+)
$key = sodium_crypto_secretbox_keygen();
$nonce = random_bytes(SODIUM_CRYPTO_SECRETBOX_NONCEBYTES);
$encrypted = sodium_crypto_secretbox($plaintext, $nonce, $key);
$decrypted = sodium_crypto_secretbox_open($encrypted, $nonce, $key);
```

| Rule | Detail |
|------|--------|
| `random_bytes()` / `random_int()` | For all security-sensitive random — never `rand()` or `mt_rand()` |
| `hash_equals()` | Timing-safe comparison for tokens, signatures |
| Sodium over OpenSSL | Simpler API, harder to misuse |
| Never roll your own crypto | Use well-tested libraries |
| Store keys outside code | Environment variables or secret management |

## HTTP Security Headers

```php
// Essential headers
header('Strict-Transport-Security: max-age=31536000; includeSubDomains');
header("Content-Security-Policy: default-src 'self'");
header('X-Content-Type-Options: nosniff');
header('X-Frame-Options: DENY');
header('Referrer-Policy: strict-origin-when-cross-origin');
header('Permissions-Policy: camera=(), microphone=(), geolocation=()');

// Hide PHP version
ini_set('expose_php', '0');
```

## Error Exposure

| Rule | Detail |
|------|--------|
| `display_errors = Off` in production | Errors reveal paths, DB structure, code |
| `log_errors = On` | Log to file or syslog, not stdout |
| Custom error pages | Show generic message, log details |
| Never expose DB errors | Attackers use them for reconnaissance |
| Hide `X-Powered-By` | `expose_php = Off` in php.ini |

## Common Vulnerability Patterns

| Vulnerability | Prevention |
|--------------|------------|
| SQL injection | Prepared statements with bound parameters |
| XSS (stored/reflected) | `htmlspecialchars()` + CSP headers |
| CSRF | Token per session + `SameSite` cookies |
| Session fixation | `session_regenerate_id(true)` on login |
| Path traversal | `realpath()` + prefix validation |
| File upload attacks | MIME validation + random names + store outside webroot |
| Timing attacks | `hash_equals()` for all comparisons |
| Insecure deserialization | Never `unserialize()` user data; use JSON |
| Open redirect | Whitelist redirect targets |
| Mass assignment | Explicit field lists, never `extract()` on user data |
| Header injection | Validate/strip `\r\n` from header values |
| Command injection | `escapeshellarg()` + avoid shell functions when possible |
| Information disclosure | Disable `display_errors`, `expose_php`, `phpinfo()` |

## Security Checklist

- [ ] `declare(strict_types=1)` in every file
- [ ] `===` for all comparisons (especially auth checks)
- [ ] Prepared statements for all database queries
- [ ] `htmlspecialchars()` for all HTML output
- [ ] CSRF tokens on all state-changing forms
- [ ] `password_hash()` / `password_verify()` for passwords
- [ ] `random_bytes()` for tokens and keys
- [ ] Session cookies: `httponly`, `secure`, `samesite`
- [ ] File uploads validated by content, not name/extension
- [ ] `display_errors = Off` in production
- [ ] CSP and security headers set
- [ ] `expose_php = Off`
- [ ] No `$_REQUEST` usage — specify input source
