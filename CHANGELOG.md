# Changelog

## [Unreleased] — PHP Compatibility & Bugfix Update

This update focuses on getting the MyBB Merge System running on modern PHP versions (7.0 through 8.3) and fixing long-standing data integrity bugs. No new features or board converters have been added yet.

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
- **bbPress**: restricted to MySQL (`$supported_databases = array("mysql")`)
- **FluxBB**: restricted to MySQL
- **IPB3**: restricted to MySQL
- **IPB4**: restricted to MySQL
- **Vanilla**: uses `GROUP_CONCAT` in users module (declares full database support)
- **WBB3**: uses `GROUP_CONCAT` in users module (declares full database support)
- **WBB4**: uses `GROUP_CONCAT` in users module (declares full database support)

The **Vanilla**, **WBB3**, and **WBB4** converters should either be restricted to MySQL or have their `GROUP_CONCAT` queries rewritten.

#### Other
- **`set_error_notice_in_progress()`** in `resources/output.php` sets a property that is never read. The function is called from avatar and attachment modules but the value is effectively discarded. This is harmless but dead code.
- **`passwordconvert` column cleanup**: The `loginconvert.php` plugin relies on a temporary `passwordconvert` column added to MyBB's `users` table during conversion. If the merge system is interrupted or old conversion artifacts remain, this column may need manual cleanup.
- **Converters marked "not supported anymore"**: Several password hash types are commented as unsupported in `loginconvert.php` (`vb3`, `ipb2`, `smf`). Users converting from very old vBulletin 3, IPB 2, or SMF 1.x installations may have login issues post-migration.
