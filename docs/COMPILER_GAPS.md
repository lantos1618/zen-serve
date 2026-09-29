# Compiler and std gaps met while building zen-serve

Compiler: `../zen-unified-integrated` at `dd65fd5c` (unified-integrated).
Each C-backend repro below is a complete program: build it with
`zen build DIR --emit-c -o a.c` and compile `a.c` with `cc`. zen-serve works
around each one; the workaround is named.

## 1. Generic actor: a bound method's `Res` result is typed `int` in C

```zen
// A generic actor calls a bound method that returns Res and matches it.
Opener = { open* = (self :: @Self) Res<(), AllocError> }
Thing = { n :: i32 = 0 }
Thing.impl(Opener, { open = (self :: @Self) Res<(), AllocError> { Ok(()) } })
Holder<W: Opener> = { inner :: W }
Holder.impl(Actor, {
    go = (self :: @Self, ctx: Context) {
        ok = self.inner.open().match({ Ok(_) => true, Err(_) => false });
        ctx.env.out.println("opened {}", ok).ignore();
    }
})
main = (env: Env) Res<i32, ActorStartError | ActorError> {
    h = env.spawn(Holder<Thing>(inner: Thing())).try();
    h.go().try();
    h.stop(); h.join();
    Ok(0)
}
```

`a.c: error: assigning to 'int' from incompatible type 'zu_tRes_…'`.
Workaround: `Watch.open` returns `bool` (src/serve/watch.zen).

## 2. Generic actor: a bound `close` dispatches to an unrelated `close`

```zen
// A generic actor's bound `close` resolves to an unrelated std `close`.
Closer = { close* = (self :: @Self) }
Thing = { n :: i32 = 0 }
Thing.impl(Closer, { close = (self :: @Self) { self.n = 1; } })
Holder<W: Closer> = { inner :: W }
Holder.impl(Actor, {
    stopped = (self :: @Self, ctx: Context) { self.inner.close(); }
})
Stream = std.net.tls
main = (env: Env) Res<i32, ActorStartError> {
    h = env.spawn(Holder<Thing>(inner: Thing())).try();
    h.stop(); h.join();
    Ok(0)
}
```

`self.inner.close()` in `stopped` is emitted as a call to
`std.net.tls.Stream.close`, a UFCS function of the same name reachable
through the import (`passing 'zu_tThing…' to parameter of incompatible type
'zu_tStream…'`). This is a wrong call selection that only the C compiler
catches. Workaround: the bound method is named `close_all`.

## 3. `i64.to_usize()` inside a generic struct's method

```zen
// i64.to_usize inside a generic struct's method.
Io = { io* = (self: @Self) i64 }
Plain = {}
Plain.impl(Io, { io = (self: @Self) i64 { 5 } })
Shard<B: Io> = {
    backend: B,
    moved = (self: @Self) usize { self.backend.io().to_usize().value_or(0) }
}
main = (env: Env) { println("{}", Shard<Plain>(backend: Plain()).moved()); }
```

`codegen cannot resolve 'to_usize'`. Workaround: a free, non-generic
`transferred(n: i64) usize` in src/serve/dev_http.zen.

## 4. Other gaps

- **"an inlining that does not terminate"** (`std/core/bool.zen:10:21`) was
  reported for a `.then` whose closure, inside a callback passed to `walk`,
  called a method that itself uses `.then`, in InotifyWatch's impl of
  `Watch`. Not reduced to a minimal program; replacing the outer
  `.then` with a two-arm `.match` avoided it (watch_inotify.zen `add`).
- **build.zen cannot choose by target OS**: `b.os.match({...})` in a `src:`
  is "unsupported build expression", so the kqueue and inotify backends
  need separate entry files and targets.
- **No include paths for `c.bind` headers in project builds**: zen-http's
  `transport.h` and std's `zen_readiness.h` are found only through
  `CFLAGS=-I…`. Every downstream project of zen-http repeats this.
- **zen-http's server cannot serve a dev server**: a fixed
  `application/octet-stream` content type, 64 KiB bodies, three statuses and
  no streaming responses. zen-serve therefore implements its own shard on
  zen-http's `TransportBackend` and worker `Engine`.
- **zen-http's build.zen lists fewer zen-crypto modules than its sources
  import** (tls13_sha384, sha2x64, tls13_cert_client and their
  dependencies); a consumer must register all of them.
- **js.bind cannot express the reload client**: no constructor calls
  (`new EventSource`) and handlers deliver only an id, not the event's
  `data`; see README, "The one hand-written script".
- **zen-web: `zen build TARGET --target js-browser` is refused for project
  targets** ("unknown argument"); the project route is `web: true`.
- **Library ergonomics**: `value_or` exists only on `Res<T>` (not
  `Res<T, E>`); `.map_err` rejected a named function argument; there is no
  `return`, so early exits are loops with `h.break(value)` returning
  `Res<T>`; a `str` nested in an actor message struct is refused while a direct
  `str` parameter is copied (so the Builder sends diagnostics as a bare
  `str`).
