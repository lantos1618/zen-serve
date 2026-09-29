# zen-serve

A development server for Zen web apps, written in Zen. It builds a project
target with the Zen JavaScript backend, serves the output, watches the
sources, rebuilds on change and live-reloads every open browser tab. A
failed build shows the compiler's diagnostics as an overlay in the page; the
next good build clears it.

```text
zen-serve <project-dir> <target> [--port N] [--host ADDR] [--tls --cert PEM --key PEM]
          [--build-cmd CMD] [--root DIR]... [--watch PATH]... [--debounce MS]
```

| Option | Default | Meaning |
| --- | --- | --- |
| `--port` | 8000 | Port to listen on (IPv4) |
| `--host` | 127.0.0.1 | Address to bind |
| `--tls` | off | HTTPS through zen-http's native TLS 1.3 (no OpenSSL) |
| `--cert`, `--key` | build/cert.pem, build/key.pem | PEM chain (leaf first) and named-curve P-256 key |
| `--build-cmd` | `"${ZEN:-zen}" build TARGET` | Run by `/bin/sh -c` in the project directory |
| `--root` | build/web, then web | Served directories, searched in order (repeatable) |
| `--watch` | src, build.zen | Files and directories watched recursively (repeatable) |
| `--debounce` | 100 | Milliseconds of quiet before a rebuild |

The default command builds the target as its `build.zen` declares it. A web
target writes the browser layout with `web: true` (or on the command line,
`zen build ROOT --backend js --target js-browser -o DIR`):
`build/web/{index.html, runtime.js, app.js}`.

## Try it

The example is zen-ui's counter (`examples/counter`, built against the
sibling `../zen-ui` checkout by the JavaScript backend in `../zen-web`):

```sh
ZEN=$PWD/../zen-web/zen ZEN_STD=$PWD/../zen-web/src \
    build/zen-serve examples/counter counter --port 8123
# open http://127.0.0.1:8123/, then edit examples/counter/src/main.zen
```

Screenshots of that session are in `docs/screenshots/`: the page before an
edit, after the edit (reloaded without touching the browser), with a
deliberate syntax error (overlay), and after the fix (two tabs recovered).

## Build and test

The C compile needs the include directories of two hand-written headers
that zen-http and std still use (`transport.h`, `zen_readiness.h`):

```sh
export ZEN_STD=$PWD/../zen-unified-integrated/src
export CFLAGS="-I$PWD/../zen-http-integrated/src -I$ZEN_STD/std/net"
../zen-unified-integrated/zen build zen-serve        # macOS/BSD (kqueue)
../zen-unified-integrated/zen build zen-serve-linux  # Linux (inotify)
../zen-unified-integrated/zen test unit
ZEN=$PWD/../zen-unified-integrated/zen ../zen-unified-integrated/zen test live
```

`build.zen` cannot yet select a module by target OS, so each watch backend
has its own entry (`src/main.zen`, `src/main_linux.zen`). Dependencies are
sibling checkouts: `../zen-http-integrated` (transports, socket setup and
the worker engine) and `../zen-crypto-integrated` (native TLS).

- `tests/unit.zen`: request-target decoding and traversal refusal, path
  containment, content types, SSE framing (round-tripped through
  `std.net.sse`), the event hub's replay, HTML injection, debounce timing
  and build coalescing.
- `tests/live.zen`: the real actors on a port against a fixture project
  built by `$ZEN`: content types and headers, `..`/encoded/absolute targets
  and a symlink escaping the root (403) versus one inside it (200), two
  concurrent event streams, one build for a burst of five edits, a syntax
  error reaching both streams as `build-error` (and a tab opened while
  broken), and recovery reloading all three.

## Architecture

```text
                 kqueue / inotify fd
                        │ readable
                        ▼
  ┌──────────────── Watcher actor ────────────────┐
  │ poll(fd, debounce deadline) → drain → re-arm  │
  │ Debounce: one request per settled burst        │
  └───────────────┬───────────────────────────────┘
                  │ changed(n)
                  ▼
  ┌──────────────── Builder actor ────────────────┐
  │ BuildQueue: single flight; N changes during a │
  │ build → exactly one more build                 │
  │ /bin/sh -c "$build_cmd" in the project        │
  └───────────────┬───────────────────────────────┘
                  │ building / built / failed(diagnostics)
                  │ + one byte on the wake pipe
                  ▼
  ┌──────────────── Server actor ─────────────────┐
  │ zen-http worker Engine (readiness loop)       │
  │  └─ DevShard: HTTP/1.1 over zen-http          │
  │     TransportBackend (plain TCP or native TLS)│
  │     • static files from the roots             │
  │     • /__zen/client.js, injected into HTML    │
  │     • /__zen/events: SSE subscribers          │
  │     • Hub: generation + last failure, replayed│
  │       to new subscribers                      │
  └───────────────┬───────────────────────────────┘
                  │ event: hello / building / reload / build-error
                  ▼
         browser tabs (EventSource → reload or overlay)
```

Each actor has its own thread. Nothing mutable is shared: the Watcher owns
the notification descriptor, the Builder owns the build process, and the
Server owns the listener, every connection and every subscriber stream. The
Server serves in bounded engine passes and yields to its mailbox between
them; the Builder rings a wake pipe after each message so a Server blocked
in its readiness wait returns at once.

### Static files

`GET` and `HEAD` only. A target must be origin-form; it is split into
segments, percent-decoded per segment, and refused (403) if any segment is
`..` or `.`, empty (`//x`), or decodes to `/`, `\` or NUL. The joined path is
then canonicalised with `realpath(3)` and must lie inside the canonical root,
so a symlink pointing out of the root is refused too. A directory serves its
`index.html`. Every response carries `Cache-Control: no-store` and the
cross-origin isolation headers (COOP/COEP) that a Zen program needing
`SharedArrayBuffer` requires. `.html` responses get one
`<script src="/__zen/client.js">` inserted before `</head>`.

### Live-reload protocol

`GET /__zen/events` answers `text/event-stream` with `retry: 1000` and:

| Event | Data | Client action |
| --- | --- | --- |
| `hello` | build generation | reload if it differs from the generation first seen (a build finished while disconnected) |
| `building` | empty | small "rebuilding…" badge |
| `reload` | new generation | `location.reload()` |
| `build-error` | diagnostics, one `data:` line per line | full-page overlay |

A client that connects while the latest build is broken receives
`build-error` right after `hello`. A `: ping` comment every 15 seconds lets
the server notice closed tabs; a subscriber more than 4 MiB behind is
dropped.

### The one hand-written script

`/__zen/client.js` (in `src/serve/client.zen`) is the only JavaScript in
zen-serve. It needs `new EventSource(...)` and the event's `data`; today's
`js.bind` has no constructor calls and its handlers deliver only an id, so
the client cannot yet be a Zen program built by the JavaScript backend.

## State-preserving hot reload

A design for swapping `app.js` without losing application state is in
[docs/HOT_RELOAD.md](docs/HOT_RELOAD.md). It is not implemented.

Compiler and library gaps met on the way, with minimal repros, are in
[docs/COMPILER_GAPS.md](docs/COMPILER_GAPS.md).

## Limits

IPv4 only; 64 simultaneous connections; request heads up to 16 KiB and no
request bodies; whole files are read per request (fine for dev bundles, not
for large media); no range requests or directory listings. The kqueue
backend holds one descriptor per watched file (at most 4096).
