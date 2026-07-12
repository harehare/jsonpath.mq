<h1 align="center">jsonpath.mq</h1>

A [JSONPath](https://en.wikipedia.org/wiki/JSONPath) ([RFC 9535](https://www.rfc-editor.org/rfc/rfc9535)-style) query engine implemented as an [mq](https://github.com/harehare/mq) module.

It evaluates JSONPath expressions against already-parsed mq values — the same dict/array/string/number/bool/`None` values produced by `json::json_parse`, `json5::json5_parse`, YAML/TOML parsing, or any other mq data source — giving `mq` deep path extraction and filtering on top of its existing JSON support.

## Features

- Root (`$`) and child access: `.name`, `['name']`
- Wildcards: `.*`, `[*]`
- Recursive descent: `..name`, `..*`, `..[...]`
- Index selectors, including negative indices: `[0]`, `[-1]`
- Slice selectors with optional start/end/step: `[1:3]`, `[::2]`, `[::-1]`
- Union of selectors: `[0,2]`, `['a','b']`
- Filter selectors (`[?(<expr>)]`) with:
  - Comparisons: `==`, `!=`, `<`, `<=`, `>`, `>=`
  - Logical operators: `&&`, `||`, `!`, parentheses
  - Existence tests: `[?(@.isbn)]`
  - `@` (current node) and `$` (root) references inside filters
  - Functions: `length()`, `count()`, `value()`, `match()`, `search()`
- Normalized RFC 9535 path strings for every match (`$['store']['book'][0]['title']`)

## Installation

Copy `jsonpath.mq` to your mq module directory, or place it anywhere and reference it with `-L`.

```sh
cp jsonpath.mq ~/.local/mq/config/
```

## Usage

```sh
mq -L /path/to/modules -I json \
  'import "jsonpath" | jsonpath::query(., "$.store.book[*].title")' data.json
```

If you copied it to the mq built-in module directory:

```sh
mq -I json 'import "jsonpath" | jsonpath::query(., "$..price")' data.json
```

## API

### `query(input, path)`

Evaluates a JSONPath expression against `input` and returns an array of all matched values, in document order.

```mq
import "jsonpath"
| jsonpath::query({"a": [1, 2, 3]}, "$.a[*]")
# => [1, 2, 3]
```

### `query_first(input, path)`

Evaluates a JSONPath expression and returns only the first matched value, or `None` if there is no match.

```mq
jsonpath::query_first({"a": {"b": 42}}, "$.a.b")
# => 42
```

### `exists(input, path)`

Returns `true` if the JSONPath expression matches at least one node.

```mq
jsonpath::exists({"a": 1}, "$.b")
# => false
```

### `count(input, path)`

Returns the number of nodes matched by the JSONPath expression.

```mq
jsonpath::count({"a": [1, 2, 3]}, "$.a[*]")
# => 3
```

### `paths(input, path)`

Returns an array of RFC 9535 normalized path strings, one for each match.

```mq
jsonpath::paths({"a": [1, 2]}, "$.a[*]")
# => ["$['a'][0]", "$['a'][1]"]
```

### `query_with_paths(input, path)`

Returns an array of `{"path": <normalized path string>, "value": <value>}` for each match — useful when you need both the matched value and where it came from.

```mq
jsonpath::query_with_paths({"a": 1}, "$.a")
# => [{"path": "$['a']", "value": 1}]
```

## Example

Given `store.json`:

```json
{
  "store": {
    "book": [
      {"category": "reference", "author": "Nigel Rees", "title": "Sayings of the Century", "price": 8.95},
      {"category": "fiction", "author": "Evelyn Waugh", "title": "Sword of Honour", "price": 12.99},
      {"category": "fiction", "author": "Herman Melville", "title": "Moby Dick", "isbn": "0-553-21311-3", "price": 8.99}
    ],
    "bicycle": {"color": "red", "price": 19.95}
  }
}
```

```sh
# All book titles
mq -L . -I json 'import "jsonpath" | jsonpath::query(., "$.store.book[*].title")' store.json
# => ["Sayings of the Century", "Sword of Honour", "Moby Dick"]

# Books under $10
mq -L . -I json 'import "jsonpath" | jsonpath::query(., "$.store.book[?(@.price < 10)].title")' store.json
# => ["Sayings of the Century", "Moby Dick"]

# Every price in the document
mq -L . -I json 'import "jsonpath" | jsonpath::query(., "$..price")' store.json
# => [8.95, 12.99, 8.99, 19.95]

# Books that have an ISBN
mq -L . -I json 'import "jsonpath" | jsonpath::query(., "$.store.book[?(@.isbn)].title")' store.json
# => ["Moby Dick"]
```

## Notes

- This module operates on mq's native values, not raw JSON text. Parse JSON/JSON5/YAML/etc. first (e.g. with the built-in `json` module or [json5.mq](https://github.com/harehare/json5.mq)), or pass `-I json`/`-I yaml`/etc. so `mq` parses the input for you.
- Object member order follows mq's own internal dict ordering, which is not guaranteed to match JSON source order.
- This is a query engine, not a mutation API — it does not modify `input` or provide JSONPath-based `set`/`delete` operations.

## Compatibility

Requires [mq](https://github.com/harehare/mq) v0.6 or later.

## License

MIT
