# XML - Ecko Std Lib Package

An XML parser for [Ecko](https://ecko.sh), written in Ecko. Parse a document
into a node tree, query it by tag or path, extract text, and rebuild.

## Install

```bash
ecko get github.com/ecko-lang/xml
```

## Usage

```ecko
import xml

doc = xml.parse(rss_source)

xml.find(doc, "item")           # every <item> descendant
xml.find(doc, "channel/item")   # path query: <item> under <channel>
xml.text(node)                  # concatenated text content
xml.attr(node, "id")            # attribute value (case-sensitive), or null
xml.build(node)                 # recompose to an XML string
```

A node is a map: elements are `{ tag, attrs, children }`, text is
`{ tag: "#text", text }`, and the root is `{ tag: "#root", ... }`.

## API

| Function | Description |
|---|---|
| `parse(src)` | Parse XML into a `#root` node tree |
| `find(node, path)` | A bare tag matches all descendants; `"a/b/c"` navigates (first segment matches descendants, each further segment its children) |
| `text(node)` | Concatenated descendant text |
| `attr(node, name)` | Attribute value (case-sensitive), or `null` |
| `build(node)` | Recompose to an XML string (attrs sorted; empty elements self-close) |

## Notes

- Case-sensitive tags and attributes (unlike the `html` package).
- Handles self-closing elements, comments, the `<?xml ?>` declaration, and
  `<!...>` declarations.
- Namespaces are kept as literal prefixes (`ns:tag`); no namespace resolution.

## Testing

```bash
ecko test tests/
```

## License

MIT - see [LICENSE](LICENSE).
