# StackTrac

StackTrac is a lightweight CLI SRE diagnostic tool that accepts a trace via `--file`, `--string`, `--url` or stdin, extracts the most likely offending stack frame, and returns the probable file and line number plus a search link (default: StackOverflow). The CLI also returns the exception message when available.

Add CI / npm / license badges to `README.md` when available.

- Node \>= 18 (the CLI falls back to `node-fetch` if built-in `fetch` is unavailable).

Global install:
    npm install -g stacktrac

Local/dev install:
    npm install --save-dev stacktrac

From a file:
    stacktrac --file `path/to/trace.txt`

From an inline string:
    stacktrac --string "Exception: Something went wrong: null pointer\n\tat com.example.Foo(Foo.java:12)"

From a URL:
    stacktrac --url "https://example.com/log.txt"

From stdin:
    cat trace.txt | stacktrac

Machine-readable output:
    stacktrac --file `path/to/trace.txt` --json

- `-f, --file <path>` — read trace from `path`  
- `-s, --string <text>` — pass trace as a string  
- `-u, --url <url>` — fetch trace from a URL  
- `-j, --json` — output JSON  
If no input option is given the CLI reads from stdin when piped.


Human readable:
    Exception: java.lang.NullPointerException: Attempt to read property 'id' of null
    Exception message: Attempt to read property 'id' of null
    Probable frame: at com.example.service.UserService.getUser(UserService.java:87)
    File: UserService.java:87
    Search: https://stackoverflow.com/search?q=java.lang.NullPointerException+UserService.getUser

JSON (`--json`):
    {
      "found": true,
      "exception": "java.lang.NullPointerException: Attempt to read property 'id' of null",
      "exceptionMessage": "Attempt to read property 'id' of null",
      "frameString": "at com.example.service.UserService.getUser(UserService.java:87)",
      "file": "UserService.java",
      "line": 87,
      "function": "com.example.service.UserService.getUser",
      "link": "https://stackoverflow.com/search?q=java.lang.NullPointerException+UserService.getUser"
    }

Notes:
- `exception` contains the exception header and message when present. `exceptionMessage` is provided for easy access to the message text.
- If the parser cannot extract a message, `exceptionMessage` may be omitted or empty.

- Accepts raw logs and traces and extracts trace blocks (Java, Node, Python heuristics).  
- Chooses the most likely user frame by ignoring common framework/internal packages.  
- Returns `exception` (header plus message when available), `exceptionMessage` (message only), `file`, `line`, `frameString`, `function` and a `link` to search results.  
Extend `lib/parser.js` to improve message extraction or add languages.

Optional environment variables:
- `STACKTRAC_IGNORE_PACKAGES` — comma-separated package prefixes to ignore.  
- `STACKTRAC_SO_BASE` — base search URL (defaults to StackOverflow search).  
- `STACKTRAC_JSON` — force JSON output when set to `1`.

Main files:
- `package.json`  
- `bin/stacktrac.js`  
- `lib/parser.js`

Local testing and linking:
    git clone <repo>
    cd <repo>
    npm install
    npm link

Run tests:
    npm test

- Fork, add tests, follow existing style and open a PR.  
- Use the issue tracker for bug reports and feature requests.

Distributed under the MIT License. See `LICENSE` for details.
## written by kamal.hakim@aexp.com Wed Nov  5 09:38:17 MST 2025

