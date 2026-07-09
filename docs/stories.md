# Map stories — guided tours as markup

A story turns a map into a narrated sequence: fly here, reveal this, highlight that — authored declaratively, played back with a scrubber, and testable like everything else. Runnable example: [`examples/story.html`](../examples/story.html).

## The one rule

**The map describes a scene; the story describes intent over time about that scene.** `<deck-story>` is a *sibling* of your layers, never a container — steps reference elements by id, and a step that contains an element is a validation **error**. Delete the story and the map is byte-for-byte the same map.

```html
<deck-map center="[-122.35, 37.85]" zoom="10" basemap="maplibre">
  <deck-layer id="regions" .../>                       <!-- a complete, ordinary map -->
  <deck-overlay id="chapter" visible="false">…</deck-overlay>

  <deck-story id="tour" interrupt="pause">
    <deck-step duration="3s" action="fly-to" center="[-122.44, 37.78]" zoom="12" curve></deck-step>
    <deck-step duration="4s" action="show-overlay" target="chapter" parallel delay="1500ms"></deck-step>
    <deck-step duration="2s" action="hide-overlay" target="chapter" delay="1s"></deck-step>
    <deck-step duration="2s" action="toggle-layer" layer="bikes" visible="true"></deck-step>
  </deck-story>

  <deck-widget type="player" story="tour" position="bottom-left"></deck-widget>
</deck-map>
```

## Steps

A step is `action="…"` plus payload attributes — the **same actions, same kebab-case payload rule** as `<deck-behavior>` and `data-emit` buttons. Anything an action can do, a step can do, including your own `DeckMap.registerAction` actions.

Timing attributes (everything else rides into the payload):

| Attribute | Meaning |
|---|---|
| `duration` | How long the step's animation runs (`"3s"`, `"800ms"`). Also passed to the action — so a `fly-to` step's camera ease matches its slot. `0`/omitted = instantaneous. |
| `delay` | Stagger before the step starts. |
| `parallel` | Start with the *previous step* instead of after everything so far (concurrent camera + reveal is the classic use). |

Sequential steps wait for **everything** before them to finish — a short `parallel` step never drags the next step into the middle of a longer one.

**Write declarative payloads.** Seeking replays steps, so a step should *set* state, not flip it: `action="toggle-layer" layer="bikes" visible="true"` (sets), not a bare toggle. Custom actions replay on seek but can't be rewound — the library warns once.

## Playback control

Playback is three actions — `story-play`, `story-pause`, `story-seek {story, t}` — so every dispatch surface works:

- **Attributes:** `autoplay` (starts after `deck-map-ready`), `loop`.
- **The built-in player:** `<deck-widget type="player" story="tour">` — play/pause, seek bar, elapsed/total. It's dumb chrome: it emits the three actions and re-renders off the story's `deck-story-tick` events, so replacing it is just your own `data-emit` buttons: `<button data-emit="story-play" data-story="tour">▶</button>`.
- **Behaviors:** `<deck-behavior on="click" layer="regions" action="story-play" story="tour">` — click to start a tour.
- **Script:** `storyEl.play() / pause() / seek(ms)`, plus `state`, `currentTime`, `duration` properties and the `deck-story-tick` event.

**One active story per map** — playing story B pauses story A. **`interrupt="pause"`** (the default) pauses playback when the user grabs the map (pointer-down or wheel); `interrupt="ignore"` opts out.

## Seeking (the scrub model)

`seek(T)` restores the story's captured initial state, then applies the *final* state of every step starting **before** T, instantly. `seek(0)` therefore means "pristine start"; seeking to the very end applies everything. Scrubbing is step-granular — a seek into the middle of a 3s camera flight lands at the flight's end state, not 40% along it.

Initial-state capture happens on first play, from the attributes the built-in-action steps will touch (layer visibility, overlay visibility, highlights, filter ranges) plus the camera. That's why declarative payloads matter: the library can know what to snapshot by reading them.

## Testing stories

Deterministic, headless, no timers to race ([testing guide](testing.md)):

```js
const h = await mountForTest(myPageHtml);
const story = h.story("tour");
await story.advance(2100);                          // manual clock — exact, rAF never runs
expect(overlay.getAttribute("visible")).toBe("true");
await story.seek(0);
expect(overlay.getAttribute("visible")).toBe("false");
```

`h.story(id).advance(ms)` drives the clock manually; `play/pause/seek/state/currentTime/duration` mirror the element API.

## Effect verbs

Three intent-level verbs, usable as step shorthands (`<deck-step fade layer="regions" duration="1s">`) or as plain actions from behaviors and buttons (`data-emit="pulse" data-layer="quakes"`):

| Verb | What it does | Mechanics |
|---|---|---|
| `fade` | Animate the layer's opacity to `to` (default 1) over `duration`. Start a reveal layer at `opacity="0"`. | attribute + GPU transition — fully scrub-restorable |
| `pulse` | Attention oscillation (opacity 1 → 0.3 → 1, `cycles` ≈ duration/600ms) | per-frame channel, cleared cleanly at the end |
| `trace` | Progressive draw along a `TripsLayer`'s timestamps; the finished trail holds | per-frame `currentTime` sweep |
| `trace` + `feature-id` | **One feature draws itself on** — inside its own multi-feature layer (Polygon/MultiPolygon outline, or LineString) | see below |
| `populate` | **Rows drop in one by one** — the layer fills up over `duration` | filterRange sweep on the always-mounted GPU filter |

Whole-layer `trace` needs per-vertex time, which is TripsLayer's purpose — give it `get-path`/`get-timestamps` data and start `current-time` below the first timestamp. All verbs honor `prefers-reduced-motion` by jumping to the final state.

### Per-feature trace: `trace layer="regions" feature-id="mission"`

```html
<deck-step duration="3s" trace layer="regions" feature-id="mission"></deck-step>
```

The Mission polygon's outline draws itself around the perimeter (75% of the slot, at uniform speed — timestamps are synthesized from cumulative distance), then the real feature reappears as the outline dissolves. Everything underneath is runtime-managed and removed afterward: the target feature is hidden by swapping the layer's data to "everyone else" through the per-frame channel (the other features are untouched, and your markup never changes), and the outline is a temporary `TripsLayer` element the runtime appends and deletes — it never appears in the legend or `ctx.layers`. Optional `color="[255, 120, 40]"` styles the outline.

**Parallel traces compose.** Fire a trace for every feature in the same beat and they all draw at once — the hides accumulate (the layer shows the union of "everyone else", i.e. nothing, while all outlines draw) and each feature reveals independently as its own trace finishes:

```html
<deck-step duration="3s" trace layer="regions" feature-id="north-beach" parallel></deck-step>
<deck-step duration="3s" trace layer="regions" feature-id="mission" parallel></deck-step>
<deck-step duration="3s" trace layer="regions" feature-id="sunset" parallel></deck-step>
```

Start the layer at `visible="false"` and turn it on in the same beat for a "draws itself onto an empty map" opening.

It's also a plain action, so **click-to-trace** is one line — clicking any polygon in the layer makes it draw itself:

```html
<deck-behavior on="click" layer="regions" action="trace" duration="2s"></deck-behavior>
```

One honest limit: point features have no outline to trace, so they fall back to `fade`. Scrubbing a story mid-trace cleans up the temp layer and every patch.

### Populate: `populate layer="bikes"`

```html
<deck-step duration="2500ms" populate layer="bikes"></deck-step>
```

The layer's rows appear one by one until it's fully populated — each frame only moves the GPU filter's range bound (a uniform update, no data re-upload), and the patch clears itself so the layer ends on its authored props. Appearance **order**:

- the layer's own `filter-field`, if it has one — the sweep runs over the *authored* range, so rows appear in filter order and the end state *is* the authored filter (racks with `filter-field="size" filter-range="[0, 5]"` populate small-to-large, and anything outside the range stays correctly hidden);
- `field="capacity"` in the payload — order by any numeric field without authoring a filter;
- neither — data order.

Works from every dispatch surface, e.g. populate on load: `<deck-behavior on="load" action="populate" layer="bikes" duration="3s">`. GeoJSON layers need an authored `filter-field` to populate (a live composite can't take a runtime filter accessor); without one it warns and falls back to `fade`.

## Animation primitives (usable without stories)

- **Camera:** `map.flyTo(coords, zoom, { duration, curve })`, or the `fly-to` action (`center`/`zoom`/`pitch`/`bearing`/`duration`/`curve`) from any behavior or button; `zoom-to-feature` accepts `duration` too. `prefers-reduced-motion` is honored in both renderer modes — moves become instant, final state identical.
- **Layer props:** `transition="get-fill-color 800ms, get-radius 400ms"` GPU-animates prop changes; on a live layer, `transition="get-position 300ms"` makes entities glide between updates.

## Not yet (honestly)

Mid-step scrub interpolation; simultaneous multi-story playback; video export (app-level — use screen capture).
