# iRule Checker

Static analysis for F5 BIG-IP iRules. Paste a rule, get the problems back with line
numbers before you go anywhere near `tmsh load`.

It runs entirely in the browser. Nothing is uploaded, and there are no dependencies —
useful when the rule you are debugging contains customer hostnames.


## What it checks

The first pass is a real TCL scanner. It tracks brace, bracket and quote state the way
TMOS parses a rule, which means it gets the awkward cases right:

- `#` only starts a comment at the beginning of a command, so a trailing `# note` is an
  argument and gets flagged
- backslash escapes suppress brace counting
- a brace inside a quoted string still counts toward matching within an event body

Comments and string bodies are blanked out before the pattern rules run, so nothing
fires on code that only appears inside a string.

On top of that sit a structural pass — top-level statements, event blocks, nesting —
and about 25 rules.

### Rules

| ID | Severity | What it means |
| --- | --- | --- |
| `TCL-UNCLOSED-BRACE` | error | A `{` is never closed. Reported at the line it opened. |
| `TCL-UNCLOSED-BRACKET` | error | A `[` is never closed. |
| `TCL-UNCLOSED-QUOTE` | error | A quoted string is never closed. |
| `TCL-UNMATCHED-CLOSE` | error | More `}` than `{`. |
| `TCL-STRAY-BRACKET` | info | A `]` with no opening `[`; TCL reads it as a literal. |
| `TCL-BRACE-IN-STRING` | info | Braces inside a string are affecting the matching. |
| `TCL-DANGLING-ELSE` | error | `else` or `elseif` starts a line, so `if` never sees it. |
| `TCL-ELSE-IF` | error | `else if` instead of `elseif`. |
| `TCL-IF-PAREN` | error | `if (...)` instead of `if { ... }`. |
| `TCL-NO-SPACE-BRACES` | error | `}{` with no space; TCL reads it as one word. |
| `TCL-BRACE-NEWLINE` | error | The opening brace was pushed to the next line. |
| `TCL-UNBRACED-COND` | warning | A condition without braces around it. |
| `TCL-UNBRACED-EXPR` | warning | `expr` without braces: slower, and unsafe on user input. |
| `TCL-SET-DOLLAR` | error | `set $name value` sets the wrong variable. |
| `TCL-INLINE-COMMENT` | warning | A trailing `#` that TCL parses as an argument. |
| `IRULE-UNKNOWN-EVENT` | error | Not a BIG-IP event, with a spelling suggestion. |
| `IRULE-DUPLICATE-EVENT` | error | The same event is bound twice in one rule. |
| `IRULE-NESTED-WHEN` | error | `when` inside another block. |
| `IRULE-TOP-LEVEL` | error | A statement outside any event block or proc. |
| `IRULE-WHEN-SYNTAX` | error | Malformed `when` line. |
| `IRULE-NO-EVENTS` | warning | The rule binds nothing, so it never runs. |
| `IRULE-UNKNOWN-NAMESPACE` | error | `HTPP::` and friends. |
| `IRULE-UNKNOWN-COMMAND` | info | Not in the known command list — a hint, not a verdict. |
| `IRULE-WRONG-EVENT` | warning | `HTTP::` in `CLIENT_ACCEPTED`, and similar mismatches. |
| `IRULE-REQ-IN-RESPONSE` | warning | Reading `HTTP::uri` in `HTTP_RESPONSE`. |
| `IRULE-STATUS-IN-REQUEST` | warning | Reading `HTTP::status` on the request side. |
| `IRULE-LB-TOO-EARLY` | info | `LB::server` before a pool member is chosen. |
| `IRULE-NO-RETURN` | warning | Execution continues past `HTTP::respond` or `HTTP::redirect`. |
| `IRULE-COLLECT-NO-RELEASE` | warning | `HTTP::collect` with no `HTTP::release`. |
| `IRULE-TCP-COLLECT` | warning | `TCP::collect` with no `TCP::release`. |
| `IRULE-PAYLOAD-NO-COLLECT` | warning | `HTTP::payload` read without collecting first. |
| `IRULE-INIT-GLOBAL` | warning | A `RULE_INIT` variable outside the `static::` namespace. |
| `IRULE-STATIC-NO-INIT` | info | `static::` values read but never initialised. |
| `IRULE-CLASS-MATCH-ARGS` | warning | `class match` missing its operator or class name. |
| `IRULE-DEPRECATED-MATCHCLASS` | warning | Use `class match`. |
| `IRULE-DEPRECATED-FINDCLASS` | warning | Use `class search -value`. |
| `IRULE-TABLE-TIMEOUT` | info | `table set` with no timeout leaks memory until TMM restarts. |
| `IRULE-PROC` | info | Procedures need TMOS 11.4 or later. |
| `IRULE-REGEX` | style | `regexp`/`regsub` on every connection is expensive. |
| `IRULE-LOG-FACILITY` | style | `log` without a facility. |
| `IRULE-STRING-COMPARE` | info | `==` on strings; prefer `eq` or `equals`. |
| `FILE-CRLF` | style | Windows line endings. |

### What it cannot do

It has no access to your device, so it cannot tell you whether a pool, data group,
profile or virtual server exists. The command list covers the common namespaces but not
every TMOS version, which is why an unrecognised command is a note rather than an error.
Treat a clean report as permission to load the rule in a lab, not as a guarantee.

## Running it

Two ways, depending on what you need.

**Single file.** `dist/irule-checker.html` has everything inlined. Download it, open it
in a browser, done — no server, no network, fine on an air-gapped jump box. Web fonts are
the only thing it fetches, and it falls back to system fonts without them.

**From source.** ES modules will not load over `file://`, so use the bundled dev server:

```bash
git clone https://github.com/your-username/irule-checker.git
cd irule-checker
npm start           # http://localhost:8080
```

There is no `npm install` step. The project has no dependencies at all.

```bash
npm test            # node --test, 26 assertions
npm run build       # regenerate dist/irule-checker.html
npm run check       # both
```

Node 18 or newer.

## Layout

```
index.html            markup for the dev build
assets/
  packet2pipeline.png the owner mark in the header
  favicon.png
src/
  styles.css
  data/
    events.js         BIG-IP events and their context families
    commands.js       namespaces and known commands
    samples.js        the examples in the picker
  core/
    scanner.js        TCL-aware brace/bracket/quote/comment scan
    structure.js      top-level statements and event blocks
    lint.js           the line rules
    analyse.js        runs every pass, sorts, de-duplicates
    util.js           finding constructor, edit distance, escaping
  ui/
    highlight.js      the syntax colouring layer
    app.js            editor, gutter, findings list
test/                 node:test, imports src/ directly
tools/
  build.js            inlines everything into dist/
  serve.js            static server for development
```

Images in `assets/` are inlined as data URIs at build time, which is why the single-file
build has no external references at all.

`core/` never touches the DOM, so the same modules run in the browser, in the tests, and
in anything else you want to plug them into:

```js
import { analyse } from './src/core/analyse.js';

const { findings } = analyse(readFileSync('rule.tcl', 'utf8'));
for (const f of findings) {
  console.log(`${f.sev} ${f.line}:${f.col} [${f.id}] ${f.msg.replace(/<[^>]+>/g, '')}`);
}
```

## Adding a rule

1. Add the check to `src/core/lint.js`. Every finding needs a stable ID, a severity, a
   message that says what is wrong, and a hint that says what to do about it.
2. Add a test to `test/rules.test.js` — one case that fires it, and one close-but-correct
   case that must stay silent. False positives are the fastest way to make a linter
   useless, so the negative case matters more than the positive one.
3. Add a row to the table above.

`npm run check` before you open a PR. See [CONTRIBUTING.md](CONTRIBUTING.md).

## Licence

Not affiliated with or endorsed by F5, Inc. "F5", "BIG-IP" and "iRules" are trademarks of
F5, Inc.
