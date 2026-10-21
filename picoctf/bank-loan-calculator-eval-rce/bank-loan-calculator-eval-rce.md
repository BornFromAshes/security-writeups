# Bank-Loan Calculator — Python `eval()` Sandbox Bypass to RCE

**Category:** Web Exploitation / Server-Side Code Injection
**Platform:** CyLab Security Academy
**Difficulty:** Medium

## Challenge Description

The target is a "Bank-Loan Calculator" web app that lets a user submit a
math-like formula (e.g. `PRT*RATE*TIME(10000*23*12)`) which is evaluated
server-side and the result returned to the page. The app blocks a list of
dangerous keywords and uses a regex filter to block suspicious-looking file
paths before evaluating the input.

**Goal:** achieve remote code execution and read `/flag.txt`.

## Recon

Before trying to break anything, I confirmed that the input field actually
reaches a server-side evaluator rather than being evaluated client-side or
just doing string matching.

- Submitted `1==1` → the app returned `True`. This confirms the input is
  passed to something that evaluates Python expressions, not just parses a
  math formula.
- Submitted `'abc'.upper()` → returned `ABC`, and `str(123)` → returned
  `123`. Both are valid Python but invalid in most other scripting
  languages' default syntax for this context, which told me the backend is
  Python, and specifically that string/object methods were being executed
  (strongly suggesting a raw `eval()` call rather than a restricted math
  parser).

## Attempt 1: Direct `os` command execution (blocked)

The natural next step from confirmed `eval()` is OS command execution:

```python
__import__('os').popen('ls -la').read()
```

This was rejected. Inspecting the error responses, the app has a
**keyword blocklist** — it checks the submitted code string for banned
substrings before evaluating it. Blocked words included things like `os`,
`eval`, `exec`, `ls`, and similar command/module names.

## Attempt 2: String concatenation to dodge the blocklist (partial success)

Since the blocklist appeared to be a simple substring check, I tried
splitting the banned words so the literal substring never appears in the
submitted code, only at runtime after Python concatenates it:

```python
__import__('o'+'s').popen('l'+'s').read()
```

This worked and executed `ls`, returning `app.py static`.

However, trying to go further — for example listing the root filesystem
with `str(__import__('o'+'s').listdir('/'))` — was blocked again. This
pointed to a **second filter**: a regex specifically targeting file-path-like
strings (the challenge source, recovered later, confirms this — a
`FILE_PATH_REGEX` matching patterns like `/`, drive letters, and
Windows-style paths). So concatenation defeated the keyword blocklist but
not the path regex, since the literal `/` character still appeared in the
submitted code.

## Attempt 3: Encode everything as character codes (full bypass)

To beat a filter that inspects the *literal text* of the submitted code, the
cleanest approach is to never write the sensitive substrings or characters
in the submission at all — instead submit numeric character codes and
reconstruct the strings at runtime with `chr()`, which Python evaluates
only when the code actually runs, after the filter has already approved it.

First, I used this to read the backend source itself:

```python
open(chr(97)+chr(112)+chr(112)+chr(46)+chr(112)+chr(121)).read()
```
(`chr(97)+chr(112)+chr(112)+chr(46)+chr(112)+chr(121)` spells `app.py`)

This returned the full Flask source. It confirmed the mechanism directly:
the `/execute` route takes `request.form['code']`, checks it against
`BLOCKLIST_KEYWORDS` and `FILE_PATH_REGEX` with `re.search()`, and if
neither matches, runs `eval(code)` and returns the result. In other words,
the filtering happens entirely on the **submitted string**, and has no
visibility into what the code resolves to once it executes — exactly the
gap the `chr()` encoding exploits.

With that confirmed, I listed the root directory the same way:

```python
str(__import__(chr(111)+chr(115)).listdir(chr(47)))
```
(`chr(111)+chr(115)` = `os`, `chr(47)` = `/`)

This returned the directory listing, which included `flag.txt` at the root.

Finally, I read the flag:

```python
open(chr(47)+chr(102)+chr(108)+chr(97)+chr(103)+chr(46)+chr(116)+chr(120)+chr(116)).read()
```
(decodes to `/flag.txt`)

This returned the flag content, completing the challenge.

## Root Cause

The application tries to sandbox a Python `eval()` call using **blocklist
filtering on the raw input string**: a keyword list and a regex over file
paths. This is a textbook example of trying to secure `eval()` with a
denylist, which is fundamentally unreliable because:

1. **Denylists only catch patterns you thought of.** Python has many
   equivalent ways to express the same operation (string concatenation,
   `chr()`/`ord()`, string formatting, `getattr`, bytes, base64, etc.), and
   enumerating all of them is not realistically possible.
2. **The filter inspects syntax, not semantics.** It never actually
   evaluates what the code *will do* — only whether the submitted text
   contains certain substrings. Anything that defers string construction to
   runtime (like `chr()`) is invisible to a text-based filter.
3. **`eval()` on user input is inherently unsafe**, regardless of
   filtering. There is no reliable way to let a user submit arbitrary
   Python and only allow "safe" execution with a blocklist approach.

## Fix

- **Never call `eval()` on user-controlled input.** For a loan-formula
  calculator, the correct approach is a proper expression parser limited to
  arithmetic operations and a fixed set of named variables (e.g. Python's
  `ast.literal_eval` for literals, or a dedicated math expression library
  such as `asteval` or a hand-written recursive-descent parser that only
  supports `+ - * / ( )` and known variable names).
- If arbitrary code execution is truly required, run it in an isolated,
  resource-limited sandbox (separate container/VM with no filesystem access
  and no network), not in-process with the web server.
- Denylisting is not a substitute for a safe-by-construction design —
  allowlisting the exact grammar you intend to support is the only
  reliable fix here.

## Tools Used

- Firefox DevTools / manual form submission to test payloads
- Python built-ins (`chr`, `__import__`, string concatenation) as the
  actual exploitation primitives — no external tools were needed since the
  vulnerability was entirely in application logic

## Key Takeaway

Blocklist-based filtering of code before `eval()`/`exec()` is nearly always
bypassable, because the filter only ever sees the *text* of the payload,
not its *runtime behavior*. Any technique that defers the sensitive
string's construction until execution time (character codes, concatenation,
encoding, string formatting) slips past a purely textual filter. The only
robust fix is to not evaluate arbitrary user input at all.
