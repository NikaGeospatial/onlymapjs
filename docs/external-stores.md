# External stores — Redux, MobX, Zustand, Jotai

OnlyMapJS exposes runtime state as a **framework-free store contract** on `MapController`: per watch-token `{subscribe, getSnapshot}` stores that anything can consume — React's `useSyncExternalStore` (the adapter's own hooks ride it internally), a MobX `autorun`, a Redux listener, a Zustand mirror. There are deliberately **no** `@nika-js/onlymap/redux`-style binding packages: every recipe below is ~20 lines against one contract, and each is covered by an integration test in the library's own suite, run against the real store libraries.

## The contract

```ts
const viewport = controller.getStore("viewport");
// { subscribe(cb): unsubscribe, getSnapshot(): Readonly<ViewportSnapshot> }

viewport.getSnapshot();
// { longitude, latitude, zoom, pitch, bearing,
//   bounds: [[west, south], [east, north]],
//   origin: "user" | "programmatic" }
```

Tokens and snapshot types:

| Token | Snapshot | Fires on |
|---|---|---|
| `"viewport"` | `ViewportSnapshot` (above) | every camera change |
| `"selection"` | `Selection \| null` | hover/click picks |
| `"layers"` | `LayerMetaSnapshot[]` (same shape as `ctx.layers`) | layer add/remove, visibility, filter, legend meta |
| `"data:<layerId>"` | `{ version, rows }` | that layer's data load / stream tick / poll |

Three guarantees the recipes rely on:

1. **Cached identity.** `getSnapshot()` returns the *same object* until that token's next event. Safe for `useSyncExternalStore` (a fresh object per call would render-loop) and for identity-based dirty checks in any store.
2. **Plain data.** Snapshots contain no functions — Redux devtools, `redux-persist`, and structured cloning all work. Rich reads (`project`, `data()`, `stats()`) stay on `ctx` / the controller.
3. **Version stamps, not rows.** `data:<id>` gives you `{version, rows}` — a change signal. Fetch actual rows with `ctx.data(id)` when the version moves; mirroring a 100k-row array into a store per stream tick is the classic performance anti-pattern.

## Origin tagging & echo suppression

Two-way camera binding creates a feedback loop: store → `flyTo` → map moves → map event → store. Two mechanisms break it:

- **`origin`** — every viewport change is tagged `"user"` (canvas gesture: drag, wheel, keyboard, double-click zoom) or `"programmatic"` (camera APIs, actions, story steps, transition frames). Both the settled signals (`om-view-changed` on `<om-map>`, `onViewChange` on `MapController`) **and** the viewport snapshot's `origin` field report the **burst** origin: if any change since the last settle was a gesture, it's `"user"` — so a drag's inertia tail can't relabel the gesture, and the snapshot can never disagree with the settled callback about the same gesture.
- **Idempotent setters** — writing an unchanged camera back to the map is a no-op frame, and the React adapter's camera props only call `setView` when values actually differ.

Sync **map → store** on the settled signal with `origin === "user"`; drive **store → map** through the camera APIs. Neither direction can then re-trigger the other.

## Redux Toolkit

```ts
import { createAction, createSlice, createListenerMiddleware, configureStore } from "@reduxjs/toolkit";

const viewportSynced = createAction<{ longitude: number; latitude: number; zoom: number }>("map/viewportSynced");
const flyToRequested = createAction<{ center: [number, number]; zoom?: number }>("map/flyToRequested");

// map → redux: settled camera only (never 60fps dispatches), user-originated only.
const controller = new MapController(el, {
  onViewChange: (view, origin) => {
    if (origin !== "user") return; // echo suppression: your own flyTo settles as "programmatic"
    store.dispatch(viewportSynced({ longitude: view.longitude, latitude: view.latitude, zoom: view.zoom }));
  },
});

// redux → map: intents flow through the camera API.
listener.startListening({
  actionCreator: flyToRequested,
  effect: (action) => controller.flyTo(action.payload.center, action.payload.zoom),
});
```

Store only the plain camera fields (Redux's serializability rule is why snapshots are plain data), and keep dispatches on the settled signal — Redux's own guidance warns against 60fps viewport dispatch.

## Zustand

A mirror store — Zustand's own `useSyncExternalStore` integration does the React half:

```ts
import { createStore } from "zustand/vanilla"; // or `create` from "zustand" in React

const useMapState = createStore(() => ({
  viewport: controller.getStore("viewport").getSnapshot(),
  selection: null as Selection | null,
}));
controller.getStore("viewport").subscribe(() =>
  useMapState.setState({ viewport: controller.getStore("viewport").getSnapshot() }));
controller.getStore("selection").subscribe(() =>
  useMapState.setState({ selection: controller.getStore("selection").getSnapshot() }));

// any component: const zoom = useMapState(s => s.viewport.zoom)
```

The mirror is read-only, so there is no echo to suppress; write back via `controller.setView`/`flyTo` from your actions.

## MobX (and mobx-keystone)

An observable mirror plus a reaction for write-back — the mirror update never re-fires the reaction because the *intent* observable (`targetCenter`) is separate from the mirrored state:

```ts
class MapVM {
  viewport = controller.getStore("viewport").getSnapshot();
  targetCenter: [number, number] | null = null;
  constructor() {
    makeAutoObservable(this, {}, { autoBind: true });
    controller.getStore("viewport").subscribe(() =>
      runInAction(() => { this.viewport = controller.getStore("viewport").getSnapshot(); }));
    reaction(() => this.targetCenter, (c) => c && controller.flyTo(c));
  }
}
```

mobx-keystone is the same bridge with a lifecycle-scoped subscription — return the unsubscribe from `onAttachedToRootStore` and keystone disposes it when the model detaches:

```ts
@model("app/MapState")
class MapState extends Model({ viewport: prop<ViewportSnapshot>() }) {
  onAttachedToRootStore() {
    return controller.getStore("viewport").subscribe(() =>
      this.setViewport(controller.getStore("viewport").getSnapshot()));
  }
  @modelAction setViewport(v: ViewportSnapshot) { this.viewport = v; }
}
```

## Jotai

```ts
import { atom } from "jotai";

const viewportAtom = atom(controller.getStore("viewport").getSnapshot());
viewportAtom.onMount = (set) =>
  controller.getStore("viewport").subscribe(() => set(controller.getStore("viewport").getSnapshot()));
```

Caveat, documented rather than solved: jotai deliberately avoids `useSyncExternalStore` (uSES updates are always sync-urgent, incompatible with time-slicing). Updates through `set` here are transition-friendly, which also means a jotai read can briefly trail the map during a transition — for camera state that is normally what you want.

## The HTML front-end

`getStore` lives on `MapController` — the programmatic/React lane — by design. The HTML manifest lane's subscription surface is DOM events and widget `watch` tokens: listen for `om-view-changed` (its `detail.origin` carries the same burst-origin signal) and read state from widget `ctx`. If you're embedding `<om-map>` and need full store bridging, drive the map through `MapController` instead — the two front-ends share one core.

## React without a store

You usually don't need any of the above — `useOmMap(["viewport", "selection"])` already rides `useSyncExternalStore` over these same stores, with tearing-safe reads and stable `ctx` identity between events. Reach for an external store when map state must live alongside app state (routing, persistence, devtools, cross-page continuity).
