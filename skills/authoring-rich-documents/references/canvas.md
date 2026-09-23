# Authoring a `<Canvas>`

Read this before authoring ANY `<Canvas>` block. Also read `references/wireframe.md` — every artboard body is a wireframe fragment and every rule there applies to it unchanged.

## When a board beats one inline frame

A `<Canvas>` is a pan/zoom board of **artboards**, one per user-visible state. It is the right block when the change is only reviewable as a set:

- A screen that gains states the reader must see together — default, empty, error, loading.
- A short sequence the user moves through — form, confirm dialog, result.
- One screen at more than one surface, where the responsive behaviour is the change.

It is the wrong block for a single state. A board of one artboard is a `<Wireframe>` with extra chrome and less context, because it leaves the prose column. **Default to `<Wireframe>`; reach for `<Canvas>` at two or more states.** Above about six artboards a board stops being scannable — split it by area or by journey, or ask whether every state is really load-bearing.

## The authored shape

```mdx
<Canvas id="settings-states" caption="Settings redesign — user-visible states" height={560}
  artboards={[
    { id: "default", title: "Default", surface: "desktop", html: `…wireframe fragment…` },
    { id: "invalid", title: "Validation error", surface: "desktop", html: `…` },
    { id: "confirm", title: "Discard changes", surface: "popover", html: `…` },
  ]}
  annotations={[
    { target: "default", placement: "right", text: "Autosave replaces the Save button" },
  ]}
  connectors={[
    { from: "default", to: "invalid", label: "invalid input" },
  ]}
/>
```

- Each artboard needs `id` (unique on the board), `title` (what the reader sees above the frame — name the STATE, not the screen), `surface`, and `html`.
- `height` is the board's viewport in the document, not the size of its content; the board pans and zooms inside it. The default suits three or four frames.

## Layout

Artboards flow into two lanes automatically, in authored order:

- **Broad lane (top):** `browser` and `desktop` frames.
- **Compact lane (below):** `mobile`, `popover` and `panel` frames.

That is deliberately simple, and it is enough for almost every board. Author the artboards in the order a reader should meet them and let the lanes do the work.

Override with `x` / `y` on an artboard only when the arrangement itself carries meaning — a branch where two outcomes should sit one above the other, or a pair meant to be read as a column. An override opts that artboard out of the flow entirely, so overriding one frame of a lane and not the rest usually produces an overlap. Override all of them or none.

## Annotations

`{ target, placement, text }` — a note pinned beside the frame it describes, with a leader line to it.

- `target` must be an artboard `id` on this board. One that isn't renders as a visible warning strip above the board rather than silently disappearing — but it is still an authoring error to fix.
- `placement`: `right` (default) | `left` | `top` | `bottom`.
- One sentence. An annotation says why the frame looks like this, or what the reader would otherwise miss — not a caption restating what is plainly visible.
- Two or three per board, not one per frame. If every frame needs a note, the notes belong in the prose around the board.

## Connectors

`{ from, to, label }` — an arrow between two artboards.

Use them **only for genuinely sequential states**: the user does something on `from` and arrives at `to`. Label the arrow with the transition (`"invalid input"`, `"confirms"`), not with a description of the destination.

Do not use connectors to express "these two are related", to draw a legend, or to link every frame to every other. A board with more arrows than frames has stopped communicating a sequence. Comparative states — default vs empty vs error — need no connectors at all; the lane order already reads left to right.

A connector naming an artboard that isn't on the board renders as a warning strip, same as a stray annotation.

## Quality bar

- **One artboard per state, and every state earns its place.** If two frames differ only in a value nobody will look at, they are one frame.
- **Title the state, not the screen.** `Validation error` and `Empty — no saved views` tell the reader what they are looking at; `Settings` three times does not.
- **Keep surfaces consistent across comparable states.** Same screen, different states → same `surface`. A different surface is a claim that the surface changed.
- **The board is not the document.** It shows the states; the prose around it says what changes and why. Do not push explanation into annotations because there is space on the board.
- **Never leave an artboard body empty.** An empty or malformed fragment renders as a visible placeholder frame, which is a bug report, not a design.

## Before you move on

- Two or more artboards, each a distinct user-visible state with a state-naming title.
- Every artboard body obeys `references/wireframe.md` — no widths, no positions, no hard-coded colour, real product copy.
- Annotation targets and connector endpoints all name real artboard ids on this board.
- Connectors join sequential states only; comparative states have none.
- The board renders with no warning strip above it.
