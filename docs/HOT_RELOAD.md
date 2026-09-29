# State-preserving hot reload (design)

Status: proposal. zen-serve currently reloads the page, which restarts the
Zen program and loses its state (a counter returns to zero). This note
describes how a rebuilt `app.js` could replace the running one while the
application's state survives.

## What a Zen web program's state is

Under the JavaScript backend a program's whole memory is one heap: an
`ArrayBuffer` addressed through a `DataView`, with `Ptr` as a byte offset,
a downward-growing stack region, and pages mapped by the runtime
(`docs/JS_BACKEND.md` in the compiler tree). Host objects are small integer
handles into the runtime's handle table; event handlers are ids. Nothing a
Zen program owns lives in JavaScript objects.

That makes two tempting shortcuts unsound:

- **Keep the heap, swap the code.** Layouts are compiler output: a field
  added, reordered or retyped moves every later byte, and generic
  instantiations and allocator headers move with it. Old bytes read by new
  code are not a value of the new type.
- **Resume the old stack.** A suspended `std.js.wait` is a chain of
  generator frames whose program counters are block numbers of the old
  build. They have no meaning in the new one.

So state must cross a reload as a value, not as memory.

## Where state lives

zen-ui already separates state from presentation:

- `View` is derived: zen-ui recomputes layout and the web host redraws it.
  It never needs preserving.
- Application state is owned by actors (with the actor runtime on the JS
  backend) or, today, by the program's event loop. zen-ui's `state.zen`
  gives it a shape: a `Snapshot<T> { value, revision }` published into a
  `Latest<T>` that accepts only newer revisions.

The proposal preserves exactly the snapshots, and only for types that
declare they can be preserved.

## Proposal

### 1. Preservable state is explicit

A state type opts in by giving itself a stable name and a schema version:

```zen
Counter = { value: i64 }
Counter.impl(Preserve, {
    key = () str { "counter" }
    version = () u32 { 1 }
    save = (self: @Self, out :: Encoder) Res<(), EncodeError> { out.i64("value", self.value) }
    load = (a: Alloc, input: Decoder, from_version: u32) Res<Counter, DecodeError> {
        Ok(Counter(value: input.i64("value").value_or(0)))
    }
})
```

`save`/`load` target a self-describing encoding (the same shape `std.json`
already writes, or a compact tagged binary), keyed by field name, so adding a
field or reordering fields does not break a reload. `from_version` lets
`load` migrate deliberately; a type without `Preserve` simply restarts from
its initial value. `@meta` can later derive `save`/`load` for plain records,
but the opt-in stays explicit: preserving state silently across a schema
change is the bug this avoids.

### 2. Actors hand over their latest snapshot

A reload is a phase in the program's lifecycle, not in JavaScript:

1. zen-serve builds; on success it sends `event: hot` with the generation
   instead of `reload` (only when the build's manifest, below, says the
   program supports hot reload).
2. The page asks the running program to quiesce: the runtime queues a
   reserved handler id, so the event loop sees it through `std.js.wait`
   like any other event. Zen code never runs inside a JavaScript callback,
   and this keeps that rule.
3. The program (or zen-ui's app driver on its behalf) asks each actor that
   owns preservable state for its current `Snapshot<T>`, encodes each as
   `{key, version, revision, bytes}`, and writes the set to a host buffer
   through one `js.bind` call. Then it returns from `main`.
4. The runtime tears down the old heap and handle table, and removes the
   DOM nodes the old host created (zen-ui's web host already rebuilds the
   whole tree from `View`).
5. The page loads the new `app.js` (same `runtime.js` unless the runtime
   itself changed; if it did, fall back to a full reload). The new program
   starts normally, and before its first frame zen-ui's app driver reads the
   saved set: for each state type it constructs, a record with a matching
   `key` is passed to `load`; a missing, older-schema-without-migration or
   failing record falls back to the initial value and is reported in the
   overlay's badge ("state for counter was reset").
6. Each restored value is published as `Snapshot { value, revision }` with
   the saved revision, into a fresh `Latest` (revisions from the old
   publisher are only compared against each other, which `Latest` requires).

Handles are not preserved: a restored value that held a `js.Ref` must
re-acquire it (`load` receives no handles), which is what zen-ui's derived
`View` does anyway.

### 3. The build says whether hot reload is possible

The compiler writes `build/web/manifest.json` beside `app.js`: the runtime
hash, and for each `Preserve` type its key, version and a layout-independent
schema hash. zen-serve compares old and new manifests:

- runtime hash changed → full reload;
- a key's schema hash changed without a version bump → full reload (the
  author changed a type but not its migration story);
- otherwise → `event: hot`.

### 4. Failure is always a full reload

If quiescing does not finish within a deadline (a program stuck in a loop
that never waits), or the new program traps during restore, the client
performs `location.reload()`. Hot reload is an optimisation of the reload
that already works, never a new way to be stuck.

## What it needs

| Piece | Owner |
| --- | --- |
| `Preserve` bound, encoder/decoder | std (or zen-ui for UI state first) |
| Reserved quiesce handler id; heap/handle teardown; loading a second `app.js` into the same page | JS backend runtime |
| Snapshot collection from actors | zen-ui app driver (actors on JS are not implemented yet; until they are, the event-loop state is the only state) |
| `manifest.json` | compiler, JS layout writer |
| `event: hot`, manifest comparison | zen-serve (`Hub`, client) |

A first, useful slice needs no actors: a single-loop program like the
counter exposes one `Preserve` value, and steps 2–6 run in its event loop.
