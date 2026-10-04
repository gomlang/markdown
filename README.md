# ecosystem::markdown

A CommonMark parser with opt-in GFM tables and task lists, public ASTs and a
configurable HTML renderer implemented in GoML. The current implementation passes all **652 CommonMark 0.31.2 examples**
with exact HTML comparison. Parsing, rendering and Unicode punctuation/symbol
classification are implemented in GoML, alongside `std::unicode` whitespace and
case folding. This module has no direct Go FFI bindings. Its `ecosystem::html`
dependency also packages a native DOM adapter, so a consuming module needs a
`go.mod` with Go 1.26.0 even when it only imports Markdown. The root `go.mod`
provides this for this library and its example; GoML resolves the dependency's
pinned Go packages during builds.

```goml
use ecosystem::markdown;

fn render_page(source: string) -> Result[string, markdown::Error] {
    markdown::to_html(source)
}
```

## Coverage

- Paragraphs, ATX and Setext headings, thematic breaks, fenced and indented code.
- Nested block quotes, lazy paragraph continuation, ordered and unordered lists,
  empty items, tight/loose rendering and tab-aware container indentation.
- All seven CommonMark HTML block forms, inline tags, comments, processing
  instructions, declarations and CDATA.
- Code spans, nested emphasis/strong emphasis with delimiter-run rules,
  escapes, soft/hard breaks and Unicode-aware delimiter classification.
- Inline, full/collapsed/shortcut reference links, images, URI/email autolinks,
  balanced destinations, multiline titles and first-definition-wins references.
- Case-folded, whitespace-normalized reference labels, all 2,125 semicolon-ended
  HTML5 named entities, numeric references and replacement of invalid scalar
  references/NUL input.

Input line endings are normalized from CRLF and lone CR to LF. Code preserves
literal interior tabs; tabs used for container structure obey four-column stops.
Lazy paragraph continuations retain their leading spaces and tabs inside code
spans and link titles, including in nested block quotes and lists.
List markers without a quote prefix end a block quote, including empty list
items and ordered lists starting at numbers other than one. Explicit quote
prefixes retain ordinary paragraph-interruption rules within the quote.
HTML blocks inside quotes and lists require their container prefixes on every
nonblank line. Their bodies preserve literal tabs and ignore Markdown fence/table
markers until the HTML block's closing marker or required blank line.
Nested code fences and HTML blocks retain the required quote and list prefixes
at each level; their literal content cannot begin a lazy outer paragraph.
Paragraph continuation tracks its quote/list prefix path, so a sibling ordered
list can start with `2.` while the same marker inside a paragraph remains text.
Complete tag lines such as `<span>` (CommonMark HTML block type 7) cannot
interrupt an existing paragraph, including lazy quote/list continuation lines.
Link/image destinations are percent-encoded during rendering, while existing
percent escapes are preserved.

The default API implements CommonMark. Separate GFM APIs enable tables and task
lists; strikethrough, extended autolinks, footnotes, math and syntax highlighting
remain outside the implemented extension set. Passing the complete reference corpus is
evidence of the tested behavior, not proof that every possible input matches
every other Markdown implementation.

## Parsing and AST

`parse(source)` uses `ParseOptions::standard()`. `parse_with_options` accepts
limits for input bytes, nesting, allocated parse nodes and parsing work. Defaults
are 4 MiB input, depth 128, 250,000 parse nodes and 20,000,000 work units.
Malformed Markdown normally remains text, as CommonMark requires; resource-limit
failures return `Error { message, line }`. Error lines are zero based internally
and one based in their string representation.

`Document` exposes `blocks` and its normalized `references` map. `Block` contains
a `BlockKind` and an inclusive, zero-based `LineSpan`; `Inline` describes the
inline tree. Public enum cases preserve heading levels, list start numbers and
tightness, fence information, literal code, link/image destinations and titles.
Applications can inspect or construct these values and pass the AST to `render`.
Source spans describe whole blocks, not byte-accurate inline token locations.

Parsing first builds block structure and collects reference definitions. A
second pass resolves inline trees, including references defined later in the
document or inside containers. Emphasis uses a delimiter stack and linked node
indices; bracket handling prevents nested links while supporting formatted image
descriptions.

`walk(document, visit_block, visit_inline)` visits nodes in source order with
parents before children. It uses an explicit stack of sibling cursors with O(depth)
auxiliary storage, without pushing every child of a wide node, and accepts captured
callbacks. `inline_text(inlines)` extracts decoded text, code content and line
breaks without formatting tags or HTML escaping; image/link labels contribute
their text. Walks have limits of 250,000 visited nodes and depth 256, and return
errors for manually constructed cyclic/deep ASTs. A separate 250,000-item budget
bounds list-item scanning, including empty items in caller-built ASTs. Errors may
follow earlier visitor callbacks; no unvisited siblings are materialized up front.
Callbacks must not mutate the tree during traversal.

AST vectors and maps have normal GoML shared-storage semantics. Returning an AST
does not make its container storage immutable, and the module does not provide
concurrent mutation support.

## GFM tables and task lists

`parse_gfm(source)` enables both extensions. `parse_gfm_with_options(source,
parse_options, gfm_options)` accepts the existing resource limits and independent
`GfmOptions { tables, task_lists }` switches. `GfmOptions::disabled()` preserves
CommonMark parsing. `to_html_gfm(source)` uses the same safe rendering defaults as
`to_html`; `render_gfm(document, render_options)` selects rendering policy.

```goml
use ecosystem::markdown;

fn render_status() -> Result[string, markdown::Error] {
    markdown::to_html_gfm(
        "Package | Status\n:- | -:\nuuid | ready\n\n- [x] build\n- [ ] publish\n",
    )
}
```

The new `GfmDocument` has `blocks` and `references`. Each `GfmBlock` keeps its
inclusive line span and a `GfmBlockKind`: `CommonMark(BlockKind)` for ordinary
leaves, `Quote`, `List` or `Table`. Lists retain ordering, start and tightness;
each `GfmListItem` exposes `blocks` and `checked: Option[bool]`. `None` denotes an
ordinary item, `Some(false)` an unchecked task and `Some(true)` a checked task.
The task marker is removed from the first paragraph's inline text. Other
paragraphs and escaped markers remain ordinary text.

`Table` exposes `alignments`, `header` and `rows`. A cell is `Vec[Inline]`;
`Alignment` is `Default`, `Left`, `Center` or `Right`. Header and delimiter widths
must match. Short body rows are padded with empty cells and extra cells are
discarded. Leading/trailing pipes are optional; escaped pipes remain cell text,
including within code spans. Unescaped pipes divide cells even inside code
spans, following GFM's block-before-inline parsing. Blank lines and other block
structures end tables. Nested lists and quotes preserve these extensions.

The existing `Document`, `BlockKind`, `ParseOptions` and visitor signatures are
unchanged. Applications opt into the separate GFM tree and traverse its public
containers; `inline_text` remains available for its cells and paragraph leaves.
Constructed table ASTs must have at least one column and matching widths for
alignments, header and every row. Task items must start with a paragraph.
Malformed constructed trees return rendering errors, and cyclic trees remain
bounded by the rendering depth limit.

Table cells, including cells inserted into short rows, consume the parse-node
budget. Row scans consume the work budget. HTML text, URL filtering, output and
depth limits also apply to all GFM content. Checkboxes are always disabled; the
renderer follows `xhtml` for their void-element spelling. It adds no JavaScript
or interactive state management.

The tests retain all ten independently expected examples from the official
[GFM tables](https://github.github.com/gfm/#tables-extension-) and
[task list items](https://github.github.com/gfm/#task-list-items-extension-)
sections. Those specification examples are licensed under
[CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/), based on John
MacFarlane's CommonMark specification and GitHub's extensions. Additional tests
cover parser options, AST metadata, nesting, escaping, block precedence,
reference links, invalid task markers, resource limits and malformed ASTs.
The downstream consumer repeats the 652 CommonMark and 2,125 entity cases with
extensions disabled, alongside the existing standard API corpus test. This is
tested extension coverage, not a claim of complete GFM support.

## Rendering

`render(document, options)` returns HTML or a checked limit/AST error.
`to_html(source)` combines parsing with `RenderOptions::standard()`.

The default renderer escapes raw HTML and permits relative URLs plus `http`,
`https`, `mailto` and `ftp` schemes. Other schemes produce empty link/image
destinations. Scheme checking accounts for whitespace/control characters after
Markdown entity decoding. `escape_html` and `safe_url` are also public helpers.
The escape helper now uses `ecosystem::html` while retaining named double-quote
references and unchanged apostrophes. Entity parsing still follows Markdown's
existing rules and has not been replaced with permissive HTML decoding.

`RenderOptions::commonmark()` preserves raw HTML and arbitrary URI schemes to
match the reference corpus. Applications select that policy explicitly. Render
options also control soft-break conversion and HTML versus XHTML void-element
spelling. Titles, alt text and code language attributes are escaped; formatted
image descriptions become plain alt text.

Default rendering limits are depth 256 and 16 MiB output. They are independent of
parse limits. Manually constructed ASTs can therefore be checked without going
through the parser. Heading levels outside 1–6 and exceeded limits return errors.

## Resource behavior

There are no recursive calls proportional to plain-text length or delimiter-run
length. Container parsing and finalization recurse on bounded nesting; inline
nesting is checked as nodes are grouped. Some unsuccessful delimiter searches
can be quadratic. Work accounting charges delimiter scans and conservative
remaining-input bounds for candidate links/HTML so those paths can terminate
with a recoverable error. This is an operation budget, not a wall-clock timeout.

Parsing allocates block text and inline nodes; it is not an incremental editor
parser or a zero-copy rope. Input limits and parse-node limits also constrain
large valid documents. Applications handling larger documents can set explicit
limits. Escaped text is measured against the remaining output budget before its escaped
copy is allocated. URL encoding emits checked fragments directly, and scheme
checks retain at most six ASCII characters. Oversized literal, title, language,
and destination fields in caller-built ASTs therefore cannot first allocate an
unbounded expanded escape buffer. The output limit does not bound the caller's
existing AST or total process memory.

## Validation and data provenance

`punctuation.goml` contains 338 sorted, disjoint ranges covering the 8,612 Unicode
15.0.0 scalars in general categories P and S. The table comes from the official
[UnicodeData.txt](https://www.unicode.org/Public/15.0.0/ucd/UnicodeData.txt), whose
SHA-256 is `806e9aed65037197f1ec85e12be6e8cd870fc5608b4de0fffd990f689f376a73`.
It preserves the previous Unicode 15 classification and uses binary search without
runtime table allocation. The data license is in [LICENSE.unicode.txt](LICENSE.unicode.txt).

Run from this library repository:

```sh
(cd ../verification && just ecosystem-test markdown)
```

The command checks formatting, library tests, the example,
fresh/cached example builds, its executable and reference interoperability.
Library tests cover AST construction/inspection, spans, captured visitors,
Unicode/reference normalization, nested lists, literal tabs, escaping, URI
policy, render options and resource/cycle errors. The example
imports only public APIs and also offers stdin conversion through `--safe`,
`--commonmark` and a JSON-array batch interface through `--json`. `--gfm` enables
safe table/task rendering; `--gfm-commonmark` enables the same extensions with
raw-HTML/URI compatibility policy and HTML void-element spelling.

The example’s native `tests/reference_test.goml` checks all 652 examples from the [CommonMark 0.31.2 reference corpus](https://spec.commonmark.org/0.31.2/spec.json), together with all 2,125 named entities. The checked-in independent corpus records the original CommonMark SHA-256 digest `d431b29d97b6f73e69d547109cf5081578fac931e72afe95639ebe766c1b2a20`; running the tests needs no Python or network access. The CommonMark specification and examples are by John MacFarlane, licensed under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/).

`entities.goml` is now a generated compatibility wrapper over ecosystem::html's
shared lookup. The independently sourced [HTML entity data](../html/data/entities.json)
comes from CPython 3.12.3 `html.entities.html5`, restricted to semicolon-ended names.
Its license remains in [LICENSE.entities.txt](LICENSE.entities.txt) and the HTML
module. The native GoML generator validates the checksum and 2,125-entry count,
then generates the shared lookup and wrapper deterministically. Its ordinary
native test verifies both files on each `(cd ../verification && just ecosystem-test markdown)` run.

To regenerate, run `../../../goml-dev/stage2/bin/goml build` from `tools`, then `_artifact/bin/markdown_entities generate ..`. `check` verifies the file without writing. The data and generator need no Python installation.

## Development and examples

Requires GoML 0.1.56 or newer. The `examples/basic/` example shares the root manifest; test-only helpers are declared in `[dev-dependencies]`. From the library root, run:

```sh
goml run --example basic
goml test
goml verify --timeout 300s
```

`goml test` builds the example and runs its tests. `goml verify` repeats the example checks as an independent module against an isolated registry snapshot. `(cd ../verification && just ecosystem-test markdown)` also retains the library-specific smoke and compatibility checks.
