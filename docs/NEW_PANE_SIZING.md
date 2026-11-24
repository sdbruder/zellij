# Initial size for a new tiled pane

| Field | Value |
| --- | --- |
| Issue | [#5984: Allow setting the initial size of a new tiled pane from the CLI](https://github.com/zellij-org/zellij/issues/5984) |
| Branch | `feat/5984-new-pane-sizing` |
| Status | Plan. No implementation yet. |
| Reported version | Zellij 0.46.1 |

## Intent

Let `zellij action new-pane` set the initial size of a new tiled pane. The user
can give a size in cells or as a percent of the space that is split. The
existing behavior stays the same when no size is given.

## Problem

`new-pane` accepts `--width` and `--height` only together with `--floating`
(`requires("floating")` in `zellij-utils/src/cli.rs`). A tiled split always
halves the target pane: `split()` returns two equal percent dimensions
(`zellij-server/src/panes/tiled_panes/tiled_pane_grid.rs`).

A script that wants a small output pane must call `zellij action resize` many
times. The final size is not predictable, because each resize moves one
boundary by an increment.

## Goals

- Accept `--width` and `--height` for tiled panes.
- Accept the existing size syntax: a bare integer is a number of cells, an
  integer with `%` is a percent. Examples: `30`, `30%`.
- Resolve a percent against the space that is split, not the whole terminal.
- Keep the default split (no size given) at the current 50/50 behavior.
- Keep the floating behavior and syntax unchanged.

## Non-goals

- A new action or a new command. This change extends `new-pane`.
- Exact sizing after creation. `zellij action resize` keeps its increment
  semantics.
- Stacked panes, in-place panes, and tiled plugin panes. A tiled plugin pane
  rejects `--direction`, so it cannot select a split axis.
- A `--size` flag that combines width and height.

## Proposed command line

```sh
# New pane below the target, 30% of the target pane's rows.
zellij action new-pane --direction down --height 30%

# New pane to the right of the target, 20 columns.
zellij action new-pane --direction right --width 20
```

| Input | Result |
| --- | --- |
| No `--width` / `--height` | Current 50/50 split |
| `--direction down` / `up` + `--height N` | New pane gets N rows |
| `--direction down` / `up` + `--height N%` | New pane gets N% of the target's rows |
| `--direction left` / `right` + `--width N` | New pane gets N columns |
| `--direction left` / `right` + `--width N%` | New pane gets N% of the target's columns |
| `--floating --width` / `--height` | Unchanged: size relative to the viewport |

### Validation

Reject these combinations with a clear CLI error:

- A size on the wrong axis: `--direction down --width`, `--direction right --height`.
- `--width` or `--height` with `--stacked` or `--in-place`.
- `--height` or `--width` without `--direction` in the first version. See
  "Open questions".
- A size of `0`, a negative value, a value above `100%`, or malformed text.

Keep the current floating validation: floating sizes reject `0`, and accept
`x`/`y` offsets as `0`.

## Semantics

### Percent basis

A percent is relative to the split space: the current rows or columns of the
target pane. Example: a target pane has 40 rows. `--height 25%` gives the new
pane 10 rows and leaves 30 rows to the target.

A percent is not relative to the whole tab. This differs from the floating
`--height 25%`, which uses the viewport. State this difference in the help text.

### Constraint type

Store the requested size as a layout constraint:

- A bare integer becomes `Constraint::Fixed(n)`.
- A percent becomes `Constraint::Percent(p)`.

This matches `SplitSize` in `zellij-utils/src/input/layout.rs` and `Dimension`
in `zellij-utils/src/pane_size.rs`. As a result:

- A percent size keeps its ratio when the terminal is resized.
- A fixed size keeps its cell count when the terminal is resized.
- The target pane takes the remaining space.

If the intent is "initial size only", and later resizes must divide all panes
equally, the constraint must be flexible instead. This is an open question.

### Minimum size

Both panes must keep at least `MIN_TERMINAL_WIDTH` / `MIN_TERMINAL_HEIGHT`
(5 cells, `zellij-server/src/tab/mod.rs`).

If the request leaves less than the minimum, reject the new pane. Use the
existing `PaneNotCreatedReason::NoRoom` path. Do not clamp the size, because
the use case is a predictable size. A silent clamp hides the failure.

### Rounding

For a percent size, round the new pane down. Give the leftover cell to the
target pane. This keeps the new pane at or below the requested size.

### Placement side

The current implementation uses `--direction` only to select the split axis.
A new tiled pane is placed after the target pane: to the right for a vertical
split, below for a horizontal split. `--direction up` and
`--direction left` do not place the new pane before the target. This change
does not alter that behavior.

## Implementation sketch

1. **CLI** (`zellij-utils/src/cli.rs`, `CliAction::NewPane`): remove
   `requires("floating")` from `width` and `height`. Keep all floating rules.
   Add the axis and mode checks in the `CliAction` to `Action` conversion.

2. **Action** (`zellij-utils/src/input/actions.rs`): parse the size with
   `PercentOrFixed::from_str`. Add `size: Option<SplitSize>` (or an equivalent
   axis-neutral type) to `Action::NewTiledPane` and `NewPanePlacement::Tiled`.
   `NewPanePlacement` derives `Serialize`/`Deserialize`; give the new field
   `#[serde(default)]` so existing messages still decode.

3. **Routing** (`zellij-server/src/route.rs`): pass the size for both the
   non-blocking path (`Action::NewTiledPane`) and the blocking path
   (`Action::NewBlockingPane` with `NewPanePlacement::Tiled`).

4. **Tab** (`zellij-server/src/tab/mod.rs`, `Tab::new_pane`): pass the size to
   the split methods. All entry paths already meet in
   `TiledPanes::split_pane_id_horizontally` / `split_pane_id_vertically`:
   focused client, linked client, and explicit `pane_id` target. Put the
   sizing logic there, not in the `Tab` wrappers.

5. **Geometry** (`zellij-server/src/panes/tiled_panes/tiled_pane_grid.rs`): add
   a new function next to `split()`:

   ```rust
   pub fn split_with_size(
       direction: SplitDirection,
       rect: &PaneGeom,
       size: SplitSize,
   ) -> Option<(PaneGeom, PaneGeom)>
   ```

   It returns the target rect and the new rect. For a percent request on a
   percent parent (`parent_pct = p`, `request = q`), set the new pane to
   `p * q / 100` and the target pane to `p - p * q / 100`. For a fixed request,
   use `Dimension::fixed(n)` and reduce the target by the same number of cells.
   Keep `split()` unchanged for every other caller.

6. **Help text**: state the percent basis for tiled panes and for floating
   panes. `new-pane --help` is generated from the clap attributes.

## Tests

Unit tests:

- `tiled_pane_grid`: percent split, fixed split, both split directions,
  rounding, target size below the minimum, request larger than the space.
- `tab`: `NewPanePlacement::Tiled` with `size: Some(SplitSize::Percent(30))`
  produces the expected `PaneGeom` for the new pane and the target. Cover the focused
  client path, the no-focus path, and the explicit pane-id path.
- `screen`: no room reports `PaneNotCreatedReason::NoRoom`.
- CLI parsing: tiled sizes parse; floating sizes parse as before; wrong axis,
  stacked, in-place, `0`, `101%`, and malformed sizes fail.
- Regression: `new-pane --direction down` without a size stays 50/50; floating
  `--height 30%` keeps its current geometry.

Manual / integration check:

```sh
zellij action new-pane --direction down --height 30% -- sleep 60
zellij action dump-layout   # check rows of the new pane
```

Resize the terminal afterwards and confirm the expected behavior for a
percent size and for a fixed size.

## Open questions

1. **No direction.** Should `--height 30%` work without `--direction` by
   letting the auto layout pick the axis first, then applying the size? First
   version: reject it.
2. **Both axes.** `--width` and `--height` together cannot both apply to one
   split. Reject the off-axis flag, or reject the pair.
3. **Percent basis vs layouts.** Layout `SplitSize::Percent` is resolved
   against the total dimension that is split (`split_space` in
   `zellij-utils/src/input/layout.rs`). This proposal uses the target pane's
   current size. Confirm that the two definitions agree for nested splits.
4. **Percent or fixed persistence.** Should a fixed size stay fixed after a
   terminal resize, or become flexible after the first resize? The constraint
   model gives "stay fixed" for free; "initial only" needs extra work.
5. **Reject or clamp.** This plan rejects a size that leaves a pane below the
   minimum. A clamp is friendlier for scripts but hides the miss.
6. **Plugin API parity.** Tiled plugin panes reject `--direction` today. Do not
   extend sizing to plugins in this change.

## References

| Concern | Location |
| --- | --- |
| CLI flags | `zellij-utils/src/cli.rs`, `CliAction::NewPane` |
| CLI to action conversion | `zellij-utils/src/input/actions.rs`, `CliAction::NewPane` arm |
| Placement type | `zellij-utils/src/data.rs`, `NewPanePlacement` |
| Routing | `zellij-server/src/route.rs`, `Action::NewTiledPane` and `Action::NewBlockingPane` |
| Dispatch | `zellij-server/src/tab/mod.rs`, `Tab::new_pane` |
| Tiled splits | `zellij-server/src/panes/tiled_panes/mod.rs`, `split_pane_id_horizontally` / `split_pane_id_vertically` |
| Geometry split | `zellij-server/src/panes/tiled_panes/tiled_pane_grid.rs`, `split()` |
| Layout sizes | `zellij-utils/src/input/layout.rs`, `SplitSize` and `split_space()` |
| Dimensions | `zellij-utils/src/pane_size.rs`, `Dimension` |
| Minimum sizes | `zellij-server/src/tab/mod.rs`, `MIN_TERMINAL_WIDTH` / `MIN_TERMINAL_HEIGHT` |
