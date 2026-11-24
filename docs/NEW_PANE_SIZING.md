# Initial size for a new tiled pane

| Field | Value |
| --- | --- |
| Issue | [#5984: Allow setting the initial size of a new tiled pane from the CLI](https://github.com/zellij-org/zellij/issues/5984) |
| Discussion | [Implementation comment on #5984](https://github.com/zellij-org/zellij/issues/5984#issuecomment-7141017026) |
| Branch | `feat/5984-new-pane-sizing` |
| Status | Implemented |
| Reported version | Zellij 0.46.1 |

## Behavior

`zellij action new-pane` accepts a size for a tiled pane:

```sh
# New pane below the target, 30% of the target pane's rows.
zellij action new-pane --direction down --size 30%

# New pane to the right of the target, 20 columns.
zellij action new-pane --direction right --width 20

# A 10-row pane that can still grow and shrink with the layout.
zellij action new-pane --direction down --height 10 --flexible

# Force a huge request down to the largest size that still fits.
zellij action new-pane --direction down --size 1000 --force
```

| Input | Result |
| --- | --- |
| No `--size` / `--width` / `--height` | Current 50/50 split |
| `--direction down` / `up` + `--height N` | New pane gets N rows |
| `--direction down` / `up` + `--height N%` | New pane gets N% of the target's rows |
| `--direction left` / `right` + `--width N` | New pane gets N columns |
| `--direction left` / `right` + `--width N%` | New pane gets N% of the target's columns |
| `--size N` or `--size N%` | Size on the axis selected by `--direction` |
| `--floating --width` / `--height` | Unchanged: size relative to the viewport |

`--size` is the axis-neutral form and conflicts with `--width`, `--height`,
`--floating`, `--stacked`, `--in-place` and `--plugin`.

## Semantics

### Percent basis

A percent is relative to the split space: the current rows or columns of the
target pane. Example: a target pane has 40 rows. `--height 25%` gives the new
pane 10 rows and leaves 30 rows to the target.

A percent is not relative to the whole tab. Floating percentages remain
relative to the viewport.

### Constraint type

- A bare integer becomes `Constraint::Fixed(n)` and stays fixed when the
  terminal is resized.
- A percent becomes `Constraint::Percent(p)` and keeps its ratio when the
  terminal is resized.
- `--flexible` turns a cell size into an initial percent constraint. The pane
  then resizes with the layout instead of keeping its cell count.
- `--flexible` has no effect on a percent size, because percent sizes are
  already flexible.
- The target pane takes the remaining space.

### Minimum size and force

Both panes must keep at least `MIN_TERMINAL_WIDTH` / `MIN_TERMINAL_HEIGHT`
(5 cells).

Without `--force`, a request that leaves either pane below the minimum fails
through the existing `PaneNotCreatedReason::NoRoom` path. The layout is not
changed.

With `--force`, the request is clamped:

- If the new pane would be too small, it gets the minimum size.
- If the target would be too small, it keeps the minimum size and the new pane
  gets the rest.
- A pane that already has fewer than two minimum-sized halves cannot be split,
  even with `--force`.

### Rounding

For a percent size, round the new pane down. Give the leftover cell to the
target pane. A force clamp recalculates the percent from the clamped cell
count.

### Placement side

`--direction` selects the split axis. A new tiled pane is placed after the
target pane: to the right for a vertical split, below for a horizontal split.
`--direction up` and `--direction left` do not place the new pane before the
target. For placement they are equivalent to `down` and `right`: the direction
only selects the split axis. This matches the pre-existing behavior.

## Validation

Rejected with a clear CLI error:

- A size on the wrong axis: `--direction down --width`,
  `--direction right --height`.
- `--width` or `--height` with `--stacked`, `--in-place` or `--plugin`.
- A tiled size without `--direction`.
- `--flexible` or `--force` without `--size`, `--width` or `--height`.
- `--flexible` or `--force` on a floating or non-tiled pane.
- A size of `0`, a value above `100%`, or malformed text.
- `--size` together with `--width`, `--height`, `--floating`, `--stacked`,
  `--in-place` or `--plugin`.

## Server exposure

The size crosses the client-server contract in both the direct
`NewTiledPaneAction` message and `TiledPlacement`. Older peers ignore the new
optional protobuf fields and keep the 50/50 behavior.

`flexible` and `force` are creation-time controls and are not stored on the
pane. `dump-layout` shows only the resulting constraint as `size=N` or
`size="N%"`.

The plugin API does not expose these fields. Plugin-created tiled panes use
`size: None, flexible: false, force: false`.

## Implementation

| Concern | Location |
| --- | --- |
| CLI flags | `zellij-utils/src/cli.rs`, `CliAction::NewPane` |
| CLI validation and parsing | `zellij-utils/src/input/actions.rs`, `tiled_pane_size` |
| Action field | `Action::NewTiledPane` |
| Placement field | `NewPanePlacement::Tiled` |
| IPC | `client_server_contract/common_types.proto` and `ipc/protobuf_conversion.rs` |
| Routing | `zellij-server/src/route.rs`, `screen.rs` |
| Tab split paths | `zellij-server/src/tab/mod.rs` |
| Geometry | `split_with_size` in `zellij-server/src/panes/tiled_panes/tiled_pane_grid.rs` |
| Tiled pane application | `zellij-server/src/panes/tiled_panes/mod.rs` |

## Test coverage

- Clap parsing and flag conflicts in `zellij-utils/src/cli.rs`.
- CLI conversion and validation in `zellij-utils/src/input/actions.rs`.
- IPC roundtrips with non-default size, flexible and force values.
- Border-style cloning preserves the sizing fields.
- Geometry unit tests: percent, fixed, flexible, force clamp, rejection,
  minimum limits, target-relative percentages, fixed parents.
- Server placement visibility in `screen_tests.rs`.
- Tab behavior for the focused-client and explicit-target split paths.
- Fixed panes keep their cell count after a terminal resize; `--flexible`
  panes scale with the layout.
- Repeated add, remove, add sequences, mixed axes, terminal resize, percent
  and fixed panes, forced and unforced requests.
- Regression: unsized splits stay 50/50; floating sizing is unchanged.

## Limitations

- No sizing without `--direction`; the automatic layout may reapply a swap
  layout after insertion.
- No sizing for stacked, in-place or tiled plugin panes.
- No plugin API parity.
- Exact post-creation sizing still uses `zellij action resize`.
