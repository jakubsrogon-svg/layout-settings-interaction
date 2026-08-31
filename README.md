# Section Layout Control — interaction prototype

Open `layout-settings-prototype.html` directly in a browser (no build step,
no server — it pulls React, ReactDOM, Babel Standalone and Tailwind from
CDN and transforms the JSX in-browser). All state is in-memory only.

## What changed vs. the current production design

The preset row (`Default` / `Compact` / `Custom`) no longer *is* the state —
it's a derived readout of it. `activePreset` is computed every render by
comparing the live `values` map against `PRESETS.default` and
`PRESETS.compact`; if neither matches, it's `'custom'`. Nothing sets
`activePreset` directly, so it can't drift from what the section list and
preview actually show. Section controls (accordions, option pickers,
per-section reset) are always live and never gated behind a preset click.

## Invariants that were awkward to satisfy

1. **Custom's disabled state vs. "the highlighted chip is always
   `activePreset`" (rules 2 and the "Custom disabled while
   `lastCustomValues` is null" rule).** The brief only calls for snapshotting
   into `lastCustomValues` when *switching away from* a custom
   configuration via a preset click. But a user can also arrive at Custom
   directly — just by editing one section value while sitting on Default or
   Compact — without ever clicking a preset chip. At that instant
   `activePreset` becomes `'custom'` (so the Custom chip must show
   checked/highlighted per rule 2), but `lastCustomValues` would still be
   `null` under a literal reading (so Custom would also have to render
   `aria-disabled`). A chip that is simultaneously checked and disabled
   isn't expressible in the radiogroup pattern, and it's a confusing state
   regardless.

   Resolution taken here: `lastCustomValues` is kept continuously in sync
   with `values` any time `computeActivePreset(values) === 'custom'` —
   on a direct section edit, a per-section reset, or a preset switch away
   from Custom — rather than only at the moment of switching presets. This
   satisfies the letter of "snapshot before switching away" (it's already
   current) while removing the contradiction. It's a real judgment call,
   not just an implementation detail — flagging it because it's the one
   place the spec's state machine underspecifies a transition.

2. **"Custom is disabled until a custom configuration exists" combined with
   arrow-key navigation.** ARIA radiogroup arrow-key traversal conventionally
   moves focus *and* selects. Custom needs to stay in the tab/arrow order
   (so it's discoverable and so focus doesn't jump around once it becomes
   enabled) but must not be selectable while disabled. Solved by keeping the
   disabled chip focusable but guarding the click/select handler — arrow
   keys move focus onto it, but activating it is a no-op until
   `lastCustomValues` exists. This is a minor deviation from the "disabled
   controls are usually unfocusable" convention, done deliberately for
   discoverability; worth a product call if it feels wrong in practice.

Everything else in the invariant list (preset clicks never touching
`expanded`; per-section reset touching exactly one key; one-click recovery
back into Custom) fell out directly from the state model without needing a
judgment call.

## Note on the open question (compact vs. density)

Built exactly as specified: `compact` is the structural preset (Split
layouts + the denser `List` skills variant), not a density/spacing axis.
No decision about splitting density into a separate control was made here.
