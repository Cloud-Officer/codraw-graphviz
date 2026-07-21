# Code Review: codraw/graphviz

## Fixes applied (2026-07-20)

- **composer.json:** PHP version constraint changed from unbounded `>=8.5` to `^8.5` (version-compatibility debt: prevents a future PHP 9 from installing against this package; no effect on any currently existing PHP version).
- **Finding 5** (`Graph::__toString()` mutates object state / sticky `directed` flag): `Graph.php` now computes directedness in a local variable on each render instead of setting the private `$directed` property; the now-unused property was removed. Rendering, flipping an edge with `setDirected(false)`, then re-rendering now correctly emits `graph`/`--` instead of the stale `digraph`.
- **Finding 11** (partially, composer.json / tooling nits):
  - Removed the misleading `symfony` keyword from `composer.json` (the package has no Symfony dependency). Metadata-only, no consumer impact.
  - Updated `phpunit.xml.dist` schema reference from `10.4` to `11.3` to match the `phpunit/phpunit: ^11.3 || ^12.0` dev requirement. Dev-only.
  - Open items (deliberately not changed): `"php": ">=8.5"` is the repo-wide convention and was kept as-is; the PSR-4 root mapping that also autoloads `Tests\` was left untouched (adding `autoload-dev`/`exclude-from-classmap` changes autoload behavior for consumers and belongs in a coordinated repo-wide change).

- **Finding 9** (inconsistent `\Stringable` declarations): `Edge` and `AttributeBag` now declare `implements \Stringable`, matching `Graph` and `Node`. Purely declarative (PHP auto-implements the interface for any class defining `__toString()`); both classes' rendered output is already asserted by the existing `GraphTest` cases, which pass unchanged.

Validation (2026-07-20): `composer install` resolves cleanly with the `^8.5` constraint (PHPUnit 12.5 installed), the full test suite passes (3 tests, 3 assertions), PHPStan reports no errors against the empty baseline, and markdownlint is clean. No test or baseline fallout from the applied fixes. A second validation pass on 2026-07-20 re-confirmed all of the above and added the Finding 9 fix (the only remaining finding whose behavior is directly covered by the existing tests); phpunit and PHPStan were re-run green afterwards.

No runtime dependencies are missing from `composer.json`: shipped code imports nothing outside the package (verified by grep), and PHPUnit is correctly confined to `require-dev`. All other findings (1-4, 6-8, 10) require escaping-policy/design decisions or would change rendered output for inputs that currently "work", so they were intentionally left open.

## Overall Assessment

`codraw/graphviz` is a very small, dependency-free component (4 classes + 1 trait, ~260 LOC) that builds Graphviz DOT source from an object model (`Graph`, `Node`, `Edge`, `AttributeBag`). The code is clean, modern PHP (constructor promotion, typed properties, fluent API) and works correctly for the controlled, framework-internal identifiers it was presumably written for. However, as a general-purpose DOT generator it has real correctness and injection gaps: node/edge/graph identifiers are emitted completely unescaped, attribute-value escaping uses the wrong function (`addslashes`) for the DOT grammar, and any string value starting with `<` bypasses escaping entirely. A graph mixing directed and undirected edges silently produces syntactically invalid DOT. Test coverage exists but is thin (Graph happy paths only). Grade: **C** — solid skeleton, but the output-encoding layer, which is the entire point of the library, is not trustworthy with arbitrary input.

## Findings

### High

#### 1. Node/edge/graph identifiers are emitted without any quoting or escaping (DOT injection / invalid output)

- `Node.php:23` — `$result = $this->name;`
- `Edge.php:42-47` — `sprintf('%s %s %s', $this->from, ..., $this->to)`
- `Graph.php:78-82` — `sprintf('%s %s {', ..., $this->name)`

A DOT ID may only contain alphanumerics/underscores (or be quoted). Any name containing a space, hyphen, dot-reserved keyword (`graph`, `node`, `edge`, `subgraph`, `strict`), or punctuation produces invalid DOT — e.g. `new Node('my node')` renders `my node;`, which Graphviz rejects. Worse, because nothing is escaped, a name is a direct injection point into the DOT stream: `new Node('a; b [image="/etc/passwd"]')` injects arbitrary statements. Graphviz attributes such as `image`, `shapefile`, and `fontpath` reference the filesystem, so if names/labels ever derive from user input and the output is fed to `dot`, this becomes a local-file-read / SSRF-adjacent vector, not just broken output. Identifiers should be validated against the DOT ID grammar or quoted-and-escaped like attribute values.

#### 2. Any attribute value starting with `<` bypasses escaping entirely

- `AttributeBag.php:41-43`

```php
if (str_starts_with($value, '<')) {
    return $value;
}
```

The intent is to pass HTML-like labels (`<<b>x</b>>`) through raw, but the check matches *any* string whose first character is `<`. Two consequences: (a) a legitimate plain-text value such as `"<= 10"` is emitted unquoted and produces invalid DOT; (b) it is a trivially reachable escape hatch around all quoting — an attacker-influenced value like `<>], evil [image="/x"` injects arbitrary attributes/statements. At minimum the check should require the value to be a well-formed HTML string (`<...>` balanced, e.g. starts with `<` **and** ends with `>`), and callers should have an explicit opt-in (e.g. an `HtmlLabel` value object) rather than sniffing content.

### Medium

#### 3. `addslashes()` is the wrong escaping function for DOT quoted strings

- `AttributeBag.php:45` — `return '"'.addslashes($value).'"';`

DOT double-quoted strings recognize only `\"` (and pass other backslash sequences through to the renderer, where `\n`, `\l`, `\r` are meaningful in labels). `addslashes` additionally escapes `'` and NUL: a label `it's` becomes `"it\'s"` and Graphviz renders the literal text `it\'s`. It also doubles backslashes, so a caller intentionally passing the label escape `Line1\nLine2` gets `Line1\\nLine2` — a literal `\n` in the rendered label instead of a line break. Correct minimal escaping is `str_replace('"', '\\"', $value)` (with a documented policy for backslashes/newlines).

#### 4. Mixed directed/undirected edges produce syntactically invalid DOT

- `Graph.php:71-80` together with `Edge.php:45`

If *any* edge is directed the graph is rendered as `digraph`, but undirected edges still render with `--` (`Edge::__toString`). In a `digraph`, `--` is a syntax error (and vice-versa `->` inside `graph`). So `(new Graph('g'))->addEdge(new Edge('a','b'))->addEdge(new Edge('c','d', directed: false))` silently emits DOT that Graphviz rejects. Since `Edge` defaults to `directed: true` while `Graph` defaults to undirected, this mismatch is easy to hit. The graph type should drive the edge operator (or an exception should be thrown on a mix; Graphviz can represent "undirected within digraph" with `dir=none`).

#### 5. **[FIXED]** `Graph::__toString()` mutates object state and the flag is sticky

- `Graph.php:71-76`

```php
foreach ($this->edges as $edge) {
    if ($edge->getDirected()) {
        $this->directed = true;
        break;
    }
}
```

A `__toString()` implementation should be side-effect free. `$this->directed` is set but never reset, so the state is stale: render a graph containing a directed edge, then flip that edge with `setDirected(false)` (a mutator `Edge` explicitly provides), and subsequent renders still say `digraph`. The directedness check should use a local variable computed on each call, or `directed` should be an explicit constructor/setter concern of `Graph`.

#### 6. Attribute keys are emitted unvalidated

- `AttributeBag.php:26-30` — `sprintf('%s=%s', $key, ...)`

Keys come straight from the array passed by the caller and are never quoted or validated, so a key containing spaces or `]` breaks the attribute list or injects into it (same class of problem as finding 1, on the other side of the `=`). Keys should be validated against the DOT ID grammar.

### Low

#### 7. `formatAttribute()` has no return type and passes non-scalars through

- `AttributeBag.php:38-53`

The final `return $value;` returns arrays/objects untouched; an array value then hits `sprintf('%s=%s', ...)` and triggers "Array to string conversion" (or a `TypeError` for non-Stringable objects) deep inside `__toString`, where exceptions are awkward. Declare a `string` return type and reject unsupported types explicitly. Also note `null` renders as an empty right-hand side (`key=`), which is invalid DOT.

#### 8. Unnamed graph renders with a double space

- `Graph.php:78-82`

With the default `name: null`, `sprintf('%s %s {', 'graph', null)` yields `graph  {` (two spaces). Harmless to Graphviz but sloppy; also an anonymous graph arguably shouldn't print the empty name slot at all.

#### 9. **[FIXED]** Inconsistent `\Stringable` declarations

`Graph` and `Node` declare `implements \Stringable`; `Edge` (`Edge.php:5`) and `AttributeBag` (`AttributeBag.php:5`) define `__toString()` but do not. PHP auto-implements it at runtime, but static analysis and `Stringable` type hints treat them differently. Declare it on all four for consistency. *Fixed 2026-07-20: all four classes now declare `implements \Stringable`.*

#### 10. `Graph::addNode()` silently overwrites a node with the same name

- `Graph.php:33-38`

Keying `$this->nodes` by name means adding two nodes with the same name silently drops the first one's attributes. That may be intended (DOT node identity is the name) but it is undocumented; consider merging attributes or throwing on conflict.

#### 11. **[PARTIALLY FIXED]** composer.json / tooling nits

- `composer.json:18` — `"php": ">=8.5"` is unbounded and will claim compatibility with PHP 9.x sight unseen; the conventional constraint is `^8.5`.
- `composer.json:25-29` — the PSR-4 root mapping `"Draw\\Component\\Graphviz\\": ""` also autoloads `Tests\` in production installs; there is no `autoload-dev` separation (mitigated only if the repo is export-ignored into a dist).
- `composer.json:9` — the `symfony` keyword is misleading; the package has no Symfony dependency.
- `phpunit.xml.dist:3` — schema pinned to `10.4` while `require-dev` demands PHPUnit `^11.3 || ^12.0`.

## Strengths

- Zero runtime dependencies and a tiny, focused API — easy to audit and hard to misuse structurally.
- Modern, clean PHP 8: constructor property promotion, typed properties, union types (`array|AttributeBag`), fluent setters.
- `AttributeHolderTrait` gives Graph/Node/Edge a consistent attribute mechanism without duplication; accepting either an array or a prebuilt `AttributeBag` is a nice ergonomic touch.
- Empty `phpstan-baseline.neon` — the code is static-analysis clean with no suppressed debt.
- CI workflows, CODEOWNERS, PR/issue templates, and security scanning config (trivy, semgrep) are all in place for such a small package.
- Output formatting (indentation, one node/edge per line) produces readable DOT, and the existing tests lock that formatting in.

## Test Coverage

Coverage is thin and Graph-centric. `Tests/GraphTest.php` has exactly three happy-path cases: a directed one-edge graph, an empty undirected graph, and graph-level attributes. Untested areas:

- **Node**: never tested at all (with or without attributes); `getNode`/`removeNode`/overwrite-by-name behavior untested.
- **Edge**: attributes on edges, `setDirected`, undirected rendering (`--`) untested.
- **AttributeBag**: none of the `formatAttribute` branches are exercised directly — string escaping (quotes, backslashes, apostrophes), the `<` HTML-passthrough path, booleans, numerics, non-scalar values.
- **Interaction bugs**: mixed directed/undirected graphs (finding 4) and the sticky `directed` flag (finding 5) would have been caught by fairly obvious tests.

The most valuable additions would be an `AttributeBagTest` covering every value-type branch (including hostile strings) and a mixed-edge `Graph` test asserting valid DOT.
