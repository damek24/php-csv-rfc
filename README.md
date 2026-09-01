# A modern object-oriented CSV API for PHP — analysis for a possible RFC

php-src state: `master` (8.6.0-dev), branch `rfc-csv`, analysis date: 2026-09-01.
Scope: the existing CSV implementation, dependencies on streams/SPL, compatibility issues,
design patterns of recent extensions, three public API variants with trade-offs, and the risks
of such an RFC on internals.

---

## 1. Summary

- The PHP 8.6 deprecation (commit `ac3ff655ea`, PR php/php-src#23437, RFC
  `deprecations_php_8_6#deprecate_splfileobject_csv_methods`, section author: Muhammed Arshid KV)
  covers **four methods** of `SplFileObject` (`fgetcsv`, `fputcsv`, `setCsvControl`, `getCsvControl`).
  The `SplFileObject::READ_CSV` flag was **not** deprecated.
- The RFC names no migration path. `SplFileObject` does not expose its stream, so a user of
  `SplFileObject::fputcsv()` has no drop-in replacement (`fputcsv()` requires a `resource`).
- All of CSV in core is a single PHPAPI parser, `php_fgetcsv()` + `php_fputcsv()` in
  `ext/standard/file.c`, shared by `fgetcsv()`, `str_getcsv()` and SPL. It is line-oriented,
  locale-dependent, and preserves several historical quirks.
- Prior art exists: Gina Banyard's `girgias/csv` extension (PECL/PIE, BSD-3, strict RFC 4180,
  no escape character, multibyte delimiter/enclosure/EOL, no stream I/O) with a stated plan for an
  RFC and a deprecation timeline for `fgetcsv()`/`fputcsv()`/`str_getcsv()`.
- Recommendation: variant A (Reader/Writer over `php_stream`, reusing the existing parser, with a
  `Dialect` compatible with `girgias/csv`), positioned as "closing the gap left by the 8.6
  deprecation", designed in coordination with the author of `girgias/csv`.

---

## 2. Map of the existing implementation

| Element | Location | Notes |
|---|---|---|
| Parser | `ext/standard/file.c:1911` `php_fgetcsv()` (~250 lines, `goto`-based state machine) | PHPAPI, declared in `ext/standard/file.h:50` |
| Writer | `ext/standard/file.c:1754` `php_fputcsv()` | `smart_str` → a single `php_stream_write()` |
| `$escape` handling | `ext/standard/file.c:1671` `php_csv_handle_escape_argument()` | 8.4 deprecation; `PHP_CSV_NO_ESCAPE = EOF`, `PHP_CSV_ESCAPE_ERROR = -500` (`file.h:45-46`) |
| Empty line → `[null]` | `ext/standard/file.c:1902` `php_bc_fgetcsv_empty_line()` | a dedicated PHPAPI function that exists only to preserve the quirk |
| Trailing whitespace | `ext/standard/file.c:1633` `php_fgetcsv_lookup_trailing_spaces()` | uses `php_mblen` |
| `fputcsv()` / `fgetcsv()` | `ext/standard/file.c:1696`, `:1819` | `PHP_Z_PARAM_STREAM` — resources only (`main/php_streams.h:303,322`) |
| `str_getcsv()` | `ext/standard/string.c:5511` | calls `php_fgetcsv(NULL, …)`; the parser has a special `stream == NULL` path |
| SPL read | `ext/spl/spl_directory.c:1853` `spl_filesystem_file_read_csv()` | duplicates the current line because the parser frees its buffer |
| SPL methods | `ext/spl/spl_directory.c:2250-2395` | `spl_csv_enclosure_param_handling()` at `:2232` |
| SPL state | `ext/spl/spl_directory.h:80-83` | `char delimiter, enclosure; int escape; bool is_escape_default` |
| SPL flag | `ext/spl/spl_directory.h:92` `SPL_FILE_OBJECT_READ_CSV 0x8` | stub `ext/spl/spl_directory.stub.php:221` — no `#[\Deprecated]` |
| Stubs | `ext/standard/basic_functions.stub.php:2586,2889,2896`; `ext/spl/spl_directory.stub.php:243-255` | `array\|false`, `$stream` untyped |
| Tests | 52 in `ext/standard/tests/file`, 3 in `ext/standard/tests/strings`, 33 in `ext/spl/tests/SplFileObject/csv`, 1 in `ext/phar/tests` | ~89 `.phpt` files usable as a compatibility oracle |

Parser change history (2019→): #8936 (HashTable instead of in/out zval), GH-11982, GH-12151,
GH-15653, buffer overflow with `\0` as delimiter, #15365 (ValueError in `str_getcsv`), #15569
("Make CSV deprecation less annoying" — `is_escape_default`), bug #66588 (SPL, premature EOF).

---

## 3. Dependencies on streams and SPL

1. **Line model.** The parser receives one line from `php_stream_get_line()`; when an enclosure
   spans a `\n`, it pulls further lines itself (`file.c:2005-2035`). Hence the `php_stream *`
   parameter and the `NULL` path for strings. It cannot be used as a push parser over an arbitrary
   buffer without rework.
2. **EOL.** `php_stream_get_line()` recognises only `\n`; `\r` only with the deprecated
   `auto_detect_line_endings` (`main/streams/streams.c:191`, `:795-810`).
3. **Locale.** `php_mblen` = `mbrlen()` with `BG(mblen_state)` (`ext/standard/php_string.h:61`),
   plus `isspace()` when skipping whitespace before an enclosure (`file.c:1946`). Parsing results
   depend on `LC_CTYPE` (test `ext/standard/tests/file/bug72330.phpt`).
4. **SPL.** Keeps its own copy of the line (`current_line`) and result (`current_zval`); the
   interplay of `READ_CSV | SKIP_EMPTY | DROP_NEW_LINE | READ_AHEAD` (`is_line_empty()`,
   `spl_directory.c:1840`) is the source of historical bugs (gh8121, bug77024, gh13685).
5. **No access to the stream.** `SplFileObject` has no `getStream()`; migrating `fputcsv()` away
   from SPL requires a second `fopen()`. Files are opened via `php_stream_open_wrapper_ex()` with a
   context (`spl_directory.c:333`).

---

## 4. Baked-in quirks (compatibility issues)

1. Empty line → `[null]`.
2. Proprietary `$escape` mechanism (`\`) that keeps the escape character in the output. The 8.4
   RFC "Kill proprietary CSV escaping mechanism" (26:2) makes `""` the default in PHP 9.
3. `fputcsv()` quotes fields containing a space or tab (`file.c:1773-1774`) — more than RFC 4180
   requires; default EOL is `\n`, not `\r\n`.
4. Delimiter/enclosure are a single `char` — no multibyte support.
5. Non-string fields go through `zval_get_tmp_string()` (array → `"Array"` + warning).
6. Trailing whitespace/EOL is stripped from unquoted fields; leading whitespace before a quote is
   skipped.
7. An unterminated enclosure consumes the stream up to EOF; `str_getcsv()` takes a different path.
8. The `SplFileObject::fgetcsv()` signature "lies" about its defaults — the state from
   `setCsvControl()` wins over the literals in the stub; with named arguments this cannot be fixed
   without `?string` (PR #22160 — rejected by Girgias as "not an improvement"). This was the direct
   trigger for the deprecation.
9. `array|false` — `false` on EOF and `false` on a read error are indistinguishable.

---

## 5. Context on internals (as of 2026-09-01)

- The RFC section names no replacement; the vote closed on 2026-08-10; merged on 2026-08-25.
  The PR reviewer noted limited capacity for a full SPL review before beta2.
- **Ignace Nyamagana Butera** (league/csv), 2026-06-23: deprecating without "at least the start"
  of a replacement API is premature; first a better API, consensus, a clear migration, then
  deprecation.
- **Takuya Aramaki** (2026-08-07) and **Robert Humphries** (2026-08-15): inconsistency with
  `READ_CSV` (without `setCsvControl()` in PHP 9 the flag stays locked to its defaults); no
  migration path for `SplFileObject::fputcsv()`. No reply from the authors found.
- **Gina Banyard**, 2026-06-23: "If you provide me with text proposal … I'm happy to include it in
  the bulk RFC. I will however not be writing it myself."
- **`girgias/csv`** (GitLab, PECL 0.4.3 from 2025-02-22, BSD-3, PHP ≥ 8.0):

  ```php
  function Csv\array_to_row(array $fields, string $delimiter = ',', string $enclosure = '"', string $eolSequence = "\r\n"): string;
  function Csv\row_to_array(string $row, string $delimiter = ',', string $enclosure = '"', string $eolSequence = "\r\n"): array;
  function Csv\collection_to_buffer(iterable $collection, string $delimiter = ',', string $enclosure = '"', string $eolSequence = "\r\n"): string;
  function Csv\buffer_to_collection(string $buffer, string $delimiter = ',', string $enclosure = '"', string $eolSequence = "\r\n"): array;
  function Csv\buffer_to_collection_lax(...): array;
  final class Csv\LazyLaxCollection implements IteratorAggregate {
      public static function createFromBuffer(string $buffer, string $delimiter = ',', string $enclosure = '"', string $eolSequence = "\r\n"): LazyLaxCollection;
      public function getIterator(): \InternalIterator;
  }
  ```
  README: "The plan is to propose an RFC and add this extension to the core of PHP with a timeline
  on deprecating the non-compliant `str_getcsv()`, `fputcsv()`, and `fgetcsv()` functions."
- Other prior art: `csvtoolkit/FastCSV-ext` (Reader/Writer/Config, file paths rather than
  streams, MIT, PHP ≥ 8.2); `ajgl/csv-rfc` (userland, RFC 4180 drop-in).

---

## 6. Design patterns of recent extensions in php-src

Sources: `ext/uri` (8.5), `ext/random` (8.2), `BcMath\Number` (8.4), `Dom\*` (8.4).

- Top-level namespace (`Uri\`, `Random\`, `BcMath\`); stub with `@generate-class-entries`;
  `final` classes with `@strict-properties`; `final readonly` value objects
  (`Uri\Rfc3986\Uri`, `BcMath\Number`); enums instead of int flags (`Uri\UriComparisonMode`).
- Exception pair: `Uri\UriException extends \Exception` + `Uri\UriError extends \Error`
  (`ext/uri/php_uri.stub.php:9-17`).
- Named constructors: `Dom\HTMLDocument::createFromFile()/createFromString()`
  (`ext/dom/php_dom.stub.php:1677-1679`); `Uri\Rfc3986\Uri::parse(): ?static` next to a throwing
  `__construct` (`php_uri.stub.php:81`).
- Native iteration: `implements \IteratorAggregate` + a C-level `get_iterator` handler
  (`ext/dom/php_dom.c:1033`), return type `InternalIterator`.
- Object handlers: `create_object`, `free_obj`, `clone_obj` copied from std
  (`ext/uri/php_uri.c:1514-1548`).
- Always-enabled extension: `PHP_NEW_EXTENSION(uri, …, [no])` (`ext/uri/config.m4:42`);
  `PHP_INSTALL_HEADERS` for PHPAPI; an `EXTENSIONS` entry with a named maintainer
  (`uri`: Kocsis, Düsterhus; `spl`: maintainers listed only up to 2014).
- Size: `ext/uri/php_uri.c` 1619 lines + bundled uriparser; `ext/random/randomizer.c` 542;
  `ext/spl/spl_directory.c` 2718; `ext/standard/file.c` 2343.

---

## 7. Public API variants

Shared assumptions: `Csv\` namespace, no escape parameter (in line with PHP 9), a record is a
`list<string>`, `Csv\CsvException extends \Exception`, `Csv\CsvError extends \Error`.

### Variant A — `Csv\Reader` / `Csv\Writer` over `php_stream`, reusing the existing parser

```php
namespace Csv;

final readonly class Dialect
{
    public function __construct(
        public string $separator = ',',
        public string $enclosure = '"',
        public string $eol = "\n",
    ) {}
}

final class Reader implements \IteratorAggregate
{
    private function __construct() {}

    /** @param resource $stream */
    public static function fromStream($stream, ?Dialect $dialect = null): static {}
    /** @param resource|null $context */
    public static function fromFile(string $path, ?Dialect $dialect = null, $context = null): static {}
    public static function fromString(string $data, ?Dialect $dialect = null): static {}

    /** @return list<string>|null  null at EOF; CsvException on I/O error */
    public function read(): ?array {}
    /** @return \InternalIterator<int, list<string>> */
    public function getIterator(): \InternalIterator {}
    public function getDialect(): Dialect {}
}

final class Writer
{
    private function __construct() {}

    /** @param resource $stream */
    public static function toStream($stream, ?Dialect $dialect = null): static {}
    /** @param resource|null $context */
    public static function toFile(string $path, ?Dialect $dialect = null, $context = null): static {}

    /** @param list<string|\Stringable|int|float|bool|null> $fields  @return int bytes written */
    public function write(array $fields): int {}
    public function getDialect(): Dialect {}
}
```

Core: `php_fgetcsv()` / `php_fputcsv()` unchanged semantically; removing the locale dependence and
the `[null]` quirk needs a single flags parameter in the parser (new call site, existing callers
stay identical).

### Variant B — a full RFC 4180 `ext/csv` with a new parser

```php
namespace Csv;

enum Quoting { case Minimal; case All; case NonNumeric; case None; }

final readonly class Dialect
{
    public function __construct(
        public string $separator = ',',      // multibyte OK
        public string $enclosure = '"',      // multibyte OK
        public string $eol = "\r\n",         // multibyte OK
        public Quoting $quoting = Quoting::Minimal,
        public bool $strict = false,         // equal column count enforced → CsvException
    ) {}
    public static function rfc4180(): static {}
    public static function excel(): static {}
}

final class Reader implements \IteratorAggregate { /* as in A */ }
final class Writer { /* as in A */ }

// thin wrappers compatible with girgias/csv
function parse_row(string $row, ?Dialect $dialect = null): array {}
function format_row(array $fields, ?Dialect $dialect = null): string {}
```

A new buffer-based (not line-based) parser in C, without `mblen`/`isspace`, supporting
`\r`-only EOL and optionally BOM; a test suite written from scratch.

### Variant C — primitives only, no I/O

```php
namespace Csv;

final readonly class Dialect { /* as in B */ }

final class Parser
{
    public function __construct(Dialect $dialect = new Dialect()) {}
    public function feed(string $chunk): void {}
    public function finish(): void {}          // forces the last record to close
    /** @return \InternalIterator<int, list<string>> records completed since the last call */
    public function records(): \InternalIterator {}
}

final class Formatter
{
    public function __construct(Dialect $dialect = new Dialect()) {}
    /** @param list<string|\Stringable|int|float|bool|null> $fields */
    public function format(array $fields): string {}
}
```

Streams stay in userland (`fread`/`fwrite`, PSR-7, Amp). Libraries (league/csv) build I/O,
headers and mapping on top.

### Assessment

| Criterion | A: Reader/Writer + existing parser | B: full RFC 4180 | C: primitives |
|---|---|---|---|
| Migration from `SplFileObject` | best: identical semantics; `fromFile()` replaces `new SplFileObject`; `foreach` replaces `READ_CSV`; differences limited to `[null]`/escape | requires moving to strict RFC 4180 — files containing `\"` change meaning | worst: the user writes the I/O loop |
| Streams | native (`php_stream`, wrappers, context, `php://memory`) | same, own buffer → `\r`-only EOL possible | none; works with any source, including async |
| Typing | `?list<string>`, `readonly Dialect`, no `false`; `$stream` must stay untyped (`resource`) | same + enums; best | same; `feed(string)` fully typed, no `resource` |
| BC | none — new `Csv\` namespace (userland collision risk as with `Uri\`/`Random\`); `fgetcsv()` untouched | none in code; a `fgetcsv` deprecation plan (if included) is a major social BC break | none |
| Maintenance cost | ~600–900 lines of glue + tests; the parser already has ~89 tests; inherits its debt | ~1.5–2.5k lines; a new parser **next to** the old one for years | ~800–1.2k lines; smallest surface, no streams/SPL dependency |
| Chance on internals | highest (closes the deprecation gap, does not touch `fgetcsv`) | medium; high only with the `girgias/csv` author as co-author | medium; risk of "why in core if it does no I/O" |

---

## 8. Arguments that could sink the RFC

1. **"Userland does it better; PIE solves distribution"** — `league/csv` and `girgias/csv` exist;
   `ext/uri` passed because URIs are security-critical and needed by core.
2. **Two CSV implementations in core** — without a deprecation timeline for
   `fgetcsv`/`str_getcsv` it is double maintenance; with one, it mobilises the anti-BC-break camp.
3. **Conflict with an existing plan** — `girgias/csv` has its own design and a stated RFC
   intention; a competing OO API without its author's involvement may get "we already have a plan".
4. **Migration semantics** — keeping the quirks = "SplFileObject with a new paint job"; dropping
   them = "this is not a replacement".
5. **Scope bikeshedding** — headers, associative mapping, BOM/encoding, quoting modes, multibyte;
   every "no" in the RFC invites "then why is this in core".
6. **Maintainer** — `spl` in `EXTENSIONS` lists maintainers only up to 2014; the #23437 reviewer
   acknowledged limited capacity. An RFC without a named maintainer and a PR before the vote has
   poor odds.
7. **Timing** — 8.6 is past feature freeze; the target is 8.7/9.0, in parallel with the `$escape`
   default change.
8. **Performance** — "the same in PHP under JIT is fast enough".

---

## 9. Recommendation

Variant **A**, positioned as closing the gap left by the 8.6 deprecation, with three deliberate
departures from `fgetcsv()` (no escape, no `[null]`, exceptions instead of `false`), without
touching `fgetcsv`/`str_getcsv`. Preconditions:

1. Co-authorship or public endorsement by the author of `girgias/csv`; a `Dialect` semantically
   compatible with `girgias/csv`, so that variant B becomes a natural follow-up.
2. A separate, small RFC item deprecating `SplFileObject::READ_CSV` (fixing the inconsistency
   already raised on the list).
3. A ready PR reusing the existing ~89 tests as an oracle + new tests for `Csv\`.
4. A declared maintainer in `EXTENSIONS`.

Variant C as a fallback if internals rejects I/O in core.

---

## Sources

- RFC Deprecations 8.6: https://wiki.php.net/rfc/deprecations_php_8_6
- RFC Deprecations 8.4 (CSV escape): https://wiki.php.net/rfc/deprecations_php_8_4
- Kill proprietary CSV escaping (externals): https://externals.io/message/103268
- PR #23437: https://github.com/php/php-src/pull/23437
- PR #22160: https://github.com/php/php-src/pull/22160
- PR #15569: https://github.com/php/php-src/pull/15569
- internals thread, part 1: https://discourse.thephp.foundation/t/re-php-dev-rfc-deprecations-for-php-8-6/5636
- internals thread, part 2: https://discourse.thephp.foundation/t/php-dev-re-fwd-rfc-deprecations-for-php-8-6/5939
- news-web 131500 / 132225: https://news-web.php.net/php.internals/131500 , https://news-web.php.net/php.internals/132225
- Girgias/csv-php-extension: https://gitlab.com/Girgias/csv-php-extension
- PECL CSV: https://pecl.php.net/package/CSV ; Packagist: https://packagist.org/packages/girgias/csv
- csvtoolkit/FastCSV-ext: https://github.com/csvtoolkit/FastCSV-ext
- thephpleague/csv: https://github.com/thephpleague/csv
- RFC 4180: https://www.rfc-editor.org/rfc/rfc4180.html
