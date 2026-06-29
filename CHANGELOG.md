# Changelog

## [Unreleased] — PHP Compatibility & Bugfix Update

This update focuses on getting the MyBB Merge System running on modern PHP versions (7.0 through 8.3), fixing long-standing data integrity bugs, removing converters for defunct forum software, and adding support for Invision Community 5 (IPS5).

---

### New Converter: Invision Community 5 (IPS5)

- **New converter skeleton for IPS5** (`boards/ipb5.php` + 13 modules in `boards/ipb5/`). Built from the IPS4 converter as a template.
- **Status: Untested** — All column names carry `TODO: verify` markers pending access to a real IPS5 database for schema verification. Key areas needing verification:
  - Table names (`core_members`, `core_groups`, `forums_forums`, etc.)
  - Column names (password hash storage, member fields, language system)
  - Permission system changes
  - Settings storage mechanism (Laravel config vs `core_sys_conf_settings`)
  - BBCode/HTML output format changes
- **Password handling**: IPS5 uses Laravel's password hashing (bcrypt `$2y$` or argon2). Added `check_ipb5()` function using `password_verify()` and `ipb5` password type to `loginconvert.php`.

### New Converter: Discourse

- **New converter for Discourse** (`boards/discourse.php` + 12 modules in `boards/discourse/`). Discourse is a modern open-source forum platform (Ruby on Rails, PostgreSQL).
- **Modules**: users, usergroups (trust levels + custom groups), forums (categories), threads, posts, polls, pollvotes, privatemessages, moderators (admin/mod boolean flags), avatars (custom + Gravatar), attachments (post_uploads), settings (site_settings), bbcode_parser (cooked HTML→BBcode).
- **Key features**:
  - Maps Discourse trust levels (0-4) to MyBB groups; TL3/TL4 get custom imported groups
  - Admin/moderator boolean flags mapped to MyBB admin/mod groups
  - Password hashing uses `password_verify()` for Ruby's `has_secure_password` bcrypt format
  - Polls module handles optional Discourse poll plugin (silently skips if polls table missing)
  - Post content converts Discourse "cooked" HTML to BBcode via custom parser
  - PostgreSQL-only (`$supported_databases = array("pgsql")`)
- Added `check_discourse()` and `discourse` password type to `loginconvert.php`

### Requirements Check Enhancements

- **MySQL engine detection**: Added check for MyISAM tables in the MyBB database during the requirements phase. Warns if MyISAM tables are found and recommends converting to InnoDB before merging.
- **MySQL version check**: Detects MySQL version and warns about MySQL 5.7 (EOL) and MySQL 8.4+ (compatibility concerns due to removed legacy features).
- **MariaDB recommendation**: Displays a recommendation to use MariaDB over MySQL when MariaDB is detected.

---

### Deprecated & Removed Converters

The following board converters have been removed to reduce technical debt and maintenance burden:

- **PunBB** — Project merged into FluxBB in 2008; FluxBB itself is discontinued (2015)
- **FluxBB** — Discontinued in 2015
- **SMF 1.x** — Released 2006-2011, replaced by SMF 2.0 in 2011
- **vBulletin 3** — Released 2004-2010, replaced by vBulletin 4 in 2010
- **WoltLab Burning Board 3 (WBB3)** — Released 2007, superseded by WBB4 (2013) and WBB5+
- **Invision Power Board 3 (IPB3)** — Released 2009, replaced by IPS 4 in 2015
- **bbPress** — Heavily tied to WordPress internals; converter design doesn't align with modern bbPress versions
- **Vanilla** — Upstream API has changed significantly; converter is not compatible with current Vanilla Forums releases

> **Note:** Password hash types for these converters (`ipb3`, `punbb`, `wbb3`, `fluxbb`, `bbpress`, `vanilla`) remain in `loginconvert.php` so that users who merged in the past can still log in.

---

### PHP Compatibility

#### PHP 8.0 Fatal Errors (Fixed)
- Replaced all 5 `create_function()` calls with anonymous functions. `create_function()` was removed in PHP 8.0 and would cause fatal errors.
  - `resources/functions.php` — `utf8_unhtmlentities()` (2 calls)
  - `boards/vbulletin3/privatemessages.php` — user array encoding
  - `boards/vbulletin4/privatemessages.php` — user array encoding
  - `boards/vbulletin5/privatemessages.php` — user array encoding

#### PHP 8.2 Deprecation Warnings (Preemptively Fixed)
- Added `merge_utf8_encode()` and `merge_utf8_decode()` wrapper functions in `resources/functions.php` with a fallback chain: `mb_convert_encoding` → `iconv` → native `@utf8_encode`/`@utf8_decode` → return as-is.
- Replaced all 10 native `utf8_encode()`/`utf8_decode()` calls across 6 files:
  - `resources/functions.php` — `encode_to_utf8()` helper
  - `loginconvert.php` — password recheck logic (2 calls)
  - `boards/vbulletin3/privatemessages.php` — serialized user array
  - `boards/vbulletin4/privatemessages.php` — serialized user array
  - `boards/vbulletin5/privatemessages.php` — serialized user array
  - `boards/ipb3/polls.php` — serialized poll choices

#### PHP 8.x Warnings (Fixed)
- **Cache handler missing `isset()` guards**: Added guards to 7 ID lookup methods (`fid()`, `fid_f()`, `fid_c()`, `tid()`, `gid()`, `aid()`, `pid()`) in `resources/class_cache_handler.php`. These methods would emit `Warning: Undefined array key` on PHP 8+ when a foreign key referenced a non-imported record. They now return `0` for missing entries, matching the behavior of `uid()`, `pollid()`, and `vid()`.
- **Undefined array keys in debug backtrace**: Changed `!$call['file']` → `empty($call['file'])` (3 checks) in `resources/class_debug.php` to avoid warnings when backtrace entries lack `file`, `line`, or `class` keys.
- **Undefined `$version_info[1]`**: Added `isset()` guard in `resources/output.php` board list parser. If the `$bbname` regex fails to match, `$version_info[1]` was previously accessed without checking.
- **`inet_ntop()` not available on all platforms**: Added `function_exists('inet_ntop')` guard in `resources/class_debug.php`. The function may not be available on Windows or minimal PHP installations.

#### Dead Code Removed
- Removed `register_globals` workaround (`$config_copy` block, 13 lines) from `index.php`. `register_globals` was removed in PHP 5.4.
- Removed `function_exists('mysql_connect')` database option block from `resources/output.php`. The `ext/mysql` extension was removed in PHP 7.0. MySQLi and PDO options remain.
- Removed outdated `=&` reference assignments from 6 object assignments across 3 files (`class_converter.php`, `class_converter_module.php`, `class_error.php`). Objects are passed by reference automatically in PHP 5+. Array reference assignments (`=&$this->settings`, `=&$this->trackers`) were intentionally preserved as they share mutable state between objects.

---

### Data Integrity Bug Fixes

- **phpBB3 `hideemail` inverted** (`boards/phpbb3/users.php`): phpBB's `user_allow_viewemail` = 1 means "show email", but MyBB's `hideemail` = 1 means "hide email". The value was previously copied directly without inverting. Now uses `int_to_01()` to correctly invert the mapping.
- **Typo `'lifed'` → `'lifted'` in bans module** (`resources/modules/bans.php`): The `lifted` field was misspelled as `lifed` in the `$integer_fields` array, causing it to be treated as a string and bypass integer validation. This affected ban expiry tracking for all board converters.
- **vBulletin3 & vBulletin4 timezone not set** (`boards/vbulletin3/users.php`, `boards/vbulletin4/users.php`): `$insert_data['timezone']` was used in a `str_replace()` call before being populated from the source data. Now correctly reads from `$data['timezoneoffset']` first.
- **`$this->trackers` → `$module->trackers` bug** (`resources/output.php`): An acknowledged TODO bug where the wrong object's trackers property was referenced in `print_per_screen_page()`. This caused the tracker reset to always use `0` instead of the module's actual tracker value.
- **XenForo `postnum_column` pointed to wrong column** (`boards/xenforo/users.php`): Was `'posts'` with a TODO comment. XenForo's actual user message count column is `message_count`.

---

### Converter Logic Improvements

- **XenForo 1 authentication class variants** (`boards/xenforo/users.php`): Added support for `XF:Core` and `XF:Core12` auth scheme class prefixes used in XenForo 1.5+. Previously only the `XenForo_Authentication_` prefix was recognized. Password conversion would silently fail for XenForo 1.5+ users if the `XF:` prefix was used.
- **phpBB3 thread visibility fallback** (`boards/phpbb3/threads.php`): Added an `else` clause that defaults `visible` to `1` when neither `topic_approved` (phpBB 3.0) nor `topic_visibility` (phpBB 3.1+) column is detected. Prevents threads from becoming invisible if future phpBB versions change column names.
- **phpBB3 poll count query** (`boards/phpbb3/polls.php`): Replaced MySQL-specific `COUNT(DISTINCT topic_id)` with a portable subquery `SELECT COUNT(*) FROM (SELECT DISTINCT topic_id FROM ...)`. Works across MySQL, PostgreSQL, and SQLite.
- **phpBB3 & SMF2 now restricted to MySQL only** (`boards/phpbb3.php`, `boards/smf2.php`): Added `$supported_databases = array("mysql")` to both converters. They previously claimed to support PostgreSQL and SQLite but used MySQL-specific `GROUP_CONCAT()` in user and forum permission queries.

---

### UI Improvements

- **Updated DOCTYPE to HTML5** (`resources/output.php`, `language/global.lang.php`): Replaced XHTML 1.0 Transitional DOCTYPE and `xmlns` attribute with standard HTML5 `<!DOCTYPE html>`.

---

### CI Updates

- **Expanded PHP version matrix** (`.github/workflows/php-syntax-check.yml`): Added PHP 8.1, 8.2, and 8.3 to the syntax check CI workflow.

---

### Known Issues & Potential Converter Concerns

The following issues were identified during analysis but **not yet addressed**. They should be reviewed before relying on the affected converters in production:

#### Partially Broken Converters
- **XenForo 1 (`boards/xenforo/`)**: The user converter maps `message_count` for post counts but does not populate several MyBB user fields: `regip`, `lastip`, `hideemail`, `invisible`, `pmnotify`, `pmnotice`, `referrer`, `threadnum`. User privacy and notification preferences may not transfer correctly.
- **phpBB3 (`boards/phpbb3/`)**: The user converter's timezone handling for phpBB 3.0 uses string replacement of `.0`/`.00` which is fragile. The thread converter's version detection (checking for `topic_approved` vs `topic_visibility` column presence) works for known versions but could break with schema changes.

#### Missing Field Mappings (Multiple Converters)
Several converters are missing mappings for MyBB fields that exist in the source software. Notable gaps across multiple converters:
- `lastip` — often not mapped (XenForo stores IPs in a separate table; other converters may simply omit it)
- `threadnum` — rarely populated even when the source software tracks it
- `displaygroup` — SMF2 stores post-count-based groups that could map here but doesn't
- `language` / `style` — often not mapped from source user preferences

#### Cross-Database Limitations
Several converters use MySQL-specific SQL and are not tested with PostgreSQL or SQLite:
- **IPB4**: restricted to MySQL
- **WBB4**: uses `GROUP_CONCAT` in users module (declares full database support)

The **WBB4** converter should either be restricted to MySQL or have its `GROUP_CONCAT` queries rewritten.

#### Other
- **`set_error_notice_in_progress()`** in `resources/output.php` sets a property that is never read. The function is called from avatar and attachment modules but the value is effectively discarded. This is harmless but dead code.
- **`passwordconvert` column cleanup**: The `loginconvert.php` plugin relies on a temporary `passwordconvert` column added to MyBB's `users` table during conversion. If the merge system is interrupted or old conversion artifacts remain, this column may need manual cleanup.
- **Converters marked "not supported anymore"**: Several password hash types are commented as unsupported in `loginconvert.php` (`vb3`, `ipb2`, `smf`). Users converting from very old vBulletin 3, IPB 2, or SMF 1.x installations may have login issues post-migration.
- **IPS5 Converter (`boards/ipb5/`)**: This is an untested skeleton built from the IPS4 converter. Column names and table structures are unverified against a real IPS5 database. The converter will appear in the board list but should be tested thoroughly before production use. Contributions from users with access to IPS5 are welcome.
- **Discourse Converter (`boards/discourse/`)**: New converter for Discourse. Tested against standard Discourse schema but may need adjustments for custom configurations, plugins, or older Discourse versions. Discourse uses PostgreSQL exclusively — ensure your PHP installation has PDO PostgreSQL support.
