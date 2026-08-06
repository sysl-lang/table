# table

Tables of text for [sysl](https://github.com/sysl-lang/sysl), laid out so the columns line up.

A cell is a **value**, not a string — anything that renders itself can go in one, so a row of a name,
a count and a flag is ordinary code and nothing has to be converted on the way in. What comes out is
whichever shape the reader wants: a plain aligned block, a box-drawn grid in five weights, a Markdown
table, tab-separated fields, or a matrix in brackets.

```
sh/sysl/table/
    table.sysl          the library
    tests.sysl          its tests, run by `sysl test .`
package.hocon           who this package is, and what it needs of the machine
```

The module is **`sh.sysl.table`**, and the three directories are that name: a dotted module name
mirrors its path from the library root. The prefix is the reverse-DNS of `sysl.sh`, so that a package
claims a name nobody else will mint rather than the top-level word `table`.

## Using it

Name it in your project's `package.hocon` and `sysl build` fetches it:

```hocon
dependencies {
  table { git = "github.com/sysl-lang/table", version = "0.1.0" }
}
```

The coordinate is an identity rather than a URL, so it carries no `https://`, and `version` is the
tag `v0.1.0` here.

Or build it into an artifact and compile against that, which needs no fetching:

```
sysl build-lib . -o /tmp/table.syslib
sysl run yourprogram.sysl --lib /tmp/table.syslib
```

## A first table

```sysl
import sh.sysl.table.*

main()
    var t = table()

    t.header(["language", "released", "typed"])
    t.add(["sysl", 2026, true])
    t.add(["C", 1972, true])
    t.add(["Python", 1991, false])

    print(t.text())
```

```
 language  released  typed
 sysl      2026      true
 C         1972      true
 Python    1991      false
```

The header is bold and underlined on a terminal; `set_ansi(false)` turns that off, and the output
above is what you get with it off.

`text` hands back a `string`. `render(out)` writes through a `*Writer` instead, which is the one that
costs nothing — a table going to the terminal never becomes a string at all.

## Styles

A style says what the table is made of, and `dividers` says whether the columns are separated inside
it. Everything follows from the pair:

```sysl
g.set_style(Light)
g.set_dividers(true)
g.set_header_line(true)
g.right_align(1usize)
g.right_align(2usize)
```

```
┌─────────┬─────┬───────┐
│  item   │ qty │ price │
├─────────┼─────┼───────┤
│ apples  │  12 │   1.5 │
│ oranges │   3 │  2.25 │
└─────────┴─────┴───────┘
```

| style | what it draws |
|---|---|
| `Plain` | columns separated by spaces, no frame — the default |
| `Ascii` | `+---+`, for a terminal that cannot be trusted with anything else |
| `Light` | `┌─┬┐ │ ├┼┤ └┴┘` |
| `Heavy` | `┏━┯┓ ┃ ┣┿┫ ┗┷┛` — heavy frame, light dividers |
| `Double` | `╔═╤╗ ║ ╠╪╣ ╚╧╝` — double frame, light dividers |
| `Rounded` | `╭─┬╮ │ ├┼┤ ╰┴╯` |
| `Markdown` | a Markdown table, alignments carried in the separator row |
| `Tabbed` | fields separated by tabs, for something else to read |
| `Matrix` | bracketed as a matrix, `⎡ ⎤` |
| `MatrixRounded` | the same in parentheses, `⎛ ⎞` |

The frame is heavy or double and the dividers inside it are not, deliberately: the frame says where
the table ends and a divider says where a column does, and drawing both at one weight makes the
second as loud as the first.

The last four ignore `dividers`, because for them the separator is not a choice a caller has.

```sysl
m.set_style(Markdown)
m.right_align(1usize)
```

```
| column |      meaning      |
| ------ | ----------------: |
| width  | how wide it looks |
| bytes  | how much it takes |
```

```sysl
x.set_style(Matrix)
```

```
⎡1  0  0⎤
⎢0  1  0⎥
⎣0  0  1⎦
```

A bracket is three characters tall whatever the matrix is, so a one-row matrix takes the middle piece
on both sides: `⎢1  2  3⎥` is a row vector, and `⎡1  2  3⎦` is nothing.

## Alignment

Two levels. A column has one, and a cell may override it:

```sysl
t.set_column_align(0usize, Center)   -- this column
t.right_align(1usize)                -- the one a table is usually asked for by name
t.cell_align(2usize, Right)          -- this cell, in the row most recently added
```

A header carries an alignment of its own, so a column alignment set later reaches the numbers without
moving the names over them. `set_header_centered(false)` lays the names out like any other row.

Columns are indexed from **0**, unlike the Scala library this was ported from, which indexed from 1.

## Rules and headers

`line()` draws a rule above the next row added, so a rule asked for after the last row has nothing to
sit above and is dropped — which is what makes it safe to call inside a loop.

```sysl
t.set_header_line(true)          -- a rule between the header and the data
t.set_header_bold(true)          -- ANSI, and dropped for Markdown and tabbed output
t.set_header_underlined(true)
t.set_header_centered(true)
```

`cell_style(col, esc)` puts an escape sequence around one cell's text, and sequences accumulate, so a
cell may be given a colour and a weight by two calls. `underline(col)` is separate from it because an
underline covers the cell's padding as well as its text, which is what makes a row of underlined
headers one continuous line rather than a line per word.

## Width is columns, not bytes and not characters

The whole of the layout rests on this. `café` is five bytes, four characters and four columns;
`日本` is six bytes, two characters and **four** columns. Only the last of the three says where the
next border falls.

```
╭──────────┬──────────╮
│   言語   │   city   │
├──────────┼──────────┤
│ 日本語   │ 京都     │
│ Français │ Montréal │
╰──────────┴──────────╯
```

That number comes from `sysl.text.columns`, which is the East Asian Width and combining-mark data of
the Unicode Character Database. A format specifier's width would have been the obvious thing to reach
for and is the wrong one: it counts bytes, deliberately, so that it means what `snprintf` means. Every
cell here is rendered with a neutral specifier and padded by this library against a count it measured
itself.

## Rendering changes nothing

`render` and `text` take `self` rather than `*self`, so a table may be rendered, added to, and
rendered again, and the second answer is the table as it then stands. Nothing is cached and no
phantom row is appended to make the arithmetic line up: the widths are measured for the rendering
that asked for them and thrown away with it.

## What it does not do

A cell is one line. Wrapping a cell too wide for its column, spanning a cell across columns, and
laying a table out to a total width are all more of the same arithmetic and none of them is here.

A row must have one cell per column, the first row settles how many, and the header must be the first
row. Those are contract clauses, so a program that breaks one stops with the line that did it —
they are mistakes in the calling code rather than conditions to be handled.

## What changed in the port

It was ported from [`edadma/table`](https://github.com/edadma/table), a Scala library, and holds that
library's output byte for byte wherever the two can both express the case — the Scala tests were 445
lines of exact expected output, and they are the specification this was written against. Four
differences are deliberate:

- **`Double` draws a double border.** There it was a name in an enumeration and nothing else: every
  branch tested for light and fell through to heavy, so a double border silently drew a heavy one.
- **`Rounded` is a style.** The four arc glyphs were declared there and never referenced, with three
  comments marking where they would have gone.
- **A collection is a cell like any other.** The stringifier there tested each value for being an
  array or a sequence and joined it by hand, because everything arrived as `Any`. Here a slice renders
  itself, so the special case does not exist.
- **Ten constructor flags became two axes.** `border`, `columnDividers`, `markdown`, `tabbed`,
  `matrix` and `matrixRounded` encoded one choice between them and admitted combinations that meant
  nothing — Markdown *and* matrix, tabbed with a heavy border. A style and a `dividers` flag say the
  same things and no others, and the drawing code reads one glyph table instead of re-deriving the
  same four-way test at twelve separate sites.

Two defects went with them: the width was measured in UTF-16 code units, which misaligns any column
holding a character outside the Basic Multilingual Plane's narrow range; and rendering mutated the
table it was rendering, so a table rendered, added to, and rendered again was quietly wrong.

## Tests

```
sysl test .
```

Forty cases, each asserting the exact bytes a caller gets. A table is one of the few things whose
whole contract is its output — a column one space narrower than it should be is not a degraded table,
it is a wrong one — so nothing here counts lines or looks for a substring.

## License

ISC
