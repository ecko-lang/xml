# xml

## `parse(src)`

Parse an XML document into a node tree. The root is a `#root` node whose
children are the top-level elements; text becomes `#text` nodes.

## `text(node)`

All text under a node, concatenated - the element's own text plus every
descendant's, in document order. Returns "" for an element with no text.

## `find(node, path)`

find(node, path) - a single tag matches descendants; "a/b/c" navigates:
the first segment matches descendants, each further segment its children.

## `attr(node, name)`

One attribute's value, or null when the node has no such attribute.

## `build(node)`

build(node) -> an XML string (attrs sorted; empty elements self-close).
