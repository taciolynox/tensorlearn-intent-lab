# Intent

## Goal
A support coordinator needs to preview the tickets in a CSV file before anyone approves a
later import. Add a pure function that takes CSV text and returns either the list of
tickets that would be imported, in input order, or a single error code that explains why
the file cannot be imported. The preview must never change data: it only reads the input
string and returns a plain result object. The real import is out of scope for this
change.

## Scope
Create only two new files: `src/import-tickets.mjs`, which exports the preview function
(proposed name `previewTickets(csvText)`, to be confirmed before implementation), and
`tests/import-tickets.test.mjs`, which covers every acceptance case below using the
built-in `node:test` runner. Excluded: any UI, database, network calls, file reads or
writes, CLI entry points, package dependencies, and the import itself. Do not edit
`src/tickets.mjs`, `tests/tickets.test.mjs`, anything under `.github/`, `CLAUDE.md`,
`package.json`, or the fixtures. If the work seems to need any other file, stop and ask
instead of expanding scope.

## Constraints
The existing `ticket(title, priority)` in `src/tickets.mjs` is the stable ticket
representation. Import and reuse it, but do not modify it. Rules:

- The header must contain `title` and may contain `priority`, in either order. A missing
  `title`, any other column name, a duplicate column name, an empty input with no header,
  or a data row whose field count differs from the header all return `unknown-column`.
- Trim surrounding whitespace from every value. An empty title after trimming returns
  `empty-title`.
- Priority must be exactly `low`, `normal` or `urgent`. A missing or blank priority
  becomes `normal`. Anything else returns `invalid-priority`.
- At most 100 data rows are accepted. Count rows after the header is validated and
  before any row values are checked, so 101 rows return `row-limit` even if a row is
  also invalid.
- Respect standard CSV quoting: a quoted field may contain commas, and a doubled quote
  inside a quoted field means a literal quote.
- Success is exactly `{"tickets":[...]}`, with each ticket as `{"title","priority"}` in
  input order. Failure is exactly `{"error":"<code>"}`. A failure never includes partial
  tickets. Use plain JavaScript with no new dependencies, and do not throw to callers.

## Plan
1. Before writing code, restate the scope and list any open questions. Wait for my
   approval. Open questions: the exported function name; whether a header-only file
   returns `{"tickets":[]}`; whether CRLF line endings and one trailing newline are
   accepted; whether blank lines are ignored or counted as malformed rows; and whether
   header names are trimmed and case-sensitive.
2. Write `tests/import-tickets.test.mjs` first, with one test per acceptance case in the
   JSON contract below plus any approved answers to the open questions.
3. Implement a small CSV tokenizer in `src/import-tickets.mjs` that handles quoted
   fields, embedded commas, escaped quotes and line endings.
4. Validate in this order: header, then row count, then each row's field count, title
   and priority. Build tickets through `ticket()`, mapping its thrown error to the
   result object.
5. Run the verification commands, report the results, and show the diff. Do not commit,
   push or open a PR unless I ask.

## Acceptance
The feature is complete when the exported function returns exactly the `expected`
object for every case below. Together, the cases cover an explicit priority, the default
priority, a quoted title containing a comma, an empty title, an unknown column, an
invalid priority, and too many rows. A case with `"repeat": 101` means the header
followed by 101 copies of its single data row. The cases must also pass as automated
tests in `tests/import-tickets.test.mjs`, and `src/tickets.mjs` must be byte-for-byte
unchanged.

```json
{
  "schema": 1,
  "change": "ticket-csv-preview",
  "allowed_changes": ["src/import-tickets.mjs", "tests/import-tickets.test.mjs"],
  "protected_paths": ["src/tickets.mjs", ".github/", "CLAUDE.md"],
  "non_goals": ["ui", "database", "network"],
  "requirements": {
    "columns": ["title", "priority"],
    "default_priority": "normal",
    "valid_priorities": ["low", "normal", "urgent"],
    "reject_unknown_columns": true,
    "reject_empty_title": true,
    "max_rows": 100,
    "side_effects": "preview-only"
  },
  "verification": ["npm test", "git diff --check"],
  "approval": "ask-before-implementation",
  "acceptance_cases": [
    {
      "csv": "title,priority\nPrinter issue,urgent\n",
      "expected": {"tickets":[{"title":"Printer issue","priority":"urgent"}]}
    },
    {
      "csv": "title\nBroken laptop\n",
      "expected": {"tickets":[{"title":"Broken laptop","priority":"normal"}]}
    },
    {
      "csv": "title,priority\nVPN down,\n",
      "expected": {"tickets":[{"title":"VPN down","priority":"normal"}]}
    },
    {
      "csv": "title,priority\n\"Printer, second floor\",low\n",
      "expected": {"tickets":[{"title":"Printer, second floor","priority":"low"}]}
    },
    {
      "csv": "title,priority\n   ,urgent\n",
      "expected": {"error":"empty-title"}
    },
    {
      "csv": "title,owner\nPrinter issue,alex\n",
      "expected": {"error":"unknown-column"}
    },
    {
      "csv": "title,priority\nPrinter issue,high\n",
      "expected": {"error":"invalid-priority"}
    },
    {
      "csv": "title\nPrinter issue\n",
      "repeat": 101,
      "expected": {"error":"row-limit"}
    }
  ]
}
```

## Verification
Run `npm test`. It must pass all existing ticket tests and every new import test. Then
run `git diff --check`, which must report no whitespace errors, and confirm with
`git status` that only the two allowed files were added. Ask for my approval before
starting the implementation, and again before touching any file outside the allowed
list, adding a dependency, or changing behavior beyond these rules. Report any failing
command with its output instead of working around it.
