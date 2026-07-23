# Chapter 33: Hydration

## 1. Overview

Server-Side Rendering (SSR) in Angular produces HTML on the server so the browser has something to paint immediately — better perceived performance, better SEO, better Core Web Vitals (especially LCP and FCP). But historically, once that server HTML reached the browser, Angular's bootstrap process on the client threw it away. The client app rendered its own component tree from scratch, tore out the server-rendered DOM, and replaced it node-for-node. This is called **destructive hydration** (destroy-and-rebuild).

Destructive rendering has real costs:

- **Visible flicker** — the user sees server HTML, then a blank flash, then the same content reappears once the client re-renders it.
- **Layout shift** — re-created nodes can lose scroll position, focus, form state, and CSS-triggered transitions/animations restart.
- **Wasted work** — the browser parsed and laid out HTML that gets discarded, then does that work again.
- **Lost user input** — if a user starts interacting (e.g., typing in a field, or clicking a button) before hydration finished, those events were lost.

Angular v16 introduced **non-destructive full hydration**, made stable in v17, and enabled by default for new SSR projects from v17 onward via `provideClientHydration()`. Instead of discarding server DOM, Angular walks the existing DOM tree and the internal view (the "TView"/"LView" data structures) side-by-side, matching each server-rendered node to its corresponding internal Angular view node, reusing it, and only attaching bindings/listeners. It also introduced **event replay** (stable in v18 via `withEventReplay()`, part of the default hydration config from v19) so that clicks/inputs that happen before hydration completes are captured and replayed once the app is interactive, rather than lost. Angular v17's `@defer` block and later **incremental hydration** (`withIncrementalHydration()`, developer preview from v19) let you hydrate parts of the page lazily — only when they scroll into view, become idle, or meet another trigger — instead of hydrating the entire page eagerly on load.

This chapter covers how non-destructive hydration works internally, how to configure it, how it interacts with `@defer`, how event replay and hydration boundaries work, the common mismatch errors you'll hit in real projects, and how hybrid rendering (per-route SSR/SSG/CSR selection) fits into the picture.

---

## 2. Core Concepts

### 2.1 Destructive vs. Non-Destructive Hydration

**Destructive (pre-v16 default when SSR was used without hydration):**
1. Server renders full HTML, sends it to the browser.
2. Browser paints server HTML (fast FCP).
3. Angular bootstraps on the client.
4. Client-side change detection runs, components render their views as if starting from an empty document.
5. Angular replaces the server-rendered DOM subtree with the freshly created DOM subtree.

**Non-destructive (`provideClientHydration()`):**
1. Server renders full HTML, and additionally serializes **TransferState** data (e.g., HTTP responses fetched during SSR) plus hydration annotations (`ngh` attributes) that describe the shape of the rendered view tree.
2. Browser paints server HTML.
3. Angular bootstraps on the client but instead of creating new DOM nodes, it **walks the existing DOM in the same traversal order** the renderer would have used, and **claims** each node into the corresponding internal view data structure.
4. Event listeners are attached to the *existing* nodes; component/directive instances are constructed and bound to those nodes.
5. TransferState prevents duplicate HTTP calls (e.g., `HttpClient` requests made during SSR are cached and replayed on the client instead of re-fetched) and preserves other manually transferred state.

Net effect: no flicker, no re-layout, no duplicate network calls, and DOM state (like `<input>` values already typed by a fast user, or CSS animations) survives.

### 2.2 `provideClientHydration()` and Its Feature Providers

`provideClientHydration()` lives in `@angular/platform-browser` and is passed to `bootstrapApplication()`'s providers (or registered in `AppModule` via `provideClientHydration()` inside `providers` for NgModule-based apps, or `BrowserModule.withServerTransition()`'s successor).

It accepts optional **hydration features**, each itself a function that returns a `HydrationFeature`:

| Feature | Purpose |
|---|---|
| `withEventReplay()` | Captures DOM events dispatched before hydration finishes and replays them once listeners are attached. Default-on from Angular v19. |
| `withI18nSupport()` | Enables hydration for applications using `$localize` / Angular i18n blocks, which otherwise require extra reconciliation because translated text nodes can differ in structure from source templates. |
| `withIncrementalHydration()` | Enables hydration boundaries to be driven by `@defer` triggers, so parts of the tree hydrate independently and lazily rather than all at once on bootstrap. Requires `@defer` blocks in the template. |
| `withHttpTransferCacheOptions()` | Configures which HTTP requests are eligible for TransferState caching/replay (e.g., include POST requests, set a max payload size, filter by headers). |
| `withNoHttpTransferCache()` | Disables the HTTP transfer cache entirely. |

Without any hydration features, `provideClientHydration()` alone gives you the baseline non-destructive DOM reconciliation only.

### 2.3 Skew Protection (Version Skew)

When an app is deployed behind a CDN and a new build goes out while old clients still have the previous build's JS cached (or mid-navigation), you can get **version skew**: the server renders HTML annotated for build N+1 hydration data shapes, but the browser is still running build N's JS, or vice versa. Angular's hydration mechanism embeds a build identifier in transferred state; if the client-side hydration logic detects the served HTML doesn't match what it expects structurally, it falls back to destructive rendering for the affected subtree (or the whole app, depending on severity) rather than silently producing a broken DOM. This is sometimes discussed under "skew protection" together with the general full-stack skew-protection features some meta-frameworks (like Angular's own deployment guidance, or platforms akin to Vercel) provide by keeping old server assets alive until clients have migrated. Angular itself doesn't solve deployment-level skew (that's an infra concern — versioned URLs/asset directories), but its hydration validation guards against corrupting the DOM if the shapes don't line up.

### 2.4 Event Replay

Before hydration finishes attaching real Angular listeners, the page is fully painted and technically "clickable," but nothing would happen if a user clicked a button — the listener isn't wired up yet. `withEventReplay()` solves this by installing lightweight, framework-agnostic **capture-phase** event listeners on the document as soon as the server HTML is parsed (this is inlined as a tiny bootstrap script, conceptually similar to Google's "Jsaction"/Wiz event-dispatch pattern). These global listeners record `(event, target)` pairs into a buffer. Once Angular hydrates the corresponding component and attaches its real listener, the buffered events matching that target are **replayed** — dispatched again — so the click the user made during the loading window still triggers the intended handler, in the same order they originally occurred.

### 2.5 Hydration Boundaries and `@defer`

`@defer` blocks already deferred rendering of some template regions on the client (see the Deferrable Views chapter). Under hydration, each `@defer` block forms a natural **hydration boundary**: content inside a `@defer` block that is server-rendered (via `@defer` + `@placeholder`, or SSR rendering the primary content directly, depending on config) is treated as a separate chunk that Angular can hydrate on its own schedule, independent from the rest of the page.

With **incremental hydration** (`withIncrementalHydration()`), the existing `@defer` triggers (`on viewport`, `on interaction`, `on hover`, `on idle`, `on timer`, `on immediate`, and custom `when` conditions) are reused, but now they don't just control *rendering* — they control *hydration timing*. A new trigger form, `hydrate on <trigger>` / `hydrate when <condition>` / `hydrate never`, lets you explicitly separate "when to render on the client if not server-rendered" from "when to hydrate if it *was* server-rendered." For example:

- `@defer (hydrate on viewport)` — the block is server-rendered (so content is visible/SEO-indexable immediately), but its Angular listeners/component instances are not attached until the block scrolls into the viewport.
- `@defer (hydrate on interaction)` — hydrates only on first click/keydown/focus within the block, deferring the JS cost of interactive widgets (e.g., a comments section) until the user actually engages with them.
- `@defer (hydrate never)` — the block stays static/non-interactive HTML forever on initial load; useful for content that's purely presentational.

This is the mechanism that lets a large page hydrate its "above the fold" interactive widgets first and defer everything below the fold, cutting Time To Interactive without cutting First Contentful Paint.

### 2.6 `ngSkipHydration`

Sometimes a component's DOM structure is fundamentally incompatible with hydration matching — most commonly because a **third-party library manipulates the DOM directly** (jQuery plugins, some charting libraries, ad embeds) in a way Angular didn't render and can't reconcile. For these, you opt a subtree out of hydration entirely using the `ngSkipHydration` attribute (or the `[ngSkipHydration]="expr"` binding) on the host element of the offending component:

```html
<my-legacy-widget ngSkipHydration></my-legacy-widget>
```

Angular then falls back to destructive rendering (destroy + recreate) for just that component's subtree, while the rest of the page continues to hydrate normally. This is an escape hatch, not a fix — every skipped subtree reintroduces flicker locally.

### 2.7 Hybrid Rendering / Server Routing (Angular v19+)

Angular's newer SSR tooling (`@angular/ssr`, integrated with the Angular CLI's application builder) supports **hybrid rendering**: different routes of the same application can be configured to use different rendering strategies — full SSR, prerendering/SSG (Static Site Generation at build time), or client-side rendering only — via a server routing configuration (`app.routes.server.ts` with `RenderMode.Server`, `RenderMode.Prerender`, or `RenderMode.Client` per route). Hydration applies per-page based on how that page was rendered: a prerendered/SSG route hydrates the same way an SSR route does (both produce static HTML the client reconciles against); a `RenderMode.Client` route skips hydration because there was no server DOM to reconcile — the client renders normally from an empty shell.

This matters for interviews because it shows you understand hydration is not "an SSR-only feature" in the narrow sense — it's the reconciliation step for *any* route where HTML pre-existed before the Angular client app took over, regardless of whether that HTML came from a live server render or a build-time static file.

---

## 3. Code Examples

### 3.1 Basic Setup — `provideClientHydration()` with Event Replay and i18n Support

```typescript
// app.config.ts
import { ApplicationConfig } from '@angular/core';
import {
  provideClientHydration,
  withEventReplay,
  withI18nSupport,
  withHttpTransferCacheOptions,
} from '@angular/platform-browser';
import { provideHttpClient, withFetch } from '@angular/common/http';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(withFetch()),
    provideClientHydration(
      withEventReplay(),
      withI18nSupport(),
      withHttpTransferCacheOptions({
        includePostRequests: true,
        maxPayloadSizeBytes: 5_000_000,
      }),
    ),
  ],
};
```

```typescript
// main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { AppComponent } from './app/app.component';
import { appConfig } from './app/app.config';

bootstrapApplication(AppComponent, appConfig).catch((err) =>
  console.error(err),
);
```

```typescript
// main.server.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { AppComponent } from './app/app.component';
import { config } from './app/app.config.server';

const bootstrap = () => bootstrapApplication(AppComponent, config);
export default bootstrap;
```

```typescript
// app.config.server.ts
import { mergeApplicationConfig, ApplicationConfig } from '@angular/core';
import { provideServerRendering } from '@angular/platform-server';
import { appConfig } from './app.config';

const serverConfig: ApplicationConfig = {
  providers: [provideServerRendering()],
};

export const config = mergeApplicationConfig(appConfig, serverConfig);
```

### 3.2 Incremental Hydration with `@defer`

```typescript
// app.config.ts
import { ApplicationConfig } from '@angular/core';
import {
  provideClientHydration,
  withEventReplay,
  withIncrementalHydration,
} from '@angular/platform-browser';

export const appConfig: ApplicationConfig = {
  providers: [
    provideClientHydration(withEventReplay(), withIncrementalHydration()),
  ],
};
```

```html
<!-- product-page.component.html -->
<app-hero-banner></app-hero-banner>

<!-- Server-rendered immediately for SEO/LCP, but the client only
     attaches Angular's interactive listeners once it scrolls into view. -->
@defer (hydrate on viewport) {
  <app-recommendations [productId]="productId()" />
} @placeholder {
  <div class="skeleton skeleton--recs"></div>
}

<!-- Hydrates only when the user actually interacts with it -->
@defer (hydrate on interaction) {
  <app-review-form [productId]="productId()" />
} @placeholder {
  <button type="button" class="write-review-cta">Write a review</button>
}

<!-- Never hydrated on load; purely decorative/static after first paint -->
@defer (hydrate never) {
  <app-related-articles [tags]="tags()" />
}

<!-- Standard client-only rendering trigger still works alongside hydrate triggers -->
@defer (on idle; hydrate on idle) {
  <app-chat-widget />
} @loading {
  <span>Loading chat…</span>
}
```

```typescript
// product-page.component.ts
import { Component, input } from '@angular/core';

@Component({
  selector: 'app-product-page',
  standalone: true,
  templateUrl: './product-page.component.html',
})
export class ProductPageComponent {
  productId = input.required<string>();
  tags = input<string[]>([]);
}
```

Note: `hydrate on viewport` / `hydrate on interaction` etc. only take effect when `withIncrementalHydration()` is registered; otherwise Angular hydrates the whole tree eagerly (ignoring the `hydrate` sub-triggers) as soon as bootstrap completes, while still honoring the plain `on ...` triggers for whether to *render* client-side content that wasn't server-rendered.

### 3.3 Opting a Third-Party Widget Out of Hydration

```html
<!-- map.component.html -->
<div class="map-container" ngSkipHydration>
  <div #mapEl id="leaflet-map"></div>
</div>
```

```typescript
// map.component.ts
import { AfterViewInit, Component, ElementRef, viewChild } from '@angular/core';
import * as L from 'leaflet';

@Component({
  selector: 'app-map',
  standalone: true,
  templateUrl: './map.component.html',
})
export class MapComponent implements AfterViewInit {
  private mapEl = viewChild.required<ElementRef<HTMLDivElement>>('mapEl');

  ngAfterViewInit(): void {
    // Leaflet manipulates the DOM directly (adds tiles, controls, etc.)
    // in ways Angular's hydration cannot reconcile against server HTML.
    L.map(this.mapEl().nativeElement).setView([51.505, -0.09], 13);
  }
}
```

---

## 4. Internal Working

### 4.1 DOM-to-View Matching (Reconciliation)

Angular's renderer normally creates DOM nodes while walking the compiled template instructions (`ɵɵelementStart`, `ɵɵtext`, `ɵɵelementEnd`, etc.) and stores references to those nodes inside the `LView` (the runtime array holding a component instance's bindings, DOM node references, child view references, and directive instances).

During hydration, the server serializes a compact tree-shape descriptor into the `ngh` (Angular Hydration) `<script>` tag embedded in the HTML (identified by an app-specific id, e.g. `<script id="ng-state" type="application/json">`, plus per-host-element `ngh` attributes referencing indices into that data). This descriptor encodes, per component/embedded view, how many nodes to expect and their nesting shape (container markers for `@if`/`@for`/`ng-template` boundaries, text node counts, etc.) — not a full virtual DOM, just enough shape information to walk in lockstep.

On the client:
1. Angular starts the same template-instruction walk it always would (same `ɵɵelementStart`/`ɵɵtext` sequence), but each "create node" instruction, instead of calling `document.createElement`/`createTextNode`, calls into the **hydration-aware renderer**, which consults the current position in the existing DOM (tracked via a cursor/pointer into `Node.firstChild`/`nextSibling`) and **claims** that existing node for this slot rather than creating a new one.
2. The claimed node is written into the `LView` at the correct index, exactly as if it had just been created — so all further Angular machinery (change detection, `Renderer2` calls, directive host bindings) operates on it identically to non-hydrated node creation.
3. Container boundaries (from structural directives, `@if`, `@for`, `@defer`) are marked in the server HTML with hydration comment markers (e.g., `<!--ngh-container-->`-style markers akin to Angular's existing `<!--container-->` comments used for `ng-template` anchors), so the client cursor knows where each dynamic view starts/ends without guessing from content alone.
4. If, at any point, the shape the client expects (based on its own compiled template) doesn't match what it finds in the DOM (wrong tag name, missing node, mismatched container marker), Angular logs a **hydration mismatch error**, discards the mismatched subtree, and falls back to creating fresh DOM for it (destructive rendering, scoped as narrowly as Angular can manage) rather than corrupting the page silently.

This is why hydration requires **the exact same render output** on server and client for a given piece of state: the matching algorithm is a structural walk with no fuzzy matching — it assumes node N at position P is the node that N will produce again, and reuses it under that assumption.

### 4.2 Event Replay Buffering Mechanism

1. As part of the SSR output, Angular injects a small, dependency-free bootstrap script into the `<head>` (or inlined near the top of `<body>`) that runs **before** the main Angular bundle downloads/parses/executes.
2. That script attaches capture-phase listeners at the document root for a fixed set of event types considered "replayable" (click, input, change, submit, keydown, etc. — the same category of events the Angular event system normally binds via delegation).
3. Each captured event is pushed into an in-memory queue along with a reference to `event.target` (and enough path info to re-locate it), without calling `preventDefault()`/`stopPropagation()` — the native browser behavior for that event still happens.
4. Once the real Angular application bootstraps and hydrates a given component, Angular's event delegation system (which normally attaches listeners once, at a shared ancestor, and dispatches based on target — similar in spirit to synthetic event systems in other frameworks) checks the buffer: any queued events whose target now maps to a hydrated element with a real registered listener get **dispatched again**, in original order, invoking the actual component method (e.g., `(click)="addToCart()"`).
5. After the whole tree finishes hydrating (or after a timeout), the temporary capture listeners and the queue are torn down; from that point on, event handling is purely the normal Angular event system.

This gives the illusion that the page was interactive the whole time, even though real listeners attach progressively.

### 4.3 Hydration Boundaries for `@defer`

Each `@defer` block compiles to a special kind of embedded view container with its own loading state machine (`Placeholder → Loading → Complete/Error`). When SSR renders a `@defer` block's primary content (rather than just its placeholder), the emitted HTML is tagged with a boundary marker so the client's hydration cursor knows: "everything between this marker and its matching end marker belongs to defer-block instance #k, and it should not be walked/claimed until block #k's hydration trigger fires."

Without `withIncrementalHydration()`, Angular still respects these markers structurally (so DOM nodes are matched correctly once hydration reaches that region) but performs the hydration walk for the whole page in one pass at bootstrap. With `withIncrementalHydration()` enabled, each boundary becomes an independent **hydration scheduling unit**: Angular registers the appropriate trigger (an `IntersectionObserver` for `on viewport`, a `requestIdleCallback` for `on idle`, event listeners for `on interaction`/`on hover`, a timer for `on timer`, etc.) and only performs the DOM-claiming walk plus listener attachment for that boundary's subtree when the trigger condition is satisfied. Nested `@defer` blocks form nested boundaries, each independently schedulable, which is what allows large pages to have a fast, small "critical hydration path" while long-tail content hydrates opportunistically.

---

## 5. Edge Cases & Gotchas

### 5.1 Non-Deterministic Rendering Breaks Matching

Anything that produces different output on server vs. client render will trigger hydration mismatches:

```typescript
// BAD — server renders one timestamp, client "re-renders" a different one internally,
// causing a text-node mismatch warning (Angular still reconciles but logs an error).
@Component({
  selector: 'app-clock',
  template: `<span>{{ now }}</span>`,
})
export class ClockComponent {
  now = Date.now(); // different value at SSR time vs. hydration time
}
```

```typescript
// BAD — random IDs used for aria-describedby / element identity
id = `field-${Math.random().toString(36).slice(2)}`;
```

```typescript
// BAD — locale/timezone-dependent formatting differing between
// the server's locale/env and the browser's locale
new Date().toLocaleString(); // server TZ may differ from client TZ
```

**Fixes:** generate IDs deterministically (e.g., `@Injectable` incrementing counter seeded identically both sides, or Angular's own `_id`-style unique-id services which are hydration-aware), pass `now`/random seeds down from a single source computed once and transferred via TransferState, and pin server-side `Intl`/timezone handling to match what you intend to render (or defer locale-sensitive formatting to run only on the client inside `afterNextRender`/`afterRender` so it doesn't need to match SSR output structurally).

### 5.2 Direct DOM Manipulation Before Hydration Completes

If application code (or a library) reaches into the DOM outside Angular's rendering pipeline — e.g., `document.querySelector(...).classList.add(...)` in a constructor, or a script tag manipulating markup — before hydration finishes claiming that region, Angular's cursor can find an unexpected attribute/child count and abort hydration for that subtree. Common offenders:

- Analytics/tag-manager scripts that inject nodes into the body before Angular hydrates.
- Browser extensions (password managers, translators) rewriting DOM structure — largely unavoidable, but worth knowing this is a real source of mismatches in production that don't reproduce locally.
- Manual `Renderer2`/`ElementRef.nativeElement` mutations performed in a component's constructor or field initializer (runs before hydration/AfterViewInit-style boundaries are settled) rather than in a lifecycle hook meant for post-render work.

**Fix:** perform imperative DOM work inside `afterNextRender()` (runs once, client-only, after the first render/hydration pass) or `afterRender()`, never directly in constructors, and audit third-party scripts that run eagerly.

### 5.3 Third-Party Widgets Breaking DOM Matching

Libraries like jQuery plugins, some chart libraries, embeddable widgets (maps, video players with custom chrome), or ad iframes frequently rewrite their container's inner DOM structure after Angular has server-rendered a "seed" version of it. Hydration's structural walk then finds nodes that don't match what the client-side template compilation expects, producing repeated mismatch warnings and forcing that subtree into destructive fallback anyway — at which point you've paid the SSR cost with none of the hydration benefit for that component.

**Fix:** mark the exact host element with `ngSkipHydration` so Angular skips reconciliation for that subtree from the start (clean, intentional destructive rendering, isolated to just that component) instead of discovering the mismatch at runtime.

### 5.4 `NgZone`/Change Detection Timing With Deferred Hydration

With incremental hydration, code that assumes "the whole page is interactive once `ApplicationRef.isStable` fires" needs revisiting — parts of the page may intentionally remain un-hydrated (not yet interactive) well after the app is "stable" in the traditional sense, since `hydrate on interaction`/`on viewport` boundaries wait for their own triggers. Tests or monitoring that wait for full-page interactivity as a single signal should account for per-boundary hydration state instead.

### 5.5 SSR + TransferState Duplication Gotcha

If `withHttpTransferCacheOptions()`/default HTTP transfer caching isn't hitting because a request's cache key differs between server and client (e.g., a header only sent client-side, or a body serialized differently for POST requests), the client will silently re-fetch — not a hydration *error*, but a common performance regression mistaken for "hydration isn't caching my API calls." Verify cache keys (URL + method + params, and body for POST when `includePostRequests: true`) match exactly between the two environments.

---

## 6. Interview Questions & Answers

**Q1. What problem does hydration solve that plain SSR alone doesn't?**
A: Plain SSR only solves the "nothing to paint until JS loads" problem — it gets HTML in front of the user fast. But without hydration, once Angular's client bundle boots, it destroys that server HTML and rebuilds the entire DOM from scratch (destructive rendering), causing visible flicker, loss of scroll/focus/animation state, and duplicated work. Hydration reuses the existing server-rendered DOM nodes instead of recreating them, eliminating that flicker and rework while still letting Angular attach live bindings and listeners to make the page interactive.

**Q2. How do you enable hydration in an Angular application?**
A: Add `provideClientHydration()` (from `@angular/platform-browser`) to the application's providers — typically in `app.config.ts` for standalone bootstrap, merged into both the client and server app configs. It requires the app to already be configured for SSR (`@angular/ssr`, `provideServerRendering()` on the server side). From Angular v17+ new projects created with SSR enabled include this by default.

**Q3. What is `withEventReplay()` and why is it necessary?**
A: There's a window between "server HTML is painted" and "Angular has hydrated and attached real event listeners" during which the page looks interactive but isn't. Without event replay, a user who clicks a button in that window gets nothing — the click is lost. `withEventReplay()` installs lightweight global capture-phase listeners immediately, buffers events by target, and replays them against the real Angular listener once that component hydrates, so early interactions aren't lost. It became Angular's default hydration behavior from v19.

**Q4. Explain how Angular decides whether to reuse or recreate a DOM node during hydration.**
*Interviewer intent:* Checking whether the candidate understands this is a structural, position-based algorithm and not some fuzzy diffing (unlike, say, a virtual-DOM reconciler with keys).
A: Angular performs the same template-instruction walk it always performs when creating a view (`ɵɵelementStart`, `ɵɵtext`, etc.), but the renderer used during hydration is hydration-aware: instead of calling `createElement`/`createTextNode`, it advances a cursor through the existing DOM (via `firstChild`/`nextSibling`) and claims whatever node it finds at that position, storing it into the `LView` as if it had just been created. Container boundaries from structural directives (`@if`, `@for`, `@defer`) are marked with comment-node anchors in the server HTML so the cursor knows where nested views start/end. If the node found doesn't structurally match what the compiled template expects (wrong tag, missing node, mismatched container marker), Angular emits a hydration mismatch error and falls back to destructively creating fresh DOM for that subtree.

**Q5. What causes a "hydration mismatch" error, and give three concrete real-world examples.**
A: A mismatch happens whenever the DOM shape rendered on the server differs from what the client's compiled template produces when it walks the same code path. Common causes: (1) non-deterministic values like `Date.now()`, `Math.random()`, or environment-dependent `Intl`/locale formatting that differ between server and client render; (2) third-party scripts or browser extensions mutating the DOM before hydration reaches that region; (3) manual `nativeElement`/`Renderer2` DOM manipulation performed in a constructor or field initializer instead of a post-render hook, altering structure before Angular's cursor gets there.

**Q6. How would you exclude a component from hydration, and when is that appropriate?**
A: Add the `ngSkipHydration` attribute (or `[ngSkipHydration]` binding) to the component's host element. Angular then destructively renders that subtree (destroy any server DOM, create fresh client DOM) instead of attempting reconciliation. It's appropriate for components wrapping third-party widgets that manipulate the DOM directly in ways Angular can't predict/reconcile (map libraries, some chart libraries, jQuery-based plugins) — trying to hydrate them normally would produce mismatch errors and force an uncontrolled fallback anyway, so explicitly opting out is cleaner and intentional.

**Q7. What is incremental hydration and what problem does it solve?**
*Interviewer intent:* Distinguishing "hydration happens" from "hydration happens *eagerly, all at once*" — a subtlety many mid-level candidates miss.
A: By default, non-destructive hydration still hydrates the entire rendered page in one pass as soon as the client app bootstraps — every component's listeners get attached up front, which costs CPU/JS-parse time proportional to total page complexity, even for content far below the fold that the user may never interact with. Incremental hydration (`withIncrementalHydration()`) lets `@defer` blocks control *when* their subtree hydrates, independent of whether it was rendered eagerly. Using `hydrate on viewport`, `hydrate on interaction`, `hydrate on idle`, or `hydrate never` on a `@defer` block, you get server-rendered HTML immediately (for LCP/SEO) but defer the JS cost of making that specific section interactive until it's actually needed, reducing Time-to-Interactive on large pages.

**Q8. What's the difference between `on viewport` and `hydrate on viewport` in a `@defer` block?**
A: `on viewport` (without `hydrate`) controls whether the deferred content is *rendered at all* on the client if it wasn't already present — i.e., it's the standard deferrable-views trigger for lazy-loading a block's code and creating its view when it scrolls into view. `hydrate on viewport` instead assumes the content *was* server-rendered (so it's already visible as static HTML) and controls when Angular attaches live bindings/listeners to that already-present DOM. You can combine both: `@defer (on viewport; hydrate on viewport)` covers the case where the block might not have been server-rendered too.

**Q9. Explain the internal mechanism of event replay — how are early click events actually buffered and re-triggered?**
A: Before the main Angular bundle executes, SSR output includes a small inline bootstrap script that attaches capture-phase native event listeners at the document root for replayable event types (click, input, submit, etc.). Each event that fires is pushed into a queue with a reference to its target, without interfering with native default behavior. Once Angular's real event-delegation system (attached at hydration time per component) registers a listener for that same target, the queue is drained: buffered events matching now-hydrated targets are re-dispatched in original order so the real component handler runs, exactly as if the user's original interaction happened after hydration. After the tree finishes hydrating, the temporary listeners and queue are discarded.

**Q10. Why can't Angular just diff the server DOM against a virtual DOM the way React does with hydration, and does Angular really work differently from React here?**
*Interviewer intent:* Tests whether the candidate can articulate the actual mechanism rather than hand-waving "it's like React." Angular has no virtual DOM; both frameworks nonetheless hydrate via structural, positional matching rather than deep diffing — the answer should surface that similarity precisely instead of asserting a false contrast.
A: Angular has no virtual DOM at all — it compiles templates directly into imperative instruction functions that manipulate real DOM nodes and a runtime `LView` data structure. Hydration works by re-running that same imperative instruction sequence but substituting node creation with node claiming from a positional cursor over the existing DOM, guided by lightweight shape/boundary markers serialized from the server render. This is actually conceptually close to how React's hydration also works (React doesn't build two virtual DOM trees and diff them for hydration either — it also walks the existing DOM positionally and attaches fiber nodes to matching elements, warning on mismatch). The real distinguishing point is that Angular's compiled-template approach means the "expected shape" comes directly from compiled instructions rather than from re-executing component render functions, but both frameworks fundamentally rely on structural/positional matching, not deep semantic diffing, and both fail similarly when server and client output diverge.

**Q11. A production app shows hydration mismatch warnings in the console but nothing visibly breaks — is it safe to ignore?**
A: Not safely, no. When a mismatch is detected, Angular falls back to destructively re-rendering the affected subtree, which silently reintroduces the flicker/rework/lost-state problems hydration was meant to eliminate — just scoped to that subtree instead of the whole page. It may look fine visually because destructive fallback still produces correct-looking output, but you lose the performance benefit locally, may lose in-flight user input in that region, and repeated warnings at scale (e.g., inside a `@for` loop) can add up to meaningful hydration cost across a page. Treat mismatch warnings as bugs to fix (deterministic rendering, `ngSkipHydration` for genuinely irreconcilable widgets), not noise.

**Q12. What is hybrid rendering in modern Angular, and how does hydration behave differently across its rendering modes?**
A: Hybrid rendering (via `@angular/ssr`'s server routing config, `app.routes.server.ts`) lets different routes of the same app use different rendering strategies: `RenderMode.Server` (SSR per-request), `RenderMode.Prerender` (build-time static generation/SSG), or `RenderMode.Client` (no server render, plain CSR). Hydration applies to any route that had pre-existing HTML for the client to reconcile against — both `Server` and `Prerender` routes hydrate via the same non-destructive mechanism (the static HTML from SSG is treated exactly like a per-request SSR response for hydration purposes). `RenderMode.Client` routes have no server DOM to reconcile, so Angular bootstraps and renders normally from an empty root with no hydration step involved.

**Q13. Your app uses `Math.random()` to generate a `key` for `@for` loop tracking. Could this cause a hydration issue, and how would you fix it?**
A: Yes — if the random value differs between server render and client hydration, Angular's `@for` block will see different track values for what should be "the same" logical item, which can confuse the hydration cursor's container-boundary matching (it expects the same number/identity of tracked views in the same order) and trigger mismatches, in addition to breaking `@for`'s own change-detection reuse optimizations independent of hydration. Fix: use a stable, deterministic identity for `track` — the item's actual id from your data model — never a value generated freshly on each render pass.

**Q14. How would you debug a hydration mismatch in a real application?**
A: Start with the browser console — Angular logs a descriptive hydration error identifying the component/DOM location where the mismatch occurred. Reproduce with SSR output directly: view the server-rendered HTML (curl the SSR endpoint or view-source before JS executes) and compare it structurally to what the same component renders client-side with `ng serve` (no SSR) for the same inputs/state. Look specifically for: values sourced from `Date`/`Math.random`/`Intl` without a shared/transferred seed, conditional template branches depending on `isPlatformBrowser`/`isPlatformServer` (a very common source, since branching on platform guarantees different output shapes server vs. client), and any third-party script/library touching that DOM region. Once isolated, either make the source deterministic (pass shared state via TransferState instead of computing independently on each side) or, for genuinely irreconcilable third-party DOM, apply `ngSkipHydration` to that component's host.

---

## 7. Quick Revision Cheat Sheet

- **Destructive hydration (old default without hydration APIs):** server HTML painted, then thrown away and rebuilt client-side → flicker, lost state, wasted work.
- **Non-destructive hydration:** `provideClientHydration()` reuses existing server DOM nodes by walking them positionally and claiming them into Angular's internal view (`LView`), instead of recreating them.
- **Enable:** `provideClientHydration(...)` in `app.config.ts`; requires SSR already set up (`provideServerRendering()` server-side).
- **`withEventReplay()`:** buffers early user events (click/input/etc.) via temporary capture-phase listeners before hydration finishes, replays them against real listeners once attached. Default since v19.
- **`withI18nSupport()`:** required for hydration to work correctly with `$localize`/i18n blocks.
- **`withIncrementalHydration()`:** lets `@defer` blocks control *when* their already-server-rendered content gets hydrated (`hydrate on viewport|interaction|hover|idle|timer`, `hydrate never`), separate from whether/when it's rendered.
- **`withHttpTransferCacheOptions()` / `withNoHttpTransferCache()`:** control TransferState caching of HTTP calls made during SSR so the client doesn't re-fetch.
- **`ngSkipHydration`:** attribute to force destructive rendering for one subtree — use for third-party widgets that mutate the DOM in ways Angular can't reconcile.
- **Matching algorithm:** positional/structural walk using compiled template instructions + serialized shape/boundary markers (`ngh` data, container comment anchors) — not a deep virtual-DOM diff; no fuzzy matching.
- **Mismatch causes:** `Date.now()`, `Math.random()`, locale/timezone differences, platform-branching (`isPlatformBrowser`/`isPlatformServer`) producing different template branches, third-party DOM mutation before hydration, imperative DOM writes in constructors instead of `afterNextRender()`.
- **On mismatch:** Angular logs an error and falls back to destructive rendering for just the affected subtree — not a silent no-op, it has a real performance/UX cost.
- **`@defer` + hydration:** each `@defer` block is a hydration boundary; without incremental hydration the whole page still hydrates in one pass at bootstrap, just with matching preserved across boundaries.
- **Hybrid rendering:** per-route `RenderMode.Server` / `RenderMode.Prerender` / `RenderMode.Client` via server routing config; hydration applies to `Server` and `Prerender` routes identically, not to `Client` routes (no server DOM to reconcile).
- **Debugging checklist:** compare SSR HTML vs. plain CSR HTML for the same component; hunt for non-deterministic values, platform-guard branching, and third-party scripts; fix via deterministic/transferred state or `ngSkipHydration`.
