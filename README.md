# Oniguruma-To-ES

A lightweight **Oniguruma to JavaScript RegExp transpiler** that runs in the browser and on your server. Use it to:

- Take advantage of Oniguruma's extended regex capabilities in JavaScript.
- Run regexes intended for Oniguruma in JavaScript, such as those used in TextMate grammars (used by VS Code, [Shiki](https://shiki.matsu.io/) syntax highlighter, etc.).
- Share regexes across your Ruby and JavaScript code.

Compared to running the actual [Oniguruma](https://github.com/kkos/oniguruma) C library in JavaScript via WASM bindings (e.g. via [vscode-oniguruma](https://github.com/microsoft/vscode-oniguruma)), this library is **much lighter weight** and its regexes **run much faster** since they run as native JavaScript.

> [!WARNING]
> This library is currently in alpha and has known bugs.

### [Try the demo REPL](https://slevithan.github.io/oniguruma-to-es/demo/)

Oniguruma-To-ES deeply understands all of the hundreds of large and small differences in Oniguruma and JavaScript regex syntax and behavior across multiple JavaScript version targets. It's *obsessive* about precisely following Oniguruma syntax rules and ensuring that the emulated features it supports have **exactly the same behavior**, even in extreme edge cases. And it's battle-tested on thousands of real-world Oniguruma regexes used in TextMate grammars via the Shiki library. A few uncommon features can't be perfectly emulated and allow rare differences, but if you don't want to allow this, you can disable the `allowBestEffort` option to throw for such patterns (see details below).

## 📜 Contents

- [Install and use](#️-install-and-use)
- [API](#-api)
- [Options](#-options)
- [Supported features](#-supported-features)
- [Unicode / mixed case-sensitivity](#️-unicode--mixed-case-sensitivity)

## 🕹️ Install and use

```sh
npm install oniguruma-to-es
```

```js
import {compile} from 'oniguruma-to-es';
```

In browsers:

```html
<script type="module">
  import {compile} from 'https://esm.run/oniguruma-to-es';
  compile(String.raw`…`);
</script>
```

<details>
  <summary>Using a global name (no import)</summary>

```html
<script src="https://cdn.jsdelivr.net/npm/oniguruma-to-es/dist/index.min.js"></script>
<script>
  const {compile} = OnigurumaToES;
</script>
```
</details>

## 🔑 API

### `compile`

Transpiles an Oniguruma regex pattern and flags to native JavaScript.

```ts
function compile(
  pattern: string,
  flags?: OnigurumaFlags,
  options?: CompileOptions
): {
  pattern: string;
  flags: string;
};
```

The returned `pattern` and `flags` can be provided directly to the JavaScript `RegExp` constructor. Various JavaScript flags might have been added or removed compared to the Oniguruma flags provided, as part of the emulation process.

#### Type `OnigurumaFlags`

A string with `i`, `m`, and `x` in any order (all optional).

> [!IMPORTANT]
> Oniguruma and JavaScript both have an `m` flag but with different meanings. Oniguruma's `m` is equivalent to JavaScript's `s` (`dotAll`).

#### Type `CompileOptions`

```ts
type CompileOptions = {
    allowBestEffort?: boolean;
    global?: boolean;
    hasIndices?: boolean;
    maxRecursionDepth?: number | null;
    optimize?: boolean;
    target?: 'ES2018' | 'ES2024' | 'ESNext';
};
```

See [Options](#-options) for more details.

### `toRegExp`

Transpiles an Oniguruma regex pattern and flags and returns a native JavaScript `RegExp`.

```ts
function toRegExp(
  pattern: string,
  flags?: OnigurumaFlags,
  options?: CompileOptions
): RegExp;
```

> [!TIP]
> Try it in the [demo REPL](https://slevithan.github.io/oniguruma-to-es/demo/).

### `toOnigurumaAst`

Generates an Oniguruma AST from an Oniguruma pattern and flags.

```ts
function toOnigurumaAst(
  pattern: string,
  flags?: OnigurumaFlags
): OnigurumaAst;
```

### `toRegexAst`

Generates a [`regex`](https://github.com/slevithan/regex) AST from an Oniguruma pattern and flags.

```ts
function toRegexAst(
  pattern: string,
  flags?: OnigurumaFlags
): RegexAst;
```

`regex`'s syntax and behavior is a strict superset of native JavaScript, so the AST is very close to representing native ESNext `RegExp` but with some added features (atomic groups, possessive quantifiers, recursion). The `regex` AST doesn't use some of `regex`'s extended features like flag `x` or subroutines because they follow PCRE behavior and work somewhat differently than in Oniguruma. The AST represents what's needed to precisely reproduce the Oniguruma behavior using `regex`.

## 🔩 Options

These options are shared by functions [`compile`](#compile) and [`toRegExp`](#toregexp).

### `allowBestEffort`

Allows results that differ from Oniguruma in rare cases. If `false`, throws if the pattern can't be emulated with identical behavior for the given `target`.

*Default: `true`.*

<details>
  <summary>More details</summary>

Specifically, this option enables the following additional features, depending on `target`:

- All targets (`ESNext` and earlier):
  - Enables use of `\X` using a close approximation of a Unicode extended grapheme cluster.
  - Enables recursion (e.g. via `\g<0>`) using a depth limit specified via option `maxRecursionDepth`.
- `ES2024` and earlier:
  - Enables use of case-insensitive backreferences to case-sensitive groups.
- `ES2018`:
  - Enables use of POSIX classes `[:graph:]` and `[:print:]` using ASCII-based versions rather than the Unicode versions available for `ES2024` and later. Other POSIX classes are always based on Unicode.
</details>

### `global`

Include JavaScript flag `g` (`global`) in results.

*Default: `false`.*

### `hasIndices`

Include JavaScript flag `d` (`hasIndices`) in results.

*Default: `false`.*

### `maxRecursionDepth`

If `null`, any use of recursion throws. If an integer between `2` and `100` (and `allowBestEffort` is `true`), common recursion forms are supported and recurse up to the specified max depth.

*Default: `6`.*

<details>
  <summary>More details</summary>

Using a high limit is not a problem if needed. Although there can be a performance cost (minor unless it's exacerbating an existing issue with runaway backtracking), there is no effect on regexes that don't use recursion.
</details>

### `optimize`

Simplify the generated pattern when it doesn't change the meaning.

*Default: `true`.*

### `target`

Sets the JavaScript language version for generated patterns and flags. Later targets allow faster processing, simpler generated source, and support for additional features.

*Default: `'ES2024'`.*

<details open>
  <summary>More details</summary>

- `ES2018`: Uses JS flag `u`.
  - Emulation restrictions: Character class intersection, nested negated character classes, and Unicode properties added after ES2018 are not allowed.
  - Generated regexes might use ES2018 features that require Node.js 10 or a browser version released during 2018 to 2023 (in Safari's case). Minimum requirement for any regex is Node.js 6 or a 2016-era browser.
- `ES2024`: Uses JS flag `v`.
  - No emulation restrictions.
  - Generated regexes require Node.js 20 or a 2023-era browser ([compat table](https://caniuse.com/mdn-javascript_builtins_regexp_unicodesets)).
- `ESNext`: Uses JS flag `v` and allows use of flag groups and duplicate group names.
  - Benefits: Faster transpilation, simpler generated source, and duplicate group names are preserved across separate alternation paths.
  - Generated regexes might use features that require Node.js 23 or a 2024-era browser (except Safari, which lacks support).
</details>

## ✅ Supported features

Following are the supported features by target.

> Targets `ES2024` and `ESNext` have the same emulation capabilities. Resulting regexes might have different source and flags, but they match the same strings.

Notice that nearly every feature below has at least subtle differences from JavaScript. Some features and subfeatures listed as unsupported are not emulatable using native JavaScript regexes, but support for others might be added in future versions of Oniguruma-To-ES. Unsupported features throw an error.

<table>
  <tr>
    <th colspan="2">Feature</th>
    <th>Example</th>
    <th>ES2018</th>
    <th>ES2024+</th>
    <th>Subfeatures &amp; JS differences</th>
  </tr>

  <tr valign="top">
    <th align="left" rowspan="3">Flags</th>
    <td><code>i</code></td>
    <td><code>i</code></td>
    <td align="middle">✅</td>
    <td align="middle">✅</td>
    <td>
      ✔ Unicode case folding (same as JS with flag <code>u</code>, <code>v</code>)<br>
    </td>
  </tr>
  <tr valign="top">
    <td><code>m</code></td>
    <td><code>m</code></td>
    <td align="middle">✅</td>
    <td align="middle">✅</td>
    <td>
      ✔ Equivalent to JS flag <code>s</code> (<code>dotAll</code>)<br>
    </td>
  </tr>
  <tr valign="top">
    <td><code>x</code></td>
    <td><code>x</code></td>
    <td align="middle">✅</td>
    <td align="middle">✅</td>
    <td>
      ✔ Unicode whitespace ignored<br>
      ✔ Line comments with <code>#</code><br>
      ✔ Whitespace/comments allowed between a token and its quantifier<br>
      ✔ Whitespace/comments between a quantifier and the <code>?</code>/<code>+</code> that makes it lazy/possessive changes it to a chained quantifier<br>
      ✔ Whitespace/comments separate tokens (ex: <code>\1 0</code>)<br>
      ✔ Whitespace and <code>#</code> not ignored in char classes<br>
    </td>
  </tr>

  <tr valign="top">
    <th align="left" rowspan="2" valign="top">Flag modifiers</th>
    <td>Group</td>
    <td><code>(?im-x:…)</code></td>
    <td align="middle">✅</td>
    <td align="middle">✅</td>
    <td>
      ✔ Unicode case folding for <code>i</code><br>
      ✔ Allows enabling and disabling the same flag (priority: disable)<br>
      ✔ Allows lone or multiple <code>-</code><br>
    </td>
  </tr>
  <tr valign="top">
    <td>Directive</td>
    <td><code>(?im-x)</code></td>
    <td align="middle">✅</td>
    <td align="middle">✅</td>
    <td>
      ✔ Continues until end of pattern or group (spanning alternatives)<br>
    </td>
  </tr>

  <tr valign="top">
    <th align="left" rowspan="10">Characters</th>
    <td>Literal</td>
    <td><code>E</code>, <code>!</code></td>
    <td align="middle">✅</td>
    <td align="middle">✅</td>
    <td>
      ✔ Code point based matching (same as JS with flag <code>u</code>, <code>v</code>)<br>
      ✔ Standalone <code>]</code>, <code>{</code>, <code>}</code> don't require escaping<br>
    </td>
  </tr>
  <tr valign="top">
    <td>Identity escape</td>
    <td><code>\E</code>, <code>\!</code></td>
    <td align="middle">✅</td>
    <td align="middle">✅</td>
    <td>
      ✔ Different allowed set than JS<br>
      ✔ Allows multibyte chars<br>
    </td>
  </tr>
  <tr valign="top">
    <td>Escaped metachar</td>
    <td><code>\\</code>, <code>\.</cpde></td>
    <td align="middle">✅</td>
    <td align="middle">✅</td>
    <td>
      ✔ Same as JS<br>
    </td>
  </tr>
  <tr valign="top">
    <td>Shorthand</td>
    <td><code>\t</code></td>
    <td align="middle">✅</td>
    <td align="middle">✅</td>
    <td>
      ✔ The JS set plus <code>\a</code>, <code>\e</code><br>
    </td>
  </tr>
  <tr valign="top">
    <td><code>\xNN</code></td>
    <td><code>\x7F</code></td>
    <td align="middle">✅</td>
    <td align="middle">✅</td>
    <td>
      ✔ Allows 1 hex digit<br>
      ✔ Above <code>7F</code>, is UTF-8 encoded byte (unlike JS)<br>
      ✔ Error for invalid encoded bytes<br>
    </td>
  </tr>
  <tr valign="top">
    <td><code>\uNNNN</code></td>
    <td><code>\uFFFF</code></td>
    <td align="middle">✅</td>
    <td align="middle">✅</td>
    <td>
      ✔ Same as JS with flag <code>u</code>, <code>v</code><br>
    </td>
  </tr>
  <tr valign="top">
    <td><code>\x{…}</code></td>
    <td><code>\x{A}</code></td>
    <td align="middle">✅</td>
    <td align="middle">✅</td>
    <td>
      ✔ Allows leading 0s up to 8 total hex digits<br>
    </td>
  </tr>
  <tr valign="top">
    <td>Escaped num</td>
    <td><code>\20</code></td>
    <td align="middle">✅</td>
    <td align="middle">✅</td>
    <td>
      ✔ Can be backref, error, null, octal, identity escape, or any of these combined with literal digits, based on complex rules that differ from JS<br>
      ✔ Always handles escaped single digit 1-9 outside char class as backref<br>
      ✔ Allows null with 1-3 0s<br>
      ✔ Error for octal > <code>177</code><br>
    </td>
  </tr>
  <tr valign="top">
    <td>Control</td>
    <td><code>\cA</code>, <code>\C-A</code></td>
    <td align="middle">✅</td>
    <td align="middle">✅</td>
    <td>
      ✔ With A-Za-z (JS: only <code>\c</code> form)<br>
    </td>
  </tr>
  <tr valign="top">
    <td colspan="2">Other (extremely rare)</td>
    <td align="middle">❌</td>
    <td align="middle">❌</td>
    <td>
      Not yet supported:<br>
      ● Non-A-Za-z with <code>\cx</code>, <code>\C-x</code><br>
      ● Meta <code>\M-x</code>, <code>\M-\C-x</code><br>
      ● Octal code point <code>\o{…}</code><br>
      ● UTF-8 encoded bytes in octal<br>
    </td>
  </tr>

  <tr valign="top">
    <th align="left" rowspan="7">Character sets</th>
    <td>Digit, word</td>
    <td><code>\d</code>, <code>\w</code>, etc.</td>
    <td align="middle">✅</td>
    <td align="middle">✅</td>
    <td>
      ✔ Same as JS (ASCII)<br>
    </td>
  </tr>
  <tr valign="top">
    <td>Hex digit</td>
    <td><code>\h</code>, <code>\H</code></td>
    <td align="middle">✅</td>
    <td align="middle">✅</td>
    <td>
      ✔ ASCII<br>
    </td>
  </tr>
  <tr valign="top">
    <td>Whitespace</td>
    <td><code>\s</code>, <code>\S</code></td>
    <td align="middle">✅</td>
    <td align="middle">✅</td>
    <td>
      ✔ ASCII (unlike JS)<br>
    </td>
  </tr>
  <tr valign="top">
    <td>Dot</td>
    <td><code>.</code></td>
    <td align="middle">✅</td>
    <td align="middle">✅</td>
    <td>
      ✔ Excludes only <code>\n</code> (unlike JS)<br>
    </td>
  </tr>
  <tr valign="top">
    <td>Any</td>
    <td><code>\O</code></td>
    <td align="middle">✅</td>
    <td align="middle">✅</td>
    <td>
      ✔ Any char (with any flags)<br>
      ✔ Identity escape in char class<br>
    </td>
  </tr>
  <tr valign="top">
    <td>Not newline</td>
    <td><code>\N</code></td>
    <td align="middle">✅</td>
    <td align="middle">✅</td>
    <td>
      ✔ Identity escape in char class<br>
    </td>
  </tr>
  <tr valign="top">
    <td>Unicode property</td>
    <td>
      <code>\p{L}</code>,<br>
      <code>\P{L}</code>
    </td>
    <td align="middle">✅<sup>[1]</sup></td>
    <td align="middle">✅</td>
    <td>
      ✔ Binary properties<br>
      ✔ Categories<br>
      ✔ Scripts<br>
      ✔ Aliases<br>
      ✔ POSIX properties<br>
      ✔ Negate with <code>\p{^…}</code>, <code>\P{^…}</code><br>
      ✔ Insignificant spaces, underscores, and casing in names<br>
      ✔ <code>\p</code>, <code>\P</code> without <code>{</code> is an identity escape<br>
      ✔ Error for key prefixes<br>
      ✔ Error for props of strings<br>
      ❌ Blocks (wontfix<sup>[2]</sup>)<br>
    </td>
  </tr>

  <tr valign="top">
    <th align="left" rowspan="2">Variable-length sets</th>
    <td>Newline</td>
    <td><code>\R</code></td>
    <td align="middle">✅</td>
    <td align="middle">✅</td>
    <td>
      ✔ Matched atomically<br>
    </td>
  </tr>
  <tr valign="top">
    <td>Grapheme</td>
    <td><code>\X</code></td>
    <td align="middle">☑️</td>
    <td align="middle">☑️</td>
    <td>
      ● Uses a close approximation<br>
      ✔ Matched atomically<br>
    </td>
  </tr>

  <tr valign="top">
    <th align="left" rowspan="6">Character classes</th>
    <td>Base</td>
    <td><code>[…]</code>, <code>[^…]</code></td>
    <td align="middle">✅</td>
    <td align="middle">✅</td>
    <td>
      ✔ Unescaped <code>-</code> outside of range is literal in some contexts (different than JS rules in any mode)<br>
      ✔ Fewer chars require escaping than JS<br>
      ✔ Error for reversed range (same as JS)<br>
    </td>
  </tr>
  <tr valign="top">
    <td>Empty</td>
    <td><code>[]</code>, <code>[^]</code></td>
    <td align="middle">✅</td>
    <td align="middle">✅</td>
    <td>
      ✔ Error<br>
    </td>
  </tr>
  <tr valign="top">
    <td>Range</td>
    <td><code>[a-z]</code></td>
    <td align="middle">✅</td>
    <td align="middle">✅</td>
    <td>
      ✔ Same as JS with flag <code>u</code>, <code>v</code><br>
    </td>
  </tr>
  <tr valign="top">
    <td>POSIX class</td>
    <td>
      <code>[[:word:]]</code>,<br>
      <code>[[:^word:]]</code>
    </td>
    <td align="middle">☑️<sup>[3]</sup></td>
    <td align="middle">✅</td>
    <td>
      ✔ All use Unicode definitions<br>
    </td>
  </tr>
  <tr valign="top">
    <td>Nested class</td>
    <td><code>[…[…]]</code></td>
    <td align="middle">☑️<sup>[4]</sup></td>
    <td align="middle">✅</td>
    <td>
      ✔ Same as JS with flag <code>v</code><br>
    </td>
  </tr>
  <tr valign="top">
    <td>Intersection</td>
    <td><code>[…&amp;&amp;…]</code></td>
    <td align="middle">❌</td>
    <td align="middle">✅</td>
    <td>
      ✔ Doesn't require nested classes for union and ranges<br>
    </td>
  </tr>

  <tr valign="top">
    <th align="left" rowspan="6">Assertions</th>
    <td>Line start, end</td>
    <td><code>^</code>, <code>$</code></td>
    <td align="middle">✅</td>
    <td align="middle">✅</td>
    <td>
      ✔ Always "multiline" (per JS)<br>
      ✔ Only <code>\n</code> as newline<br>
    </td>
  </tr>
  <tr valign="top">
    <td>String start, end</td>
    <td><code>\A</code>, <code>\z</code></td>
    <td align="middle">✅</td>
    <td align="middle">✅</td>
    <td>
      ✔ Like JS <code>^</code> <code>$</code> without JS flag <code>m</code><br>
    </td>
  </tr>
  <tr valign="top">
    <td>String end or before terminating newline</td>
    <td><code>\Z</code></td>
    <td align="middle">✅</td>
    <td align="middle">✅</td>
    <td>
      ✔ Only <code>\n</code> as newline<br>
    </td>
  </tr>
  <tr valign="top">
    <td>Search start</td>
    <td><code>\G</code></td>
    <td align="middle">☑️</td>
    <td align="middle">☑️</td>
    <td>
      ● Supported when used at start of pattern (if no top-level alternation) and when at start of all top-level alternatives<br>
    </td>
  </tr>
  <tr valign="top">
    <td>Word boundary</td>
    <td><code>\b</code>, <code>\B</code></td>
    <td align="middle">✅</td>
    <td align="middle">✅</td>
    <td>
      ✔ Unicode based (unlike JS)<br>
    </td>
  </tr>
  <tr valign="top">
    <td>Lookaround</td>
    <td>
      <code>(?=…)</code>,<br>
      <code>(?!…)</code>,<br>
      <code>(?&lt;=…)</code>,<br>
      <code>(?&lt;!…)</code>
    </td>
    <td align="middle">✅</td>
    <td align="middle">✅</td>
    <td>
      ✔ Same as JS<br>
      ✔ Allows variable-length quantifiers and alternation within lookbehind<br>
    </td>
  </tr>

  <tr valign="top">
    <th align="left" rowspan="3">Quantifiers</th>
    <td>Greedy, lazy</td>
    <td><code>*</code>, <code>+?</code>, <code>{2,}</code>, etc.</td>
    <td align="middle">✅</td>
    <td align="middle">✅</td>
    <td>
      ✔ Includes all JS forms<br>
      ✔ Adds form <code>{,n}</code> for implicit min 0<br>
      ✔ Explicit bounds have upper limit of 100,000 (unlimited in JS)<br>
      ✔ Error with assertions (same as JS with flag <code>u</code>, <code>v</code>)<br>
    </td>
  </tr>
  <tr valign="top">
    <td>Possessive</td>
    <td><code>?+</code>, <code>*+</code>, <code>++</code></td>
    <td align="middle">✅</td>
    <td align="middle">✅</td>
    <td>
      ✔ <code>+</code> suffix doesn't make interval (<code>{…}</code>) quantifiers possessive (creates a chained quantifier)<br>
    </td>
  </tr>
  <tr valign="top">
    <td>Chained</td>
    <td><code>**</code>, <code>??+*</code>, <code>{2,3}+</code>, etc.</td>
    <td align="middle">✅</td>
    <td align="middle">✅</td>
    <td>
      ✔ Further repeats the preceding repetition<br>
    </td>
  </tr>

  <tr valign="top">
    <th align="left" rowspan="4">Groups</th>
    <td>Noncapturing</td>
    <td><code>(?:…)</code></td>
    <td align="middle">✅</td>
    <td align="middle">✅</td>
    <td>
      ✔ Same as JS<br>
    </td>
  </tr>
  <tr valign="top">
    <td>Atomic</td>
    <td><code>(?>…)</code></td>
    <td align="middle">✅</td>
    <td align="middle">✅</td>
    <td>
      ✔ Supported<br>
    </td>
  </tr>
  <tr valign="top">
    <td>Capturing</td>
    <td><code>(…)</code></td>
    <td align="middle">✅</td>
    <td align="middle">✅</td>
    <td>
      ✔ Is noncapturing if named capture present<br>
    </td>
  </tr>
  <tr valign="top">
    <td>Named capturing</td>
    <td>
      <code>(?&lt;a>…)</code>,<br>
      <code>(?'a'…)</code>
    </td>
    <td align="middle">✅</td>
    <td align="middle">✅</td>
    <td>
      ✔ Duplicate names allowed (including within the same alternation path) unless directly referenced by a subroutine<br>
      ✔ Error for names invalid in Oniguruma or JS<br>
    </td>
  </tr>

  <tr valign="top">
    <th align="left" rowspan="4">Backreferences</th>
    <td>Numbered</td>
    <td><code>\1</code></td>
    <td align="middle">✅</td>
    <td align="middle">✅</td>
    <td>
      ✔ Error if named capture used<br>
      ✔ Refs the most recent of a capture/subroutine set<br>
    </td>
  </tr>
  <tr valign="top">
    <td>Enclosed numbered, relative</td>
    <td>
      <code>\k&lt;1></code>,<br>
      <code>\k'1'</code>,<br>
      <code>\k&lt;-1></code>,<br>
      <code>\k'-1'</code>
    </td>
    <td align="middle">✅</td>
    <td align="middle">✅</td>
    <td>
      ✔ Error if named capture used<br>
      ✔ Allows leading 0s<br>
      ✔ Refs the most recent of a capture/subroutine set<br>
    </td>
  </tr>
  <tr valign="top">
    <td>Named</td>
    <td>
      <code>\k&lt;a></code>,<br>
      <code>\k'a'</code>
    </td>
    <td align="middle">✅</td>
    <td align="middle">✅</td>
    <td>
      ✔ For duplicate group names, rematch any of their matches (multiplex)<br>
      ✔ Refs the most recent of a capture/subroutine set (no multiplex)<br>
      ✔ Combination of multiplex and most recent of capture/subroutine set if duplicate name is indirectly created by a subroutine<br>
    </td>
  </tr>
  <tr valign="top">
    <td colspan="2">To nonparticipating groups</td>
    <td align="middle">☑️</td>
    <td align="middle">☑️</td>
    <td>
      ✔ Error if group to the right<sup>[5]</sup><br>
      ✔ Duplicate names (and subroutines) to the right not included in multiplex<br>
      ✔ Fail to match (or don't include in multiplex) ancestor groups and groups in preceding alternation paths<br>
      ❌ Some rare cases are indeterminable at compile time and use the JS behavior of matching an empty string<br>
    </td>
  </tr>

  <tr valign="top">
    <th align="left" rowspan="2">Subroutines</th>
    <td>Numbered, relative</td>
    <td>
      <code>\g&lt;1></code>,<br>
      <code>\g'1'</code>,<br>
      <code>\g&lt;-1></code>,<br>
      <code>\g'-1'</code>,<br>
      <code>\g&lt;+1></code>,<br>
      <code>\g'+1'</code>
    </td>
    <td align="middle">✅</td>
    <td align="middle">✅</td>
    <td>
      ✔ Allowed before reffed group<br>
      ✔ Can be nested (any depth)<br>
      ✔ Doesn't alter backref nums<br>
      ✔ Reuses flags from the reffed group (ignores local flags)<br>
      ✔ Replaces most recent captured values (for backrefs)<br>
      ✔ Error if named capture used<br>
    </td>
  </tr>
  <tr valign="top">
    <td>Named</td>
    <td>
      <code>\g&lt;a></code>,<br>
      <code>\g'a'</code>
    </td>
    <td align="middle">✅</td>
    <td align="middle">✅</td>
    <td>
      ● Same behavior as numbered<br>
      ✔ Error if reffed group uses duplicate name<br>
    </td>
  </tr>

  <tr valign="top">
    <th align="left" rowspan="3">Recursion</th>
    <td>Full pattern</td>
    <td>
      <code>\g&lt;0></code>,<br>
      <code>\g'0'</code>
    </td>
    <td align="middle">☑️</td>
    <td align="middle">☑️</td>
    <td>
      ● Limited support<sup>[6]</sup><br>
    </td>
  </tr>
  <tr valign="top">
    <td>Numbered, relative</td>
    <td><code>(…\g&lt;1>?…)</code>, etc.</td>
    <td align="middle">❌</td>
    <td align="middle">❌</td>
    <td>
      ● Not yet supported<br>
    </td>
  </tr>
  <tr valign="top">
    <td>Named</td>
    <td><code>(?&lt;a>…\g&lt;a>?…)</code>, etc.</td>
    <td align="middle">☑️</td>
    <td align="middle">☑️</td>
    <td>
      ● Limited support<sup>[6]</sup><br>
    </td>
  </tr>

  <tr valign="top">
    <th align="left" rowspan="8">Other</th>
    <td>Comment group</td>
    <td><code>(?#…)</code></td>
    <td align="middle">✅</td>
    <td align="middle">✅</td>
    <td>
      ✔ Allows escaping <code>\)</code>, <code>\\</code><br>
      ✔ Comments allowed between a token and its quantifier<br>
      ✔ Comments between a quantifier and the <code>?</code>/<code>+</code> that makes it lazy/possessive changes it to a chained quantifier<br>
    </td>
  </tr>
  <tr valign="top">
    <td>Alternation</td>
    <td><code>…|…</code></td>
    <td align="middle">✅</td>
    <td align="middle">✅</td>
    <td>
      ✔ Same as JS<br>
    </td>
  </tr>
  <tr valign="top">
    <td>Keep</td>
    <td><code>\K</code></td>
    <td align="middle">☑️</td>
    <td align="middle">☑️</td>
    <td>
      ● Supported if at top level and no top-level alternation is used<br>
    </td>
  </tr>
  <tr valign="top">
    <td>Absence operator</td>
    <td><code>(?~…)</code></td>
    <td align="middle">❌</td>
    <td align="middle">❌</td>
    <td>
      ● Some forms are supportable<br>
    </td>
  </tr>
  <tr valign="top">
    <td>Conditional</td>
    <td><code>(?(1)…)</code></td>
    <td align="middle">❌</td>
    <td align="middle">❌</td>
    <td>
      ● Some forms are supportable<br>
    </td>
  </tr>
  <tr valign="top">
    <td>Char sequence</td>
    <td>
      <code>\x{1 2 …N}</code>,<br>
      <code>\o{1 2 …N}</code><br>
    </td>
    <td align="middle">❌</td>
    <td align="middle">❌</td>
    <td>
      ● Not yet supported<br>
    </td>
  </tr>
  <tr valign="top">
    <td colspan="2">JS features unknown to Oniguruma are handled using Oniguruma syntax</td>
    <td align="middle">✅</td>
    <td align="middle">✅</td>
    <td>
      ✔ <code>\u{…}</code> is an error<br>
      ✔ <code>[\q{…}]</code> matches <code>q</code>, etc.<br>
      ✔ <code>[a--b]</code> includes the invalid reversed range <code>a</code> to <code>-</code><br>
    </td>
  </tr>
  <tr valign="top">
    <td colspan="2">Invalid Oniguruma syntax</td>
    <td align="middle">✅</td>
    <td align="middle">✅</td>
    <td>
      ✔ Error<br>
    </td>
  </tr>
</table>

The table above doesn't include all aspects that Oniguruma-To-ES emulates (including error handling, most aspects that work the same as in JavaScript, and many aspects of non-JavaScript features that work the same in the other regex flavors that support them).

### Footnotes

1. Target `ES2018` doesn't allow Unicode property names added in JavaScript specifications after ES2018 to be used.
2. Unicode blocks are easily emulatable but their character data would significantly increase library weight. They're also a deeply flawed and arguably-unuseful feature, given the ability to use Unicode scripts and other properties instead.
3. With target `ES2018`, the specific POSIX classes `[:graph:]` and `[:print:]` are an error if option `allowBestEffort` is `false`, and they use ASCII-based versions rather than the Unicode versions available for target `ES2024` and later.
4. Target `ES2018` doesn't support nested negated character classes.
5. It's not an error for *numbered* backreferences to come before their referenced group in Oniguruma, but an error is the best path for Oniguruma-To-ES because (1) most placements are mistakes and can never match (based on the Oniguruma behavior for backreferences to nonparticipating groups), (2) erroring matches the behavior of named backreferences, and (3) the edge cases where they're matchable rely on rules for backreference resetting within quantified groups that are different in JS and aren't emulatable. Note that it's not a backreference in the first place if using `\10` or higher and not as many capturing groups are defined to the left (it's an octal or identity escape).
6. Recursion depth is limited, and specified by option `maxRecursionDepth`. Any use of recursion results in an error if `maxRecursionDepth` is `null` or `allowBestEffort` is `false`. Additionally, some forms of recursion are not yet supported, including mixing recursion with backreferences, using multiple recursions in the same pattern, and recursion by group number. Because recursion is bounded, patterns that fail due to infinite recursion in Oniguruma might find a match in Oniguruma-To-ES. Future versions will detect this and throw an error.

## ㊗️ Unicode / mixed case-sensitivity

Oniguruma-To-ES fully supports mixed case-sensitivity (and handles the Unicode edge cases) regardless of JavaScript [target](#target). It also restricts Unicode properties to those supported by Oniguruma and the target JavaScript version.

Oniguruma-To-ES focuses on being lightweight to make it better for use in browsers. This is partly achieved by not including heavyweight Unicode character data, which imposes a couple of minor/rare restrictions:

- Character class intersection and nested negated character classes are unsupported with target `ES2018`. Use target `ES2024` or later if you need support for these Oniguruma features.
- With targets before `ESNext`, a handful of Unicode properties that target a specific character case (ex: `\p{Lower}`) can't be used case-insensitively in patterns that contain other characters with a specific case that are used case-sensitively.
  - In other words, almost every usage is fine, including `A\p{Lower}`, `(?i:A\p{Lower})`, `(?i:A)\p{Lower}`, `(?i:A(?-i:\p{Lower}))`, and `\w(?i:\p{Lower})`, but not `A(?i:\p{Lower})`.
  - Using these properties case-insensitively is basically never done intentionally, so you're unlikely to encounter this error unless it's catching a mistake.

## 👀 Similar projects

[JsRegex](https://github.com/jaynetics/js_regex) transpiles [Onigmo](https://github.com/k-takata/Onigmo) regexes to JavaScript (Onigmo is a fork of Oniguruma with mostly shared syntax and behavior). It's written in Ruby and relies on the [Regexp::Parser](https://github.com/ammar/regexp_parser) Ruby gem, which means regexes must be pre-transpiled on the server to use them in JavaScript. Note that JsRegex doesn't always translate edge case behavior differences.

## 🏷️ About

Oniguruma-To-ES was created by [Steven Levithan](https://github.com/slevithan).

If you want to support this project, I'd love your help by contributing improvements, sharing it with others, or [sponsoring](https://github.com/sponsors/slevithan) ongoing development.

© 2024–present. MIT License.
