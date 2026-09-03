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

Window operations are chainable and answer the window on success or an Error.
The managed window remembers its last known extent and current colors, so a
renderer can clear and repaint without carrying parallel native state:

- `write:` and `write:atX:y:`
- `renderLines:` (clear, repaint rows, synchronize once)
- `fillCodepoint:fromX:y:width:height:foreground:background:` and `clear`
- `foreground:background:` with validated `TwinColor` values
- `title:`
- `cursorX:y:`
- `moveToX:y:`, `resizeTo:by:`, and `scrollByX:y:`
- `show`, `hide`, `focus`, `raise`, and `lower`
- `sync`
- `close`

For windows whose policy differs from the ergonomic default, use
`newWindow:width:height:cursor:attributes:flags:` or its scoped `withWindow...`
counterpart. `Twin` names the supported cursor, attribute, and flag constants;
attribute bits are disjoint and can be combined with `+`.

Connections expose display/server metadata, `flush`, `sync`, `nextEvent`, and
`waitForEvent`. An event is copied into a `TwinEvent` value and its native
allocation is freed before it reaches application code. `kind` classifies the
event as `#key`, `#mouse`, `#change`, `#gadget`, `#menuRow`, `#control`,
selection/control variants, `#display`, or `#unknown`; the raw numeric fields
remain available when needed. Predicates cover key, mouse, resize, expose,
standard close requests, and Shift/Control/Alt modifiers. `observeEvent:` keeps
a managed window's cached extent current after resize events.

`[connection eventPumpDo: handler]` answers a `TwinEventPump`. Its `poll`,
`wait`, `drainAtMost:`, and `drain` operations provide one nonblocking step, one
blocking step, or a bounded drain of queued events (`drain` caps each batch at
256). The pump deliberately does
not own the connection and does not install an endless application loop; an IDE
can multiplex Twin's `fileDescriptor` with evaluator, LSP, subprocess, timer,
and LLM inputs at the application layer.

## Examples

Each example is a small standalone Birnam project with a local path dependency
on this checkout:

```sh
# A scoped hello window; press any key to exit.
birnam run examples/hello -- :0

# Colors, rectangular cell fill, positioned text.
birnam run examples/colors -- :0

# Blocking event pump with key, resize, and close handling.
birnam run examples/events -- :0
```

Omit the display argument to use `:0`. The event example exits on `q` or the
window close gadget and prints received event kinds/codes in the terminal.

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

This is the complete custom-drawn application foundation: resource ownership,
configurable windows, deterministic cell repainting, RGB color, geometry and
stacking, event classification, and composable event consumption. Twin menus,
gadgets, inter-client messaging, and selection ownership remain intentionally
outside this layer until an application chooses to use those server-side
facilities; they are not required for a text editor that draws its own chrome.
