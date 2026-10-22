# php56-php2shell-patches

Fix for CVE-2026-109618 and CVE-2026-109619 ("PHP2Shell") for PHP 5.6.40.

Published as a stop-gap measure so people have some breathing room to port their codebases to PHP version 8.

PHP 5.6 reached end of life in January 2019 and gets no official fixes. This is a port of the upstream PHP 7.4 fix (php-src commit `712e60f9ed`, "Avoid state corruption via bailout") to 5.6.40. It is not an official PHP release.

## Applying

The patch applies to the official `php-5.6.40.tar.xz` release and to the `php-5.6.40` tag of php-src:

    cd php-5.6.40
    patch -p1 < ../php-5.6.40-php2shell.patch

Then configure and build as usual.

## What it changes

CVE-2026-109619:

- `ext/session/session.c`: the session upload progress handler is disabled, as in the upstream 7.4/8.0/8.1 fixes.
- `main/SAPI.c`, `main/main.c`: request info pointers are set to NULL after being freed; the auto globals and `modules_activated` are reset at request shutdown.
- `Zend/zend_execute_API.c`: user error/exception handlers and their stacks, the error handling mode, the current exception, the autoload stack and the object store are reset in `shutdown_executor()`.
- `Zend/zend_llist.c`: `zend_llist_destroy()` resets `head` and `tail`.

CVE-2026-109618:

- `Zend/zend_alloc.c`, `Zend/zend_alloc.h`: new `zend_mm_safe_error_ex()`. Memory limit and allocation size overflow errors print the fatal error to stderr and end the process with `_exit(1)` instead of bailing out.
- `Zend/zend_alloc.c`, `Zend/zend_strtod.c`, `Zend/zend_operators.c`, `Zend/zend_API.c`, `Zend/zend_API.h`: the overflow checks in `safe_address`, `_safe_emalloc_string`, `Balloc`, string concatenation, `ZVAL_STRING` and the `add_*string*` functions call `zend_mm_safe_error_ex()`.
- `Zend/zend_API.c`, `main/php_variables.c`: request variable arrays start at the minimum table size (`array_init_small()`).
- `main/rfc1867.c`: only `Content-Disposition` and `Content-Type` part headers are kept, duplicate headers are ignored, and the number of parts is limited by the new `max_multipart_body_parts` setting.

## Behavior changes

- Session upload progress (`session.upload_progress.*`) no longer reports anything.
- New INI setting `max_multipart_body_parts`. The default, -1, means `max_input_vars` + `max_file_uploads`. Parts past the limit are dropped with a warning.
- On memory limit exhaustion the error is still printed, but the process exits with status 1 right away and shutdown functions do not run. Under mod_php the Apache child exits and the connection is closed without a response; Apache starts a new child.

## Testing

- Zend/tests, tests/basic, tests/lang, ext/session/tests and ext/standard/tests/general_functions (2392 tests), patched build against unpatched build. Differences: the eight `ext/session/tests/rfc1867*` upload progress tests fail, and `tests/lang/bug45392` prints one extra blank line.
- Built with the full Slackware 14.2 configure set (ZTS, apache2handler, CLI, CGI, FPM, ext/intl) using gcc 5.5.
- Uploads, the part limit, sessions and the memory limit path checked through php-cgi and mod_php.

## License

The patch is derived from php-src and is distributed under the PHP License, version 3.01. See `LICENSE`.
