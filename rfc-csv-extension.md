# PHP RFC: CSV Extension (ext/csv)

- Version: 0.9 (draft)
- Date: 2026-09-23
- Author: Damian Jóźwiak <damian.jozwiak.lodz@gmail.com>
- Based on: the `girgias/csv` extension by Gina Peter Banyard <girgias@php.net> (BSD-3-Clause)
- Status: Draft
- Target Version: next PHP minor release
- Implementation: https://github.com/damek24/php-csv-rfc
- First Published at: http://wiki.php.net/rfc/csv_extension

## Introduction

PHP's built-in CSV tooling (`fgetcsv()`, `fputcsv()`, `str_getcsv()` and the CSV methods of
`SplFileObject`) predates RFC 4180 adoption in the wider ecosystem and carries behaviour that
cannot be fixed without breaking compatibility:

- A proprietary `$escape` mechanism incompatible with every other CSV implementation. Relying on
  its default was deprecated in PHP 8.4 ("Kill proprietary CSV escaping mechanism") and the
  default becomes `""` in PHP 9.0.
- Locale-dependent parsing: the parser consults `LC_CTYPE` through `mblen()`/`isspace()`, so the
  same file can parse differently depending on `setlocale()`.
- Single-byte delimiters and enclosures only.
- An empty line parses as `[null]`; trailing whitespace of unquoted fields is silently stripped;
  fields containing a space are quoted on output although RFC 4180 does not require it.
- There is no inverse of `str_getcsv()`; writing a CSV string requires a `php://memory` stream.

PHP 8.6 deprecated `SplFileObject::fgetcsv()`, `SplFileObject::fputcsv()`,
`SplFileObject::setCsvControl()` and `SplFileObject::getCsvControl()`
(RFC: deprecations_php_8_6) **without providing a replacement**. Two gaps were raised on the
mailing list during that discussion and remain open:

1. `SplFileObject::fputcsv()` has no drop-in replacement: the procedural `fputcsv()` requires a
   stream resource, and `SplFileObject` does not expose its underlying stream.
2. `SplFileObject::READ_CSV` was not deprecated, yet `setCsvControl()`, the only way to
   configure the dialect it uses, was.

This RFC proposes to close those gaps by adding a small, RFC 4180 compliant CSV extension to the
core, based on the existing `girgias/csv` extension, extended with file/stream support.

## Proposal

Add a new, always-enabled extension `ext/csv` providing six functions and one class in the `Csv\`
namespace. All of them follow RFC 4180: fields containing the delimiter, the enclosure, CR, LF or
the EOL sequence are enclosed; enclosures inside enclosed fields are escaped by doubling; there is
**no escape character**. Delimiters, enclosures and EOL sequences may be multibyte, and parsing is
binary safe and locale-independent.

### String and row conversion

```php
namespace Csv;

function array_to_row(array $fields, string $delimiter = ',', string $enclosure = '"', string $eolSequence = "\r\n"): string {}

function row_to_array(string $row, string $delimiter = ',', string $enclosure = '"', string $eolSequence = "\r\n"): array {}
```

`array_to_row()` formats one row (the inverse of `str_getcsv()`, which PHP currently lacks);
`row_to_array()` parses one row.

### Collection conversion

```php
function collection_to_buffer(iterable $collection, string $delimiter = ',', string $enclosure = '"', string $eolSequence = "\r\n"): string {}

function buffer_to_collection(string $buffer, string $delimiter = ',', string $enclosure = '"', string $eolSequence = "\r\n"): array {}

function buffer_to_collection_lax(string $buffer, string $delimiter = ',', string $enclosure = '"', string $eolSequence = "\r\n"): array {}
```

`collection_to_buffer()` accepts any `iterable` of arrays (including generators) and checks that
every row has the same number of fields, throwing a `ValueError` otherwise, as does
`buffer_to_collection()` when parsing. The `_lax` variant permits rows of varying width.

### File and stream handling

```php
function collection_to_file(string $file, iterable $collection, string $delimiter = ',', string $enclosure = '"', string $eolSequence = "\r\n"): void {}
```

`collection_to_file()` writes rows one at a time through the streams layer, so a generator-backed
collection of any size is written in constant memory. `$file` accepts anything the streams layer
accepts (paths, `php://`, `compress.zlib://`, user wrappers). Failures to open, write or finish
writing (e.g. a failing flush of a compressed stream on close) throw.

```php
/**
 * @not-serializable
 * @strict-properties
 */
final class LazyLaxCollection implements \IteratorAggregate
{
    private function __construct() {}

    public function getIterator(): \InternalIterator {}

    public static function createFromBuffer(string $buffer, string $delimiter = ',', string $enclosure = '"', string $eolSequence = "\r\n"): LazyLaxCollection {}

    public static function createFromFile(string $file, string $delimiter = ',', string $enclosure = '"', string $eolSequence = "\r\n"): LazyLaxCollection {}
}
```

`LazyLaxCollection` iterates rows lazily. `createFromFile()` opens and holds a *private* stream:
rows are read in chunks with an enclosure-aware scanner (an EOL sequence inside an enclosed field
does not terminate a row), and the internal window buffer is compacted as rows are consumed, so
memory usage is proportional to the longest row rather than to the file. The stream is owned
exclusively by the object (it is not registered as a userland-visible resource) and is closed
when the object is destroyed. Iteration can be restarted (`foreach` twice), including after an
early `break`, as long as the underlying stream is seekable.

### Error handling

- `ValueError` for invalid dialect arguments (empty delimiter/enclosure/EOL, delimiter identical
  to enclosure or to the EOL sequence) and for inconsistent row widths in the strict functions.
- `TypeError` when a collection element is not an array, or a field is not stringable.
- `Error` for I/O failures (open, read, write, flush-on-close) and for rewinding a non-seekable
  stream.
- No function returns `false`; end of iteration is expressed by the iterator protocol.

### Migration from the deprecated SplFileObject API

| Deprecated (8.6)                              | Replacement                                        |
| --------------------------------------------- | -------------------------------------------------- |
| `new SplFileObject($f)` + `READ_CSV` + flags  | `Csv\LazyLaxCollection::createFromFile($f, ...)`   |
| `SplFileObject::fgetcsv()` in a loop          | `foreach (LazyLaxCollection::createFromFile(...))` |
| `SplFileObject::fputcsv()` in a loop          | `Csv\collection_to_file($f, $rows, ...)`           |
| `SplFileObject::setCsvControl()`              | dialect arguments of the calls above               |
| `str_getcsv()` (no inverse)                   | `Csv\row_to_array()` / `Csv\array_to_row()`        |

Note the deliberate semantic differences from the legacy API: no escape character (in line with
the PHP 9.0 default), no `[null]` for empty lines, no locale dependence, no trailing-whitespace
stripping, exceptions instead of `false`, and `"\r\n"` as the default EOL sequence as specified by
RFC 4180.

## Backward Incompatible Changes

None. No existing function, class or constant is touched. The `Csv\` top-level namespace becomes
reserved for this extension, following the policy on namespaces in bundled extensions; userland
code declaring symbols in `Csv\` may conflict, as with the `Uri\` and `Random\` namespaces before
it.

## Proposed PHP Version(s)

Next minor release of PHP.

## RFC Impact

- **To SAPIs:** none beyond a new always-enabled extension.
- **To Existing Extensions:** none. `ext/standard` and `ext/spl` are unchanged.
- **To Opcache:** none.
- **New Constants:** none.
- **php.ini Defaults:** none.

## Open Issues

- Whether the extension should be disableable at build time. The draft implementation currently
  supports `--disable-csv`; the proposal is to make it always enabled, like `ext/uri` and
  `ext/random`, since a migration target for a deprecated core API must be reliably present.
- Naming of `LazyLaxCollection` and whether a strict lazy variant should ship in the first
  version.
- Whether `SplFileObject::READ_CSV` should be deprecated in a separate follow-up RFC once this
  replacement exists (the author's intention).

## Unaffected PHP Functionality

`fgetcsv()`, `fputcsv()`, `str_getcsv()` and the deprecated `SplFileObject` CSV methods keep their
current behaviour. This RFC neither changes nor deprecates them.

## Future Scope

- A deprecation timeline for the non-compliant `str_getcsv()`, `fputcsv()` and `fgetcsv()`
  functions, as previously outlined by the `girgias/csv` project. Deliberately **not** part of
  this RFC.
- Deprecation of `SplFileObject::READ_CSV`, resolving the inconsistency left by the 8.6
  deprecations.
- Header handling (mapping rows to associative arrays), encoding/BOM handling, and further
  convenience APIs, which can be built in userland on top of the provided primitives.

## Proposed Voting Choices

Add the CSV extension to the core as described? Yes / No (2/3 majority required).

## Patches and Tests

Draft implementation targeting php-src `master`: `ext/csv`: a port of `girgias/csv` 0.6.0
(BSD-3-Clause, copyright Gina Peter Banyard and Contributors) with the new `collection_to_file()`
and `LazyLaxCollection::createFromFile()` APIs; 75 phpt tests, including regression tests for
sparse collections, multibyte dialect tokens at read-chunk boundaries, iterator cleanup on error
paths, and stream lifecycle (private stream ownership, flush-on-close failures).

https://github.com/damek24/php-csv-rfc

## Implementation

After the project is implemented, this section should contain:

1. the version(s) it was merged into
2. a link to the git commit(s)
3. a link to the PHP manual entry for the feature

## References

- RFC 4180: https://www.rfc-editor.org/rfc/rfc4180
- `girgias/csv` extension: https://gitlab.com/Girgias/csv-php-extension (PECL/PIE: `girgias/csv`)
- Deprecations for PHP 8.6 (SplFileObject CSV methods): https://wiki.php.net/rfc/deprecations_php_8_6
- Deprecations for PHP 8.4 (proprietary CSV escaping): https://wiki.php.net/rfc/deprecations_php_8_4
- Mailing list discussion of the 8.6 deprecation and its migration gaps:
  https://discourse.thephp.foundation/t/re-php-dev-rfc-deprecations-for-php-8-6/5636 and
  https://discourse.thephp.foundation/t/php-dev-re-fwd-rfc-deprecations-for-php-8-6/5939

## Rejected Features

- **An object-oriented Reader/Writer API next to the functions.** Providing both an OO and a
  procedural API for the same functionality adds complexity without benefit, particularly when
  the OO surface would consist of static methods only. The single `LazyLaxCollection` class
  exists because lazy iteration requires holding state.
- **Compatibility with the legacy `fgetcsv()` dialect** (escape character, `[null]` rows, locale
  dependence). Migrating code is expected to adopt the standard-compliant behaviour rather than
  carry the legacy quirks forward.
