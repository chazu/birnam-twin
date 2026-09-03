# birnam-twin

An idiomatic Birnam interface to the
[Twin](https://github.com/cosmos72/twin) text-mode window system. This library
depends on [`birnam-alien-twin`](https://github.com/chazu/birnam-alien-twin),
which owns the C ABI bridge and vendored `libtw`; this repository adds managed
objects and Birnam-shaped failure/lifetime behavior without duplicating that
native wrapper.

## Use

Pin this repository as a Birnam Git dependency and fetch it explicitly:

```toml
[dependencies]
birnamTwin = { git = "https://github.com/chazu/birnam-twin.git", rev = "<full-commit-id>" }
```

```sh
birnam fetch
birnam check
birnam build
```

The module is named `#birnamTwin` and requires `#twin`, so Birnam recursively
fetches and builds the pinned `birnam-alien-twin` dependency and propagates its
wrapped `libtwin.dylib` artifact.

## Managed API

Open a connection and window with scoped cleanup:

```birnam
let result := [Twin withConnectionTo: ":0" do: [:connection |
  [connection withWindow: "Workspace" width: 80 height: 24 do: [:window |
    [window write: "hello" atX: 2 y: 1];
    [window sync];
    [connection waitForEvent]]]];
```

`Twin connect` and `Twin connect:` answer a `TwinConnection` or a Birnam
`Error`, never a nullable raw handle. Connections and windows make repeated
`close` harmless, guard operations after close, and offer scoped
`withConnection...` / `withWindow...` helpers built on Birnam's `ensure:`.

Window operations are chainable and answer the window on success or an Error:

- `write:` and `write:atX:y:`
- `title:`
- `cursorX:y:`
- `resizeTo:by:`
- `sync`
- `close`

Connections expose display/server metadata, `flush`, `sync`, `nextEvent`, and
`waitForEvent`. An event is copied into a `TwinEvent` value and its native
allocation is freed before it reaches application code. `kind` classifies the
event as `#key`, `#mouse`, `#change`, `#gadget`, `#menuRow`, `#control`,
`#display`, or `#unknown`; the raw numeric fields remain available when needed.

## Development

```sh
birnam fetch
birnam check
birnam build
birnam test
birnam run
```

Pass a live Twin display to create, write, synchronize, sample one queued event
without blocking, and clean up a scoped window:

```sh
birnam run -- :0
```

The current slice deliberately stops at the primitives already provided by
`birnam-alien-twin`: connection metadata, basic windows, UTF-8 output, and
copied events. An IDE can establish its event loop and ownership model here.
Menus, gadgets, selections, richer drawing/cell operations, and listener
integration should be added to the C bridge only when an IDE slice demands
them, then surfaced here as managed Birnam objects.
