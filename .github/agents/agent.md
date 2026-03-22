# DZCP Core-Beta – Agent Documentation

This document provides context, conventions, and guidance for automated agents (AI coding assistants, bots, CI agents) working on this repository.

---

## Project Overview

**DZCP (deV!L`z ClanPortal)** is a PHP-based clan portal / community management system. The codebase targets **PHP 7.0+** and uses **PDO** for database access, **Smarty** as the template engine, and **phpFastCache** for multi-tier caching.

---

## Directory Structure

```
/
├── admin/            – Administration panel pages
├── artikel/          – Article/news module
├── banner/           – Banner management module
├── downloads/        – Download management module
├── forum/            – Forum module
├── inc/              – Core system files (bootstrap, classes, helpers)
│   ├── _cache_/      – File-based cache storage (runtime, not committed)
│   ├── _logs_/       – Runtime log files (not committed)
│   ├── _templates_/  – Smarty template directories
│   ├── _templates_c/ – Smarty compiled template cache (not committed)
│   ├── _uploads_/    – User-uploaded files (not committed)
│   ├── additional-functions/ – Optional extra functions (PHP files)
│   ├── additional-languages/ – Optional extra language files
│   ├── configs/      – System configuration (config.php, db config)
│   ├── images/       – Static images for the core system
│   ├── lang/         – Language files
│   ├── menu-functions/ – Menu/navigation helpers
│   ├── securimage/   – CAPTCHA library
│   ├── bbcode.php    – BBCode parser
│   ├── cache.php     – Cache class (wraps phpFastCache, supports file/memory/network tiers)
│   ├── common.php    – Main application bootstrap & helper class
│   ├── cookie.php    – Cookie helper (static class)
│   ├── crypt.php     – Simple XOR-based symmetric encryption class
│   ├── database.php  – PDO database wrapper (supports mysql/pgsql/sqlsrv)
│   ├── debugger.php  – Debug console & error-logging helpers
│   ├── fileman.php   – File manager helpers
│   ├── javascript.php – JavaScript output helpers
│   ├── netapi.php    – Network/IP utility helpers
│   ├── notification.php – In-page notification system
│   ├── secure.php    – Input sanitisation helpers & session startup
│   ├── sessions.php  – Custom session handlers (Memcache/APC/MySQL backends)
│   ├── settings.php  – Settings table CRUD (extends common)
│   ├── sfs.php       – Stop Forum Spam integration
│   └── stringParser.php – String encoding/decoding helpers
├── index.php         – Front-end entry point
├── ajax.php          – AJAX entry point
├── thumbgen.php      – On-the-fly thumbnail generator
├── vendor/           – Composer-managed dependencies (not committed)
├── composer.json     – Composer configuration
└── dzcp_community_beta.sql – Initial database schema
```

---

## Core Classes & Conventions

### `common` (inc/common.php)
- Central static helper class. All modules call `common::methodName(...)`.
- Bootstrapped once via `new common()` in `common.php`.
- Holds global static state: `$database`, `$sql`, `$smarty`, `$userip`, `$userid`, `$cache`, etc.
- Key methods: `data()`, `getUserIndex()`, `permission()`, `check_ip()`, `visitorIp()`, `sendMail()`, `error()`, `info()`, `nav()`, `pwd_encoder()`, `checkme()`, `isBanned()`, `get_files()`.

### `database` (inc/database.php)
- Final PDO wrapper supporting named configurations (`default`, `sessions`, `test`, etc.).
- **Method naming**: `select()`, `fetch()`, `insert()`, `update()`, `delete()`, `rows()`, `show()`, `create()`, `optimize()`, `query()`.
- `fetch($qry, $params, $field)` returns a single row; pass a field name as the third argument to get a scalar value.
- `rows($qry, $params)` returns the row count for a SELECT/SHOW query without returning data.
- SQL table names use the `{prefix_tablename}` placeholder syntax, resolved by `rep_prefix()`.
- Debug constants: `show_pdo_delete_debug`, `show_pdo_update_debug`, `show_pdo_insert_debug`, `show_pdo_select_debug` (defined in `debugger.php`).
- **Never** use raw string concatenation for user-supplied or time-derived values in queries – always use parameterized queries (`?` placeholders with a params array).

### `cache` (inc/cache.php)
- Extends `phpFastCache\CacheManager`.
- Three tiers: `file`, `memory` (APCu/APC/WinCache/Zend), `net` (Memcache/Memcached/Redis/Predis).
- Auto-selects the best available tier via `AutoGet/AutoSet/AutoExists/AutoDelete`.
- Dedicated methods per tier: `FileGet/FileSet`, `MemGet/MemSet`, `NetGet/NetSet`, `AutoMemGet/AutoMemSet` (net preferred over memory).
- TTL constants live as `const TIME_*` on the `Cache` class.

### `session` (inc/sessions.php)
- Custom PHP session save handler; backend selected via the `sessions_backend` constant (`memcache`, `apc`, `mysql`, `php`).
- `encode()`/`decode()` use the `Crypt` class (see below); method names are **lowercase**: `encrypt()` / `decrypt()`.
- SQL backend uses `$db->fetch()` (not `selectSingle`) and `$db->rows()` (not `select` + `rowCount`).
- `sql_gc()` uses parameterized queries for time-based comparisons.
- `mem_open()` returns `true` if already connected (idempotent open).
- `apc_read()` enters the lock section when `$this->_lockTimeout > 0`.
- `sql_open()` returns `true` on success (i.e., when `$this->db instanceOf database`).

### `Crypt` (inc/crypt.php)
- Simple XOR + encoding class.
- Modes: `Crypt::MODE_BIN`, `Crypt::MODE_B64`, `Crypt::MODE_HEX`.
- Public methods: **`encrypt($data)`** and **`decrypt($crypt)`** (all lowercase).
- Properties set via magic `__set('Key', ...)` and `__set('Mode', ...)`.
- **Note**: The `Hash` property does not exist on `Crypt`; set `Mode` to change encoding.

### `settings` (inc/settings.php)
- Extends `common`. Provides typed settings CRUD against the `{prefix_settings}` table.
- `get($key)`, `set($key, $value)`, `add(...)`, `remove($key)`, `load()`.

### `cookie` (inc/cookie.php)
- Static helper. Call `cookie::init($name)` first, then `get/put/save/clear/delete`.

### `notification` (inc/notification.php)
- Static notification queue. `add_error/add_success/add_notice/add_warning`.
- `get($index)` renders and clears queued notifications via Smarty.

### `DebugConsole` (inc/debugger.php)
- Static logging/debug class.
- `insert_error`, `insert_info`, `insert_successful`, `insert_warning`, `insert_log`, `sql_error_Exception`, `show_logs`, `save_log`, `wire_log`.
- Output only shown when `config::$view_error_reporting` is `true`.

---

## Database Conventions

- All table names use `{prefix_tablename}` in queries (resolved to actual prefix at runtime).
- Always use parameterized queries: `$db->fetch("SELECT … WHERE id = ?", [$id])`.
- Use `$db->rows(...)` to count matching rows without fetching data.
- Use `$db->fetch(...)` to get a single row or single field.
- Use `$db->select(...)` to get all matching rows as an associative array.
- The `database::query()` method is for raw DDL/admin statements only (no params support).

---

## IP & Security Conventions

- `common::$userip` is always an array with keys `'v4'` (IPv4) and `'v6'` (IPv6).
- Always access `self::$userip['v4']` or `self::$userip['v6']` – never pass the whole array to `isIP()`.
- `isIP(string $ip, bool $v6 = false)` validates an IP address string.
- `check_ip()` is called during bootstrap to validate and potentially block the current visitor.

---

## Template Engine

- **Smarty** is used for all HTML output.
- Template files live under `inc/_templates_/{theme}/`.
- Use `common::getSmarty()` to obtain the singleton Smarty instance.
- Compiled templates are stored in `inc/_templates_c/` (runtime, not committed).

---

## Caching Guidelines

- Cache keys should be deterministic strings (e.g., `md5('prefix_' . $id)`).
- Use `config::$use_system_cache` guard before writing to the cache.
- Use appropriate TTL constants from `Cache::TIME_*`.
- `AutoMem*` methods prefer network cache (Redis/Memcache) over local memory.

---

## Known Patterns & Anti-Patterns

| Anti-Pattern | Correct Pattern |
|---|---|
| `$db->selectSingle(...)` | `$db->fetch(...)` |
| `$db->select(...); $db->rowCount()` just to count | `$db->rows(...)` |
| `$crypt->Encrypt(...)` | `$crypt->encrypt(...)` |
| `$crypt->Decrypt(...)` | `$crypt->decrypt(...)` |
| Concatenating integer time into SQL string | Parameterized query with `?` placeholder |
| `show_pdo_delete_debug \|\| show_pdo_delete_debug \|\| ...` (duplicate) | `show_pdo_delete_debug \|\| show_pdo_update_debug \|\| show_pdo_insert_debug \|\| show_pdo_select_debug` |
| Reusing a `bool` parameter variable as an array | Introduce a new `$results` variable |

---

## Running / Testing

This project does not currently have an automated test suite (PHPUnit or similar). To test:

1. Install PHP 7.4+ and a MySQL-compatible database.
2. Run `php composer install` in the project root to install dependencies.
3. Import `dzcp_community_beta.sql` into your database.
4. Configure `inc/configs/config.php` with your database credentials.
5. Serve the project via a web server (Apache/Nginx) pointing to the project root.
6. Enable `config::$view_error_reporting = true` and `config::$debug_dzcp_handler = true` during development to surface errors via `DebugConsole`.

---

## Commit & PR Guidelines

- Make **surgical, minimal changes** – do not reformat unrelated code.
- Follow the existing code style (spaces, brace placement, PHPDoc comments).
- Parameterize all SQL queries.
- Test session backends locally or via integration environment before merging session-handler changes.
- Reference the relevant bug/issue in the PR description.
