# xml

An XML parser written in Ecko: parse a document into a node tree, query it by
tag or path, extract text, and build XML back. Enough for the XML that APIs
answer in - S3 and GCS listings, WebDAV, RSS, SOAP. Pure computation - no
capabilities.

## Install

```bash
ecko get github.com/ecko-lang/xml
```

```ecko norun
import xml
```

## Usage

```ecko
import xml

feed = "<rss><channel><title>News</title><item id=\"1\"><title>R&amp;D</title></item><item id=\"2\"><title>Docs</title></item></channel></rss>"
doc = xml.parse(feed)

print(len(xml.find(doc, "item")))
# 2
print(xml.text(doc, "channel/title"))
# News
item = xml.find_one(doc, "item")
print(xml.attr(item, "id"), xml.text(item))
# 1 R&D
print(xml.build(item))
# <item id="1"><title>R&amp;D</title></item>
print(xml.escape("R&D <draft>"))
# R&amp;D &lt;draft&gt;
```

A node is a map: elements are `{ tag, attrs, children }`, text is
`{ tag: "#text", text }`, and the root is `{ tag: "#root", attrs, children }`.

### An S3 listing

```ecko
import xml

page = "<ListBucketResult><IsTruncated>true</IsTruncated><Contents><Key>a.gz</Key><Size>1024</Size><ETag>&quot;9b2c&quot;</ETag></Contents><NextContinuationToken>tok1</NextContinuationToken></ListBucketResult>"
listing = xml.parse(page)
for c in xml.find(listing, "Contents") {
    print(xml.text(c, "Key"), int(xml.text(c, "Size")), xml.text(c, "ETag"))
}
# a.gz 1024 "9b2c"
if xml.text(listing, "IsTruncated") == "true" {
    print(xml.text(listing, "NextContinuationToken"))
}
# tok1
```

### Malformed input

Malformed input throws, so catch it where the document comes from outside:

```ecko
import xml

e = try {
    xml.parse("<a><b>x</c></a>")
} catch (e) {
    e
}
print(e.kind, e.reason, e.line, e.column)
# parse mismatched_tag 1 8
```

## API

| function | returns |
|---|---|
| `parse(src, max_depth: 256, max_size: 16777216)` | the document as a `#root` node tree, entities decoded and CDATA as text |
| `find(node, path)` | every element matching `path`: a bare tag matches all descendants, `"a/b/c"` navigates children |
| `find_all(node, path)` | the same list as `find`, named to read beside `find_one` |
| `find_one(node, path)` | the first match of `find`, or `null` |
| `text(node, path: null)` | all text under `node`; with `path`, the text of `find_one(node, path)` or `null` |
| `attr(node, name)` | the attribute's value (case-sensitive), or `null` |
| `build(node)` | the node as an XML string: attributes sorted, empty elements self-closed, text and values escaped |
| `escape(s)` | `s` with `& < > " '` replaced by entities |
| `unescape(s)` | `s` with the predefined entities and numeric references decoded |

## Notes

### Errors

Malformed input throws kind `parse` with `format: "xml"`, `reason`,
`message`, `offset` (characters from the start), and 1-based `line` and
`column`:

| kind | reason | when |
|---|---|---|
| `parse` | `unterminated` | a comment, CDATA section, processing instruction or tag that never closes; also a bare `<` in text |
| `parse` | `bad_tag` | a tag that does not parse (an unquoted or valueless attribute, `</ a>`), or a `<` inside an attribute value |
| `parse` | `bad_name` | an element or attribute name that is not an XML name, such as `<1a/>` |
| `parse` | `mismatched_tag` | `</c>` where `</b>` was due; the message names where `<b>` opened |
| `parse` | `unexpected_close` | a closing tag with nothing open |
| `parse` | `unclosed` | the document ends with an element still open (the position is its start tag) |
| `parse` | `duplicate_attribute` | the same attribute twice on one element |
| `parse` | `bad_entity` | a `&` that does not begin `&name;` |
| `parse` | `unknown_entity` | an entity other than the five predefined ones, such as `&nbsp;` |
| `parse` | `bad_char_ref` | `&#...;` that is not a number, or not a character XML allows (`&#1;`, `&#xFFFE;`) |
| `parse` | `doctype_subset` | a `<!DOCTYPE ... [ ... ]>` with an internal subset |
| `parse` | `bad_declaration` | `<!ELEMENT>` and similar outside a DOCTYPE, or a DOCTYPE after the root |
| `parse` | `text_outside_root` | non-whitespace text or CDATA outside every element |
| `parse` | `no_root` | no element at all, including an empty string |
| `limit` | - | over `max_size` or `max_depth`; carries `limit` (`"max_size"` or `"max_depth"`) and `max` |
| `bug` | `bad_path` | an empty path or an empty segment (`""`, `"/a"`, `"a//b"`) |
| `bug` | `bad_name` | `build` given a tag or attribute name that is not an XML name |
| `bug` | `bad_argument` | an argument of the wrong type, or a `build` value that is not a string, number or Bool |

`unescape` throws the same `parse` errors as `parse`, with the offset into its
argument.

### Behaviour

- **Malformed input throws.** Before 0.38.0 the parser accepted anything:
  `<a><b>x</c></a>` and an unclosed `<b>` both came back as trees. A
  truncated download or an HTML error page now fails at the line that is
  wrong rather than producing a quietly wrong answer.
- **Entities are decoded.** Before 0.38.0, `<ETag>&quot;abc&quot;</ETag>` came back as
  the literal `&quot;abc&quot;`. It is now `"abc"`, in attribute values too.
  Code that unescaped by hand should stop, or `&amp;lt;` will be decoded
  twice. `build` escapes on the way out, so `build(parse(x))` still parses
  back to the same tree.
- **CDATA is text.** Its content becomes a `#text` node verbatim. 0.37
  dropped it.
- **No DTDs, by decision.** Only the five predefined entities and numeric
  references exist. A DOCTYPE with an internal subset is refused outright
  rather than parsed, so entity-expansion attacks ("billion laughs") have
  nothing to expand. A DOCTYPE without a subset is skipped, and an external
  DTD it names is never fetched.
- **Bounded.** `max_size` (characters, default 16 MiB) is checked before any
  parsing; `max_depth` (default 256) caps element nesting, self-closing
  elements included. Both throw kind
  `limit`, the same kind as the runtime's own ceilings. Raise them per call:
  `xml.parse(src, max_depth: 1024)`.
- **Fragments are allowed.** Several top-level elements parse as siblings
  under `#root`, as before. Text outside them must be whitespace.
- **Whitespace is kept.** Whitespace between elements stays as `#text`
  nodes, as in 0.37, so existing trees are unchanged. `text(node, path)` and
  `find` are unaffected by it.
- **Namespaces stay as written.** `s3:Key` is a tag named `s3:Key`; it does
  not match `find(doc, "Key")`, and `xmlns` is an ordinary attribute. S3
  uses a default namespace, so its tags have no prefix.
- **A byte order mark is skipped.** A leading U+FEFF (common from .NET and
  Windows services) is not content and does not count as a column.
- **`build` refuses bad names.** A tag or attribute name that is not an XML
  name would let a name inject markup, so it throws instead.
- Case-sensitive tags and attributes (unlike the `html` package).
- Attribute values are not whitespace-normalised.
- Speed: one native tokenising pass, then a linear walk. A 1 MB S3 listing
  (4,000 objects) parses in about 0.8 s.

### Limits

No namespace-aware matching (`{ns}Key`), no attribute predicates in paths, no
pretty-printed output or XML declaration from `build`, and no streaming parse:
the whole document is held in memory.

## Testing

```bash
ecko test
```

## License

[MIT](LICENSE)
