---
name: php-guidelines
description: Use when writing, reviewing, refactoring, building, or deploying PHP code. Covers type system, strict types, OOP (classes, interfaces, traits, enums), modern PHP 8.x features (match, readonly, property hooks, pipe operator), JSON handling, DI and SOLID principles, PSR standards, Composer, functions, generators, fibers, namespaces, error handling, attributes, arrays, strings, version migration (8.0-8.5), testing, build/deploy, and anti-patterns. For security see php-security; for performance see php-performance; for networking/regex see php-networking; for internationalization see php-intl.
---

# PHP Guidelines

## Overview

Modern PHP guidelines based on the [PHP Manual](https://www.php.net/manual/en/) covering PHP 8.x idioms. PHP has evolved dramatically — write modern, type-safe PHP, not legacy PHP 5 patterns.

### Core Principles

- **Always use `declare(strict_types=1)`** — first line of every file
- **Type everything** — parameters, returns, properties, constants
- **Use `===` not `==`** — loose comparison causes security bugs
- **Errors are exceptions** — catch them, don't suppress with `@`
- **Composition over inheritance** — interfaces + traits over deep hierarchies

## Strict Types

```php
<?php
declare(strict_types=1);  // MUST be first statement

function add(int $a, int $b): int {
    return $a + $b;
}
add("1", "2"); // TypeError in strict mode, silently coerces without it
```

| Mode | Behavior |
|------|----------|
| Coercive (default) | `"123"` silently becomes `123`, truncates floats |
| Strict (`strict_types=1`) | TypeError on any mismatch (except `int` to `float`) |
| Scope | Per-file — each file must declare independently |

## Type System

### Scalar Types

| Type | Falsy values | Gotchas |
|------|-------------|---------|
| `bool` | `false` | `-1` is truthy; `"0"` is falsy but `"false"` is truthy |
| `int` | `0` | Overflow silently becomes `float`; use `PHP_INT_MAX` |
| `float` | `0.0`, `-0.0` | **Never compare for equality** — use epsilon; `NAN != NAN` |
| `string` | `""`, `"0"` | No native Unicode — use `mbstring`; `"0"` is falsy |
| `null` | `null` | `isset()` returns false for null; use `?Type` or `Type\|null` |

### Type Declarations

```php
// Union types (PHP 8.0+)
function handle(int|string $value): string|false { }

// Intersection types (PHP 8.1+)
function process(Countable&ArrayAccess $data): void { }

// DNF types (PHP 8.2+)
function route((Logger&Handler)|NullHandler $h): void { }

// Nullable shorthand
function get(?int $id): ?string { }  // same as int|null

// Special return types
function loop(): never { while(true) {} }   // never returns
function fire(): void { }                    // returns nothing
```

| Type | Use for |
|------|---------|
| `mixed` | Accepts anything (avoid — be specific) |
| `void` | Function returns nothing visible |
| `never` | Function never returns (exit, throw, infinite loop) |
| `iterable` | `array\|Traversable` |
| `callable` | Parameters/returns only — cannot type properties |
| `self`, `static`, `parent` | Class context references |

### Type Juggling Pitfalls

```php
// BAD: loose comparison — security holes
0 == "a"       // false in PHP 8 (was TRUE in PHP 7!)
"0" == false   // true — "0" is falsy
"" == null     // true
"0" == null    // false (inconsistent!)
"123" == "123.0" // true — numeric string comparison

// GOOD: always strict
0 === "a"      // false
"0" === false  // false

// Float precision trap
0.1 + 0.7 == 0.8   // FALSE — binary representation
floor((0.1 + 0.7) * 10) // 7, not 8!
// Use: abs($a - $b) < PHP_FLOAT_EPSILON
```

## OOP

### Classes & Properties

```php
class User {
    // Constructor promotion (PHP 8.0+)
    public function __construct(
        private readonly string $name,       // readonly (PHP 8.1+)
        public int $age = 0,
        public private(set) string $role = 'user', // asymmetric visibility (PHP 8.4+)
    ) {}
}

$user = new User(name: 'Alice', age: 30); // named arguments
```

| Rule | Detail |
|------|--------|
| Always declare property types | Untyped properties are error-prone |
| Use `readonly` for immutable data | Can only be set once (PHP 8.1+) |
| Constructor promotion | Reduces boilerplate — use for simple DTOs |
| `private(set)` | Read public, write private (PHP 8.4+) |
| Dynamic properties deprecated | PHP 8.2+ — declare all properties explicitly |
| Reading uninitialized typed property | Throws Error |

### Property Hooks (PHP 8.4+)

```php
class Temperature {
    public float $celsius {
        get => ($this->fahrenheit - 32) / 1.8;
        set => $this->fahrenheit = ($value * 1.8) + 32;
    }
    public function __construct(private float $fahrenheit) {}
}
```

### Interfaces & Abstract Classes

```php
interface Renderable {
    public function render(): string;
}

abstract class Widget {
    abstract protected function draw(): void;
    public function display(): void { $this->draw(); }
}

class Button extends Widget implements Renderable, Clickable { }
```

| Rule | Detail |
|------|--------|
| Interface methods must be public | All of them |
| Small interfaces | 1-3 methods — compose larger ones |
| Abstract for shared behavior | Interfaces for contracts |
| Avoid constructors in interfaces | Reduces flexibility |
| `final` prevents extension | Use when inheritance isn't intended |
| `final` class constants (8.1+) | Prevent override in children |

### Traits

```php
trait Timestampable {
    public DateTime $createdAt;
    public DateTime $updatedAt;
    public function touch(): void { $this->updatedAt = new DateTime(); }
}

class Post {
    use Timestampable;
    use TraitA, TraitB {
        TraitA::method insteadof TraitB;   // resolve conflict
        TraitB::method as protected aliased; // alias + change visibility
    }
}
```

| Rule | Detail |
|------|--------|
| Horizontal code reuse | Not a substitute for inheritance |
| Precedence | Class > Trait > Parent |
| Each class gets own static copies | Static properties not shared across classes |
| Traits can have constants (8.2+) | Access via using class, not trait name |
| Can require abstract methods | Force using class to implement |

### Enumerations (PHP 8.1+)

```php
// Pure enum
enum Status {
    case Pending;
    case Active;
    case Inactive;
}

// Backed enum — stored as int or string
enum Color: string {
    case Red = '#FF0000';
    case Green = '#00FF00';
    case Blue = '#0000FF';
}

$color = Color::from('#FF0000');      // Color::Red or ValueError
$color = Color::tryFrom('#FFFFFF');   // null (safe)
echo $color->value;                   // '#FF0000'
echo $color->name;                    // 'Red'

// Methods and interfaces on enums
enum Suit: string implements HasLabel {
    case Hearts = 'H';
    case Spades = 'S';

    public function label(): string {
        return match($this) {
            self::Hearts => 'Hearts',
            self::Spades => 'Spades',
        };
    }
}
```

| Rule | Detail |
|------|--------|
| Use enums over class constants | Type-safe, exhaustive matching |
| Back with `int` or `string` only | All cases must have explicit unique values |
| `from()` throws on invalid | `tryFrom()` returns null |
| Can have methods and implement interfaces | But no state (properties) |
| Cases are singletons | Same instance every time |
| Can use traits | But not traits with properties |

### Visibility & Late Static Binding

| Modifier | Access |
|----------|--------|
| `public` | Everywhere (default for interface methods) |
| `protected` | Class + children |
| `private` | Declaring class only — not inherited |
| `public protected(set)` | Read public, write protected (PHP 8.4+) |
| `public private(set)` | Read public, write only in class (PHP 8.4+) |

```php
// static:: resolves at runtime, self:: at compile time
class ParentClass {
    public static function create(): static { return new static(); }
}
class ChildClass extends ParentClass {}
ChildClass::create(); // Returns ChildClass, not ParentClass
```

### Covariance & Contravariance (PHP 7.4+)

| Direction | Rule | Example |
|-----------|------|---------|
| Covariant return | Child can return MORE specific | `Parent: Animal` -> `Child: Dog` |
| Contravariant param | Child can accept LESS specific | `Parent: Dog` -> `Child: Animal` |

### Magic Methods

| Method | Triggered by |
|--------|-------------|
| `__construct()` / `__destruct()` | Object creation / destruction |
| `__get($name)` / `__set($name, $val)` | Accessing undefined property |
| `__call($name, $args)` / `__callStatic()` | Calling undefined method |
| `__toString()` | String conversion |
| `__invoke()` | Using object as function |
| `__serialize()` / `__unserialize()` | Serialization (preferred over `__sleep`/`__wakeup`) |
| `__clone()` | `clone $obj` (shallow copy by default) |
| `__debugInfo()` | `var_dump()` output |

| Rule | Detail |
|------|--------|
| `__construct` / `__destruct` / `__clone` | Can be any visibility; all others must be public |
| Call `parent::__construct()` explicitly | Not called implicitly by PHP |
| `__clone()` runs AFTER shallow copy | Use to deep-copy nested objects |
| Exceptions in `__destruct` during shutdown | Cause fatal error |

## Dependency Injection & SOLID

### Dependency Injection

```php
// BAD: Hard-coded dependency — untestable
class UserService {
    private Database $db;
    public function __construct() {
        $this->db = new MySqlDatabase();  // tight coupling
    }
}

// GOOD: Inject dependency via constructor
class UserService {
    public function __construct(private DatabaseInterface $db) {}
}

// Usage — easy to swap implementations and mock in tests
$service = new UserService(new MySqlDatabase());
$service = new UserService(new SqliteDatabase());  // same interface
$service = new UserService($mockDb);               // testing
```

| Principle | Rule |
|-----------|------|
| **S** — Single Responsibility | One class = one reason to change |
| **O** — Open/Closed | Open for extension, closed for modification — use interfaces |
| **L** — Liskov Substitution | Subtypes must be substitutable for their base types |
| **I** — Interface Segregation | Many small interfaces > one large interface |
| **D** — Dependency Inversion | Depend on abstractions (interfaces), not concretions |

| Rule | Detail |
|------|--------|
| Constructor injection | Preferred — makes dependencies explicit |
| Type-hint interfaces | Not concrete classes — enables swapping |
| DI container ≠ DI | Containers are optional convenience; DI is the pattern |
| Avoid Service Locator | Hiding dependencies inside a container = anti-pattern |
| `final` by default | Mark classes `final` unless designed for extension |

## PSR Standards & Composer

### PSR Standards

| Standard | Purpose |
|----------|---------|
| **PSR-1** | Basic coding standard — `<?php` tag, UTF-8, class naming |
| **PSR-4** | Autoloading — namespace maps to directory structure |
| **PSR-12** / **PER** | Extended coding style — indentation, braces, spacing |
| **PSR-3** | Logger interface (`Psr\Log\LoggerInterface`) |
| **PSR-7** | HTTP message interfaces (request/response) |
| **PSR-11** | Container interface (`Psr\Container\ContainerInterface`) |
| **PSR-15** | HTTP handlers and middleware |

### Composer

```bash
# Initialize project
composer init

# Add dependency
composer require monolog/monolog

# Install from lock file (deployment — deterministic)
composer install --no-dev --optimize-autoloader

# Update to latest compatible versions (development)
composer update

# Autoload — include once at entry point
require 'vendor/autoload.php';
```

| Rule | Detail |
|------|--------|
| Commit `composer.lock` | Ensures identical versions across team/environments |
| `composer install` in production | Never `composer update` — use lock file |
| `--no-dev` in production | Exclude dev dependencies |
| `--optimize-autoloader` / `-o` | Converts PSR-4/PSR-0 to classmap for speed |
| PSR-4 autoloading | Namespace `App\` → directory `src/` |
| `composer dump-autoload -o` | Regenerate optimized autoload after changes |
| Security auditing | `composer audit` checks for known vulnerabilities |

## Modern PHP 8.x Patterns

### Match Expression (PHP 8.0+)

```php
$label = match($status) {
    Status::Active => 'Active',
    Status::Pending, Status::Inactive => 'Not active',
    default => throw new UnexpectedValueException(),
};
// Uses ===, returns value, no fall-through, throws UnhandledMatchError if no match
```

### Named Arguments & Nullsafe (PHP 8.0+)

```php
// Named arguments — any order, skip defaults
array_slice(array: $arr, offset: 2, length: 5, preserve_keys: true);

// Nullsafe — short-circuits to null
$city = $user?->getAddress()?->getCity()?->getName();
```

### Readonly (PHP 8.1+ properties, 8.2+ classes)

```php
// Readonly property
class User {
    public function __construct(public readonly string $name) {}
}

// Readonly class — ALL properties are readonly
readonly class Point {
    public function __construct(public float $x, public float $y) {}
}
```

### Override Attribute (PHP 8.3+)

```php
class Child extends ParentClass {
    #[Override]
    public function process(): void { }  // Error if parent doesn't have process()
}
```

### Pipe Operator (PHP 8.5+)

```php
$result = $input |> trim(...) |> strtolower(...) |> ucfirst(...);
```

### Deprecated & NoDiscard Attributes (PHP 8.4+/8.5+)

```php
#[Deprecated('Use newMethod() instead', since: '2.0')]
public function oldMethod(): void { }

#[NoDiscard]
function validate(string $data): bool { }
(void) validate($data); // explicitly discard
```

## Functions

### Arrow Functions & Closures

```php
// Arrow function — single expression, auto-captures by value
$doubled = array_map(fn($n) => $n * 2, $numbers);

// Traditional closure — captures explicitly
$tax = 0.1;
$withTax = array_map(function($price) use ($tax) {
    return $price * (1 + $tax);
}, $prices);

// First-class callable syntax (PHP 8.1+)
$fn = strlen(...);
$method = $obj->method(...);
$static = MyClass::method(...);
```

### Variadic & Spread

```php
function sum(int ...$nums): int { return array_sum($nums); }
sum(1, 2, 3);
sum(...[1, 2, 3]);  // spread
```

### Generators

```php
function readLines(string $file): Generator {
    $f = fopen($file, 'r');
    try {
        while ($line = fgets($f)) { yield trim($line); }
    } finally {
        fclose($f);
    }
}

// Delegation
function combined(): Generator {
    yield from range(1, 5);
    yield from ['a', 'b', 'c'];
}

// Associative yields
function metadata(): Generator {
    yield 'name' => 'Alice';
    yield 'age' => 30;
}
```

| Rule | Detail |
|------|--------|
| Memory-efficient | Process large datasets without loading into memory |
| State preserved between yields | Execution pauses and resumes |
| `yield from` delegates | Preserves keys — watch for overwrites |
| Return value via `getReturn()` | Available after generator completes |

### Fibers (PHP 8.1+)

```php
$fiber = new Fiber(function(): string {
    $value = Fiber::suspend('paused');
    return "got: $value";
});
$result = $fiber->start();      // 'paused'
$fiber->resume('hello');
$fiber->getReturn();            // 'got: hello'
```

| Fibers vs Generators | Detail |
|---------------------|--------|
| Generators | Stackless, yield only in generator function |
| Fibers | Full-stack suspension, suspend from anywhere in call chain |

## Namespaces

```php
namespace App\Models;

use App\Contracts\Renderable;
use App\Services\{UserService, AuthService};  // group imports (PHP 7.0+)
use function App\Helpers\format_date;
use const App\Config\MAX_RETRIES;
```

| Rule | Detail |
|------|--------|
| First statement (except `declare`) | Must come before any code |
| Per-file scope | Imports don't transfer to included files |
| Functions/constants fall back to global | If not found in namespace |
| Dynamic names bypass imports | `new $className()` ignores `use` statements |
| No whitespace before `namespace` | After `<?php` tag |

### Name Resolution

| Form | Resolution |
|------|-----------|
| `\Full\Path` | Fully qualified — resolves as-is |
| `Imported\Rest` | First segment checked against imports |
| `Unqualified` | Checked against import table, then current namespace |
| `namespace\Foo` | Current namespace prepended |

## Error Handling

```php
try {
    $result = riskyOperation();
} catch (NotFoundException $e) {
    return defaultValue();
} catch (ValidationException | AuthException $e) {
    log($e->getMessage());
    throw $e;
} finally {
    cleanup();  // always runs
}
```

| Rule | Detail |
|------|--------|
| Catch `\Throwable` for everything | `Exception` + `Error` |
| Use union catches | `catch (TypeA \| TypeB $e)` (PHP 8.0+) |
| Catch without variable | `catch (SpecificException)` (PHP 8.0+) |
| Call `parent::__construct()` | In custom exception classes |
| Log exceptions, don't display | Never show stack traces to users |
| `throw` is an expression (8.0+) | `$x ?? throw new Exception()` |
| Never use `@` suppression | Hides real problems; custom handlers still fire |
| PHP 7+ throws `Error` | Not `Exception` — use `\Throwable` |

### Error Configuration

| Setting | Development | Production |
|---------|-------------|------------|
| `error_reporting` | `E_ALL` | `E_ALL` |
| `display_errors` | `On` | `Off` |
| `log_errors` | `On` | `On` |
| `error_log` | stderr | syslog or file |

## Attributes (PHP 8.0+)

```php
#[Attribute(Attribute::TARGET_METHOD | Attribute::IS_REPEATABLE)]
class Route {
    public function __construct(
        public string $path,
        public string $method = 'GET',
    ) {}
}

#[Route('/api/users', method: 'POST')]
public function createUser(): Response { }

// Read via Reflection
$attrs = (new ReflectionMethod($class, $method))->getAttributes(Route::class);
$route = $attrs[0]->newInstance();
```

## Arrays & Strings

### Arrays

```php
$map = ['key' => 'value', 'other' => 42];
$list = [1, 2, 3];

// Destructuring
['name' => $name, 'age' => $age] = $userData;
[$first, , $third] = $list;  // skip second

// Spread (PHP 7.4+ for numeric, 8.1+ for string keys)
$merged = [...$array1, ...$array2];
```

| Rule | Detail |
|------|--------|
| Null coalescing | `$map['key'] ?? $default` |
| Keys are `int\|string` only | Objects/arrays cannot be keys |
| Iteration order preserved | Arrays are ordered maps |
| Empty array is falsy | `if ($arr)` checks non-empty |

### Strings

```php
// Interpolation — use braces for clarity
$msg = "Hello {$user->name}, you have {$count} items";

// Heredoc (indented, PHP 7.3+)
$html = <<<HTML
    <div class="card">
        <h1>{$title}</h1>
    </div>
    HTML;

// Nowdoc — no interpolation
$sql = <<<'SQL'
    SELECT * FROM users WHERE id = :id
    SQL;
```

| Rule | Detail |
|------|--------|
| Use `{$var}` in double-quoted | Not `${var}` (deprecated PHP 8.2) |
| Single-byte encoding | Use `mb_strlen()`, `mb_substr()` for Unicode |
| `===` for string comparison | `==` does numeric coercion on numeric strings |
| Negative offset (7.1+) | `$str[-1]` for last character |

## JSON

```php
// Encode — always use JSON_THROW_ON_ERROR (PHP 7.3+)
$json = json_encode($data, JSON_THROW_ON_ERROR | JSON_UNESCAPED_UNICODE | JSON_UNESCAPED_SLASHES);

// Decode — assoc arrays are faster than objects for data access
$data = json_decode($json, true, 512, JSON_THROW_ON_ERROR);

// Validate without decoding (PHP 8.3+)
if (json_validate($json)) { /* valid */ }

// Large integers — prevent precision loss
$data = json_decode($json, true, 512, JSON_BIGINT_AS_STRING);
```

| Flag | Effect |
|------|--------|
| `JSON_THROW_ON_ERROR` | Throw `JsonException` instead of returning `false`/`null` |
| `JSON_UNESCAPED_UNICODE` | Don't escape UTF-8 chars — smaller output |
| `JSON_UNESCAPED_SLASHES` | Don't escape `/` — cleaner URLs |
| `JSON_PRESERVE_ZERO_FRACTION` | Keep `10.0` instead of `10` |
| `JSON_BIGINT_AS_STRING` | Decode large ints as strings (prevent precision loss) |
| `JSON_NUMERIC_CHECK` | Convert numeric strings to numbers — **use cautiously** (phone numbers!) |
| `JSON_INVALID_UTF8_SUBSTITUTE` | Replace broken UTF-8 with `U+FFFD` (PHP 7.2+) |
| `JSON_PRETTY_PRINT` | Human-readable output — dev/debug only |

| Rule | Detail |
|------|--------|
| Always `JSON_THROW_ON_ERROR` | Never check `json_last_error()` manually |
| Input must be UTF-8 | `mb_convert_encoding()` first if unsure |
| `json_validate()` (8.3+) | Faster than decode when you just need validity |
| Assoc arrays over objects | `json_decode($json, true)` — faster property access |

## Testing

```php
class UserTest extends TestCase {
    public function testCreateUser(): void {
        $user = new User('Alice', 30);
        $this->assertSame('Alice', $user->getName());
    }

    #[DataProvider('ageProvider')]
    public function testAgeValidation(int $age, bool $valid): void {
        if (!$valid) {
            $this->expectException(\InvalidArgumentException::class);
        }
        new User('Test', $age);
    }

    public static function ageProvider(): array {
        return [
            'valid' => [25, true],
            'zero' => [0, false],
            'negative' => [-1, false],
        ];
    }
}
```

| Rule | Detail |
|------|--------|
| `assertSame()` over `assertEquals()` | Strict comparison (type + value) |
| Data providers for table-driven tests | `#[DataProvider('method')]` attribute |
| `expectException()` before the call | Not after |
| Test naming | `*Test.php`, method `test*` |
| `t.Helper()` equivalent | N/A — PHP traces show full stack |

### Static Analysis

| Tool | Usage |
|------|-------|
| PHPStan | `composer require --dev phpstan/phpstan` — levels 0–9 strictness, start low and increase |
| Psalm | `composer require --dev vimeo/psalm` — similar to PHPStan, adds taint analysis for security |

```bash
# Run in CI and locally before commits
vendor/bin/phpstan analyse src/ --level=6
vendor/bin/psalm --show-info=false
```

| Rule | Detail |
|------|--------|
| Baseline file for legacy code | `phpstan.neon` with `ignoreErrors`, reduce over time |
| Level progression | Start level 0, increase one level per sprint/cycle |
| CI gate | Fail build on any new error — never ignore regressions |
| Psalm taint analysis | Detects SQL injection, XSS flows automatically — see `php-security` |

---

## PHP Version Migration Reference

### PHP 8.0

**Adopt:** named args, union types, match, constructor promotion, nullsafe `?->`, attributes, `throw` as expression, `static` return type, `str_contains()`, `str_starts_with()`, `str_ends_with()`

**Breaking:** `0 == "a"` now false; internal functions throw TypeError on wrong types; private methods not inherited; `\Stringable` interface auto-implemented

### PHP 8.1

**Adopt:** enums, readonly properties, fibers, intersection types, `never` type, first-class callables `fn(...)`, `new` in initializers, `array_is_list()`, `final` class constants

**Stop using:** passing `null` to non-nullable internals; implicit float-to-int; `Serializable` without `__serialize()`

### PHP 8.2

**Adopt:** readonly classes, DNF types `(A&B)|C`, `true`/`false`/`null` as standalone types, trait constants, `Random\Randomizer`

**Stop using:** dynamic properties; `${var}` interpolation; `utf8_encode()`/`utf8_decode()`

### PHP 8.3

**Adopt:** typed class constants, `#[Override]`, dynamic constant fetch `C::{$name}`, `json_validate()`, `str_increment()`/`str_decrement()`

**Stop using:** `get_class()` without arg (use `$obj::class`); string increment on non-alphanumeric

### PHP 8.4

**Adopt:** property hooks, asymmetric visibility `private(set)`, `#[Deprecated]` attribute, lazy objects, `new` without parens in chain, `mb_trim()`/`mb_ltrim()`/`mb_rtrim()`

**Stop using:** implicit nullable `f(Type $x = null)` (use `?Type`); `trigger_error(E_USER_ERROR)` (use exceptions)

### PHP 8.5

**Adopt:** pipe operator `|>`, `#[NoDiscard]`, `(void)` cast, closures in constants, attributes on constants, `#[Override]` on properties

**Stop using:** `(boolean)`/`(integer)`/`(double)` casts; backtick operator; `__sleep()`/`__wakeup()`

---

## Build & Deploy

### Always Use Makefile

Before running `composer install` or any build command, check if a `Makefile` exists. If it does, **use it**.

| Situation | Action |
|-----------|--------|
| Makefile exists with relevant target | `make deploy`, `make build`, `make test` |
| Makefile exists, no matching target | List targets, pick closest |
| No Makefile | `composer install`, `php artisan`, etc. |

### Permissions & Ownership

```bash
stat -c '%U:%G' * | sort | uniq -c | sort -rn | head -5
chown -R <user>:<group> .
```

### Temporary Files

| File type | Location |
|-----------|----------|
| Build intermediates | `/tmp` — never the project directory |
| Dependencies | `vendor/` (Composer standard) |

### Test File Placement

| Test type | Location |
|-----------|----------|
| Temporary | `/tmp` |
| Permanent, dir exists | `tests/` (follow existing structure) |
| Permanent, no dir | Create `tests/` at project root |

---

## Anti-Pattern Quick Reference

| Anti-Pattern | Better Alternative |
|--------------|--------------------|
| No `strict_types` | `declare(strict_types=1)` in every file |
| `==` comparison | `===` everywhere |
| `@` error suppression | Try-catch or proper validation |
| No type declarations | Type params, returns, properties, constants |
| `catch (Exception)` only | `catch (\Throwable)` for `Error` too |
| Dynamic properties | Declare explicitly (`#[AllowDynamicProperties]` if must) |
| Class constants for finite sets | `enum` (PHP 8.1+) |
| Deep inheritance | Interfaces + traits composition |
| `__sleep()` / `__wakeup()` | `__serialize()` / `__unserialize()` |
| `${var}` interpolation | `{$var}` |
| `get_class()` no arg | `$obj::class` |
| `switch` fall-through | `match` expression |
| Manual null chain | Nullsafe `?->` |
| `array_key_exists` + access | `??` null coalescing |
| String constants | Backed enums |
| Float `==` | Epsilon: `abs($a - $b) < PHP_FLOAT_EPSILON` |
| `global $var` | Dependency injection |
| `extract()` | Explicit assignment |
| Implicit nullable param | Explicit `?Type` |
| `(boolean)` cast | `(bool)` |
| `json_last_error()` checking | `JSON_THROW_ON_ERROR` flag |
