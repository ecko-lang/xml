# Changelog

## 0.39.0

### Fixed

- A leading UTF-8 byte order mark (common from .NET and Windows services) is
  skipped instead of failing with `text_outside_root`; columns on the first
  line do not count it.
- `max_depth` now counts self-closing elements: `parse("<a/>", max_depth: 0)`
  throws `limit` like an element with content would.
- `escape`, `unescape` and every query function name a wrong argument with
  `bug`/`bad_argument` instead of failing with "contains doesn't support int
  and string" or similar.
- `build` writes Int, Float, Decimal and Bool text and attribute values as
  they print, instead of failing.
- `build` refuses a tag or attribute name that is not an XML name with
  `bug`/`bad_name`; a name containing `"` or a space could inject markup.
- `parse` refuses input that is not well-formed XML but was accepted: a name
  that is not an XML name (`<1a/>`, reason `bad_name`), a `<` inside an
  attribute value (`bad_tag`), and a character reference to a character XML
  does not allow (`&#1;`, `&#xFFFE;`, `bad_char_ref`).
- An empty path or an empty path segment (`""`, `"/a/b"`, `"a//b"`) throws
  `bug`/`bad_path` instead of quietly matching nothing.

### Breaking

- `find`, `find_all`, `find_one` and `text` throw `bug`/`bad_path` for a path
  with an empty segment. Migration: write tags joined by single `/`, with no
  leading or trailing `/`.
- `parse` refuses invalid names, `<` in attribute values and disallowed
  character references (see Fixed). Migration: such input was never
  well-formed XML; catch the `parse` error where the document comes from
  outside.
