# Chapter 52: Debugging Angular Applications

## 1. Overview

Debugging a production-grade Angular application is a different discipline from debugging a toy component. At staff/principal level you are expected to reason about three overlapping layers simultaneously:

1. **The rendered DOM and component tree** — what Angular *thinks* it rendered, versus what the browser shows.
2. **The reactive/data layer** — RxJS streams, signals, zone-triggered change detection, and the timing of asynchronous work.
3. **The compiled artifact** — AOT-generated `ɵɵdefineComponent` factories, minified bundles, and source maps that (sometimes) let you walk back to your original TypeScript.

Interviewers use debugging questions to separate people who have *memorized* Angular APIs from people who have actually chased a ghost bug through a minified production bundle at 2 a.m. This chapter covers the tooling (Angular DevTools, `ng.*` console globals, source maps, RxJS instrumentation) and — more importantly — the *systematic methodology* for narrowing down a bug when you have none of your usual comforts (no source maps, no verbose errors, a customer who says "it's broken" and nothing else).

The throughline of this chapter: **debugging is a search problem**. Your job is to shrink the search space as fast as possible using the cheapest signal available — console errors, then DevTools inspection, then targeted logging, then binary-search bisection of code paths, then, only as a last resort, stepping through minified code.

---

## 2. Core Concepts

### 2.1 Angular DevTools browser extension

Angular DevTools is a Chrome/Edge DevTools panel (there's also a standalone version) built on top of the same internal APIs that power `ng.*` console utilities. It has three main surfaces:

**Component tree inspector**
- Renders the live component hierarchy, including directives attached to a host element (shown as sub-rows under the component).
- Selecting a node lets you inspect/edit `@Input()` values, component properties, and injected dependencies live, and it highlights the corresponding DOM node.
- It also exposes signal values (in modern Angular) alongside plain properties, since signals aren't just fields on the instance in the way properties are.
- A node selected in DevTools is bound to `$ng0`, `$ng1`, etc. in the console — you can then call `ng.getComponent($ng0)` or access it directly as a JS object for ad hoc scripting.

**Profiler**
- Records change detection cycles as a timeline (flame-graph-like bars per CD pass), showing which components ran `ngDoCheck`/template checks and how long each took.
- Each bar in the timeline corresponds to a single **tick** of the Angular zone (or an OnPush check pass). Clicking a bar reveals a per-component breakdown of time spent, sorted so you can immediately spot the outlier component causing jank.
- You can toggle "record change detection only for components that changed" and see a directive-level list of what triggered a check versus what the check actually *did* (a component can be checked but not update the DOM if nothing changed — the profiler distinguishes "checked" from "changed" via color coding).
- Critical use case: diagnosing "my app CD's too much." If you see hundreds of components re-checked on a single mouse move, that's the profiler telling you zone.js is triggering global CD from an event that only one small subtree cares about — the fix is usually `ChangeDetectionStrategy.OnPush`, moving logic outside the zone (`NgZone.runOutsideAngular`), or migrating to signals-based `OnPush` end-to-end.

**Dependency Injection tab**
- Visualizes the injector tree (module injectors, standalone component injectors, element injectors) as a graph.
- For a selected node, shows every token resolvable from that injector, which parent injector actually provided it, and lets you jump to the provider's declaration.
- Invaluable for diagnosing "why did I get a different instance than I expected" (e.g., a service accidentally provided at component level shadowing the root singleton) or `NullInjectorError` triage — you can literally see the chain of injectors that was walked and where it terminated.

Angular DevTools requires the app to be running in **dev mode** by default (production builds strip the debug hooks it depends on, see §5). As of newer Angular versions, you can opt into a lightweight production-mode support by calling `enableProdMode()` alongside explicitly retaining Angular's debug APIs, but the default posture is: DevTools works out of the box only in dev builds.

### 2.2 `ng.getComponent()` / `ng.getContext()` and friends

Angular exposes a small set of debugging globals on `window.ng` when running in dev mode. These are the same primitives Angular DevTools itself uses under the hood.

- `ng.getComponent(element)` — given a DOM element, returns the component instance whose host is that element (or `null` if the element isn't a component host, e.g. it's inside a component but not the root).
- `ng.getContext(element)` — returns the full template context object for the element, which includes things like implicit `*ngFor` variables (`$implicit`, `index`, `first`, `last`) that aren't available via `getComponent`.
- `ng.getDirectives(element)` — returns an array of all directive instances (structural + attribute) attached to that element.
- `ng.getOwningComponent(element)` — walks up to find the nearest component that *declares* the element (useful for elements inside embedded views/`ng-template` output where `getComponent` returns `null`).
- `ng.getInjector(element)` — returns the element injector for that node, which you can `.get(SomeToken)` against to check what's actually resolvable there.
- `ng.applyChanges(component)` — forces Angular to re-run change detection for that component, useful for verifying a bound property "should" have caused a UI update.
- `ng.getListeners(element)` — lists event listeners Angular has attached to the element, including their target and whether they're synthetic (animation) listeners.

Practical workflow: right-click an element → **Inspect** in Chrome DevTools → in the Console, `$0` refers to the selected element → run `ng.getComponent($0)`. This is the single fastest way to answer "what component instance backs this DOM node, and what's its current state?" without adding a single line of code or a breakpoint.

### 2.3 Debugging change detection issues

Change detection bugs cluster into a small number of recurring shapes:

1. **"My OnPush component didn't update."** Root cause is almost always one of: (a) you mutated an object/array in place instead of creating a new reference, so Angular's reference-identity check on `@Input()` sees no change; (b) the update happened inside a callback that runs *outside* the Angular zone (a third-party library, a WebSocket `onmessage`, a raw `addEventListener` with `{ passive: true }` outside zone patching); or (c) the update happened, but on a component that isn't a descendant of the dirty path, so it was never checked. Diagnosis: use `ng.getComponent()` to inspect current value first (did the model actually change?), then check whether `ChangeDetectorRef.markForCheck()` needs to be called manually.
2. **"My app is checking way more than it should."** Every `(click)`, `setTimeout`, `Promise.then`, XHR/fetch callback that's monkey-patched by zone.js triggers a full application-root-down change detection tick by default. The Profiler tab is the right tool here — record a tick, look at how many components were "checked" versus how many actually "changed."
3. **`ExpressionChangedAfterItHasBeenCheckedError` (ExpressionChanged)** — a dev-mode-only guard where Angular runs a second CD pass in dev mode specifically to catch a value that changed *after* it was already displayed within the same cycle (e.g., in `ngAfterViewInit` you set a `@Input`-bound property, or a child mutates a parent's bound value during its own check). This never throws in production because the second verification pass is disabled — which means the underlying bug (state changing after render) can still cause visible one-frame flicker or stale UI in prod even without the error. Treat this error as a canary, not just an annoyance to suppress.
4. **Infinite/repeated CD loops** — a getter or method call in a template (`{{ getValue() }}`) that returns a new object/array reference on every call defeats memoization further down and can cause endless dirty-checking if paired with an `ngOnChanges` that mutates state.

The mental model to internalize: change detection walks the component tree from root, and for each node either does a full check (`Default` strategy) or skips unless one of the OnPush "wake" conditions occurred (input reference change, event fired within that subtree, `markForCheck()`/`ApplicationRef.tick()` called manually, or an `async` pipe emission). Debugging is the process of figuring out, for a given node, which of those conditions did or didn't fire.

### 2.4 Source maps and debugging AOT-compiled code

Angular always AOT-compiles for production builds. Template expressions and bindings become plain instantiations of instructions like `ɵɵproperty`, `ɵɵadvance`, `ɵɵtemplate` inside a generated `template()` function attached to a component's `ɵcmp` definition — there is no interpreter walking your `.html` file at runtime; your template *is* compiled to a JS function.

Source maps bridge four layers back to your original source:
- **TS → JS**: the TypeScript compiler emits a `.js.map` mapping compiled JS back to `.ts`.
- **Template → generated code**: the Angular compiler emits mapping information so that a runtime error thrown inside the generated `template()` function (e.g., a `TypeError: Cannot read property 'x' of undefined` from an interpolation) maps back to the specific expression in your `.html` file, not just the component class.
- **JS → minified/bundled JS**: the bundler (esbuild/webpack via the Angular CLI) emits a final source map composing all of the above, so a stack trace in a production bundle can, if the map is available, resolve all the way back to `app.component.ts:42` and even the originating template line.
- **`ng build --source-map`** (or the `sourceMap` option in `angular.json`) controls whether these are emitted at all; by default production builds omit maps for size/security reasons unless explicitly configured (`"sourceMap": true`, or `{"scripts": false, "styles": true, "vendor": true, "hidden": true}` for hidden source maps that are generated but not referenced from the bundle — uploaded to an error-tracking service instead).

**Hidden source maps** are the standard staff-level answer for "how do you get readable stack traces in production without exposing your source to end users": generate the map, do *not* emit a `//# sourceMappingURL=` comment in the shipped file, and upload the map privately to your error monitoring service (Sentry, Datadog, Bugsnag). The service resolves stack traces server-side using the private map; end users downloading the bundle never get a way to reconstruct your source through the browser's Sources panel.

### 2.5 Debugging RxJS streams

Angular is built on RxJS for the router, `HttpClient`, forms (`valueChanges`/`statusChanges`), and any custom reactive service state. Debugging a stream means answering: is it emitting, when, with what value, and did it complete/error?

**`tap`-based logging** is the workhorse tool — insert `tap()` at each pipeline stage to observe values flowing through without altering them:

```typescript
source$.pipe(
  tap(v => console.log('[source] emitted:', v)),
  filter(v => v.active),
  tap(v => console.log('[after filter]:', v)),
  switchMap(v => this.api.load(v.id).pipe(
    tap({
      subscribe: () => console.log('[api] subscribed'),
      next: r => console.log('[api] next:', r),
      error: e => console.log('[api] error:', e),
      complete: () => console.log('[api] complete'),
    })
  )),
).subscribe();
```

This immediately tells you whether the bug is upstream (source never emits, or the filter drops everything) or downstream (the inner observable from `switchMap` never resolves, or resolves with unexpected shape).

**RxJS DevTools** (browser extension backed by the `rxjs-spy`-style instrumentation, or the newer official tooling) attaches to observables tagged with an operator like `spyTag('user-search')` and visualizes subscription lifecycles, emitted values, and teardown timing in a dedicated panel — useful when `tap` chains get too noisy or when you need to see *all* active subscriptions app-wide rather than one pipeline at a time.

**Common RxJS bug categories:**
- **Subscription never fires** — cold observable never subscribed (forgot `.subscribe()`, or used it in a template without `async` pipe and without subscribing manually).
- **Stale closures in `switchMap`/`mergeMap`** — capturing a component property by reference inside the projection function that has since changed.
- **Memory leaks** — forgetting `takeUntil(this.destroy$)` / not using `takeUntilDestroyed()`, so subscriptions outlive the component and keep firing (and referencing) destroyed component instances — this can also cause "why does this callback still update a component I navigated away from" bugs.
- **Wrong flattening operator** — `mergeMap` when `switchMap` was needed causes race conditions where an older, slower response overwrites a newer one; diagnosing this requires timestamped `tap` logs showing "started A, started B, completed B, completed A" — the completion order, not the start order, actually explains the bug.
- **Multicast/cold-vs-hot confusion** — resubscribing to a cold HTTP observable causes duplicate requests, easily confirmed by a `tap({ subscribe: () => console.count('subscribed') })`.

### 2.6 Debugging production issues without source maps

When source maps are unavailable (rare intentionally, common accidentally — e.g., the CDN drops the `.map` file, or a customer's proxy strips them), your options are:

1. **Symbol/name-preserving builds** — ensure `ng build` isn't over-minifying identifiers you rely on for grep-ability; at minimum keep readable component/class names via `--named-chunks` or by inspecting webpack/esbuild stats.
2. **Error monitoring breadcrumbs** — Sentry/Datadog RUM breadcrumbs (route changes, HTTP calls, console logs, DOM clicks) reconstruct a timeline of user actions leading to the error even when the stack trace itself is opaque minified gibberish.
3. **Correlate minified stack frames to bundle chunk + line/col manually** — even without a `.map`, the *shape* of a stack trace (which chunk, which line, which column, repeated across many user sessions) is enough to bisect by re-deploying with source maps temporarily enabled behind a feature flag, or by diffing against a locally-rebuilt bundle from the exact same commit SHA to re-derive mapping offline.
4. **Reproduce locally against the exact deployed build** — pull the exact commit/tag that was deployed, run the same production build command, and reproduce with dev tools attached to that (unminified where possible, or with local source maps you now control).
5. **Feature-flag a "debug mode"** that turns on verbose logging / `console.table` dumps of internal state for affected users or in staging, without shipping full debug tooling to all users.
6. **Session replay tools** (LogRocket, FullStory, Sentry Replay) when you need to *see* what the user saw, because the DOM state at the time of the error is more informative than the stack trace itself for CSS/rendering bugs.

### 2.7 Common production bug categories

- **Timing/race bugs** — work in dev (fast local network, fast machine) but not in prod (slow network exposes race between two async operations). RxJS flattening-operator bugs (§2.5) are the classic example.
- **Environment/config drift** — `environment.prod.ts` pointing at wrong API base URL, feature flags mismatched, or CSP headers blocking something that worked in dev.
- **Zone.js interaction with third-party code** — a library that schedules its own `requestAnimationFrame`/timers outside Angular's knowledge, so UI updates silently don't trigger CD (looks identical to an OnPush bug but the root cause is "this code never entered the zone at all").
- **Bundle-splitting / lazy-loading failures** — a lazy chunk 404s after a new deploy because the user's browser tab is still running old `index.html` referencing chunk hashes that no longer exist on the CDN (classic "ChunkLoadError").
- **Memory leaks accumulating over a long-lived SPA session** — subscriptions, detached DOM listeners, or third-party widgets not cleaned up on route change, surfacing only after the user has navigated the app for 30+ minutes — never visible in a quick dev-mode check.
- **Locale/timezone/Intl differences** between the CI/dev machine and real user machines, causing date-formatting-dependent bugs invisible until deployed globally.
- **CSS/rendering issues specific to real device/DPI/OS font rendering** — not reproducible on your dev machine at all; requires BrowserStack/real-device debugging or session replay.

### 2.8 Systematic debugging methodology

A methodology worth stating explicitly, since interviewers are grading *process* as much as tool knowledge:

1. **Reproduce** — get a reliable repro, even a flaky one ("happens ~1 in 10 times after clicking X twice quickly"). No repro, no fix, only guesses.
2. **Characterize the symptom precisely** — "broken" is not a bug report. Is it a thrown error (check the stack), a stale/incorrect value (check the data layer), a missing render (check CD), or a timing issue (check ordering across async boundaries)?
3. **Bisect** — binary-search across commits (`git bisect`), across environments (dev vs. staging vs. prod), across code paths (comment out half the pipeline), or across data (does it happen with all records or one malformed one)?
4. **Instrument at the boundary, not everywhere** — add `tap()`/`console.log` at the *narrowest* point where you can still distinguish "bug is before this point" from "bug is after this point," then move the instrumentation inward with each iteration (this is the RxJS/data-layer analogue of a debugger breakpoint bisection).
5. **Form a hypothesis before you look at more logs** — write down what you expect to see *if* your theory is correct, then check. This prevents "log everything and stare at it" thrashing.
6. **Fix at the right layer** — a stale OnPush view can be "fixed" with `markForCheck()` sprinkled everywhere, but the correct fix is usually upstream (immutable update pattern, or `NgZone.run()` around the offending callback). Prefer root-cause fixes over local patches that just mask the symptom.
7. **Add a regression test or an assertion/log that would have caught this earlier** next time, closing the loop.

---

## 3. Code Examples

### 3.1 `tap`-based RxJS debug logging pattern

```typescript
import { Observable, of, throwError } from 'rxjs';
import { catchError, delay, switchMap, tap } from 'rxjs/operators';

interface SearchResult {
  id: string;
  name: string;
}

/**
 * A reusable tap-based debug operator: logs subscribe/next/error/complete
 * with a tag and timestamp, without altering the stream's values.
 * Drop this in anywhere in a pipeline to "see" that stage of the pipeline.
 */
function debugTap<T>(tag: string) {
  return tap<T>({
    subscribe: () => console.log(`%c[${tag}] subscribe`, 'color:#888', performance.now().toFixed(1)),
    next: (value) => console.log(`%c[${tag}] next`, 'color:#2a2', value),
    error: (err) => console.error(`%c[${tag}] error`, 'color:#c33', err),
    complete: () => console.log(`%c[${tag}] complete`, 'color:#888', performance.now().toFixed(1)),
    unsubscribe: () => console.log(`%c[${tag}] unsubscribe/teardown`, 'color:#888'),
  });
}

class SearchService {
  search(term: string): Observable<SearchResult[]> {
    // simulated backend call
    return of([{ id: '1', name: term }]).pipe(delay(300));
  }
}

class SearchComponent {
  private searchService = new SearchService();

  wireUpSearch(term$: Observable<string>) {
    term$
      .pipe(
        debugTap('term$'),
        switchMap((term) =>
          this.searchService.search(term).pipe(
            debugTap('search$ (inner)'),
            catchError((err) => {
              console.error('search failed, falling back to empty result', err);
              return of([] as SearchResult[]);
            }),
          ),
        ),
        debugTap('final-results$'),
      )
      .subscribe((results) => {
        // If final-results$ never logs "next", the bug is upstream.
        // If term$ logs but search$ (inner) never subscribes, switchMap's
        // projection function is throwing synchronously — check that next.
        console.log('rendering results:', results);
      });
  }
}
```

Why this pattern matters in an interview answer: the `subscribe`/`next`/`error`/`complete`/`unsubscribe` observer object (rather than a single `tap(fn)` for `next` only) is what lets you catch teardown-related bugs (subscription torn down before a value arrived — a classic `switchMap` cancellation surprise) and silent synchronous throws inside `switchMap`'s projection, which otherwise look identical to "just never emitted."

### 3.2 Using `ng.getComponent()` in the browser console

```typescript
// --- app-debug.util.ts ---
// Not shipped to production; imported only in dev, or run ad hoc in DevTools console.

// Step 1: In Chrome DevTools Elements panel, click the DOM node for the
// component you want to inspect. Chrome stores it as `$0`.

// Step 2: In the Console:
//   const cmp = ng.getComponent($0);
//   console.log(cmp);
// -> returns the live component instance. Mutating properties on `cmp`
//    directly does NOT trigger change detection by itself.

// Step 3: force a CD pass to see if the UI reflects a manual change:
//   cmp.someInput = 'new value';
//   ng.applyChanges(cmp);

// Step 4: inspect the template context (e.g., for a node inside *ngFor)
//   const ctx = ng.getContext($0);
//   console.log(ctx.$implicit, ctx.index, ctx.first, ctx.last);

// Step 5: inspect the injector actually backing that element, to debug
// "why did I get the wrong instance of this service":
//   const injector = ng.getInjector($0);
//   injector.get(SomeService) === expectedSingleton; // true/false tells you
//   whether a shadowing provider exists closer to this element.

// Step 6: list every directive attached to the host element (structural +
// attribute directives), useful when multiple directives compete for the
// same DOM node and you're not sure which is active:
//   ng.getDirectives($0);

// A small helper you can paste into the console repeatedly while iterating:
function inspect(el: Element) {
  return {
    component: (window as any).ng.getComponent(el),
    context: (window as any).ng.getContext(el),
    directives: (window as any).ng.getDirectives(el),
    injector: (window as any).ng.getInjector(el),
  };
}
// usage: inspect($0)
```

### 3.3 A debug-only directive that logs change detection cycles

```typescript
import { Directive, DoCheck, Input, OnChanges, SimpleChanges } from '@angular/core';

/**
 * Attach as `<my-component cdLog="MyComponent">` to log every time Angular
 * runs change detection against the host, and whether ngOnChanges fired
 * with an actual diff. Strip via a build-time flag/tree-shaking in prod
 * (e.g. behind `if (!environment.production)` or a separate dev-only module).
 */
@Directive({
  selector: '[cdLog]',
  standalone: true,
})
export class ChangeDetectionLogDirective implements DoCheck, OnChanges {
  @Input('cdLog') label = 'unnamed';

  private checkCount = 0;

  ngOnChanges(changes: SimpleChanges): void {
    const changed = Object.keys(changes)
      .map((key) => `${key}: ${JSON.stringify(changes[key].previousValue)} -> ${JSON.stringify(changes[key].currentValue)}`)
      .join(', ');
    console.log(`%c[cdLog:${this.label}] ngOnChanges`, 'color:#06c', changed || '(no bound inputs changed)');
  }

  ngDoCheck(): void {
    this.checkCount++;
    // DoCheck fires on EVERY change detection pass reaching this node,
    // regardless of whether any binding actually changed — this is the
    // signal to watch when diagnosing "checked too often."
    console.log(`%c[cdLog:${this.label}] ngDoCheck #${this.checkCount}`, 'color:#999');
  }
}
```

```typescript
// Usage in a template, temporarily, while chasing a CD bug:
// <app-product-card [product]="p" cdLog="ProductCard-{{ p.id }}"></app-product-card>
//
// Reading the console output tells you immediately:
// - ngDoCheck firing far more often than ngOnChanges -> this component is
//   Default strategy and being swept up in unrelated global CD ticks;
//   candidate for OnPush.
// - ngOnChanges firing with a changed object reference but the DOM not
//   visibly updating -> look at *ngIf/*ngFor trackBy or a stale template
//   reference, not change detection itself.
```

---

## 4. Internal Working

### 4.1 How Angular DevTools reads the internal `LView` tree

Angular's runtime (Ivy) represents each component/embedded view as an `LView` — a plain JS array holding, at fixed offsets, the view's DOM nodes, child component instances, the `TView` (shared static template metadata compiled once per component type), context, injector, and bookkeeping flags (dirty bit, `CheckAlways` vs `OnPush` flag, etc.). The whole application is a tree of `LView`s linked via parent/child pointers (`LContainer`s hold embedded views for structural directives like `*ngFor`/`*ngIf`).

Angular DevTools does **not** parse your compiled JS or re-implement Angular's renderer. Instead:
1. It injects a content script into the page that talks to a small runtime "backend" bundled with your app (or attached lazily) which calls the same internal instrumentation APIs that back `ng.getComponent`/`ng.getDirectives`/etc. — specifically APIs that walk from a DOM node back to its owning `LView` via `getLContext()`(an internal function that reads a hidden marker Angular stamps onto each root DOM node’s element, `__ngContext__`, pointing at the `LView` and node index).
2. From an `LView`, it walks `TView.data` and the `LView` array in lockstep to reconstruct the list of directives/components on each node, their instances, and metadata like input/output property names (which Ivy retains as compiled metadata on the component definition, `ɵcmp.inputs`/`ɵcmp.outputs`, even in profiles builds, unless explicitly stripped).
3. Child `LView`s/`LContainer`s are discovered by following the `LView`'s child pointers, which is how the extension reconstructs the full component tree rather than just a flat list.
4. For the **profiler**, DevTools hooks into `ApplicationRef.tick()`/the internal change-detection scheduler and instruments each `refreshView()`/`checkView()` call (the functions that actually walk a `TView`'s instructions to update bindings), timing them and recording whether the view was marked dirty going in versus whether it produced DOM mutations — that's the "checked" vs "changed" distinction shown in the flame graph.
5. The **DI tab** walks the injector hierarchy by following each `LView`'s stored injector reference up through its `NodeInjector` chain (element injectors), then up through the module/environment injector chain, reading each injector's provider records (which are retained in dev-mode/DevTools-mode builds as structured data, not just closures) to render the graph.

This is why DevTools requires cooperation from the running app build: the debug APIs (`getLContext`, retained input/output names, retained provider records) are compiled in for dev builds and are either stripped or made non-authoritative in a stock production build (see §5.1).

### 4.2 How `ng.probe`/global utils expose internal component instances

Historically (View Engine, pre-Ivy) Angular exposed `ng.probe(element)` returning a `DebugElement` wrapper. Ivy replaced this with the current `ng.getComponent`/`ng.getContext`/etc. family, but the underlying mechanism is conceptually the same:

- When running with debug APIs enabled, Angular's renderer attaches a hidden property (`__ngContext__`) to each DOM node it creates, whose value is either the index into the owning `LView` (for a plain node) or, for a component host element, a reference that lets the runtime recover the exact `LView` and the node's position within it.
- `ng.getComponent(element)` reads `element.__ngContext__`, uses it to locate the `LView`, looks up the `TView.data` entry at that node index to confirm it's a component host, and returns `lView[nodeIndex]` — which is precisely the component instance Angular stores in the view array at that slot.
- `ng.getContext(element)` instead returns the *view's context* — for a component that's the component instance itself; for an embedded view created by a structural directive (`*ngFor`), the context is the object holding `$implicit`, `index`, `first`, `last`, etc. that the structural directive populated when creating the view.
- These globals are registered onto `window.ng` by a call the framework makes during bootstrap (`publishDefaultGlobalUtils()` internally), guarded by whether the app was built/configured to include them — this registration is exactly what's skipped/no-op'd when Angular determines it's in a "don't expose debug internals" mode.

### 4.3 How source maps map back to TS from compiled JS

A source map is a JSON document (`.js.map`) with, most importantly, a `mappings` field: a compact VLQ (variable-length quantity) base64-encoded string where each segment encodes, relative to the previous segment, "generated column → (source file index, original line, original column, name index)". Browsers/tools decode this incrementally as they need to translate a generated position back to source.

For Angular specifically there are two compilation stages that must each preserve/compose mappings:
1. **Template compilation**: the Angular compiler (`@angular/compiler-cli`) turns your `.html` template (plus inline template strings) into instruction calls inside a generated `template()` function. It emits mapping segments so that, e.g., the `ɵɵproperty('value', ctx.user.name)` call generated from `[value]="user.name"` maps back to that exact attribute in your `.html` file — this is why a runtime error thrown while evaluating a binding can, with source maps enabled, point you directly at the offending template expression instead of just "somewhere in app.component.ts".
2. **TypeScript → JS compilation**: `tsc`'s ordinary source-map emission maps the generated JS for your class/methods back to your `.ts` source lines.
3. **Bundling/minification**: esbuild or webpack (the CLI's build backends) *compose* the two prior maps into one final map, additionally accounting for the code motion introduced by minification (renaming, dead-code elimination, concatenation across files) so a single mapping ultimately chains: `minified-bundle.js:1:48291` → `app.component.js:42:10` → `app.component.ts:42:10` (or straight through to `app.component.html:15:8` for template-originated code) in one hop, since map composition flattens the chain rather than requiring you to manually walk each stage.

Devtools (Chrome Sources panel) reads the `//# sourceMappingURL=` comment (or, for hidden maps, a manually configured association) at the bottom of the JS file, fetches the referenced `.map`, and uses it to let you set breakpoints in — and see stack traces resolved to — your original `.ts`/`.html` even though the browser is executing the fully bundled/minified JS.

---

## 5. Edge Cases & Gotchas

### 5.1 Production builds stripping `ng.*` globals unless enabled

By default, `ng build` (production configuration) does **not** register the `ng.*` debugging globals on `window`, and Angular DevTools will show a "not detected"/limited state against such a page. This is deliberate: exposing full component-tree introspection and property-level read/write in production is both a bundle-size cost and a minor information-disclosure/tampering surface. Concretely:
- `isDevMode()` / whether `enableProdMode()` was called historically gated this; in newer Angular the relevant knob is whether the app was built with default production optimizations (which omit the calls to `publishDefaultGlobalUtils()`/`publishGlobalUtil()` that wire up `window.ng`).
- You *can* deliberately re-enable them in a production build if you need DevTools support in a staging/production-like environment (e.g., calling the appropriate provider/bootstrap option that keeps debug utilities available), but this is an explicit opt-in you must make and ship consciously — not something you get "for free" just because the code otherwise works.
- If DevTools shows nothing at all, the very first checks are: (1) is this actually an Angular app (`ng-version` attribute present?), (2) is it a production build with debug globals stripped, (3) is the extension itself just not injected yet (SPA loaded via unusual bootstrapping, iframe boundaries DevTools doesn't cross by default).

### 5.2 Minified property names complicating heap snapshots

When you take a heap snapshot (Chrome Memory panel) of a production build to chase a leak, class and property names are typically minified/mangled (`a`, `b`, `_c`) unless you've configured the minifier to preserve class names (`keep_classnames`/similar options) or you're inspecting a build with named-but-minified output. This turns "find all retained `SearchService` instances" into "find all retained instances of some class named `t`," which is far harder to do by name search alone. Mitigations:
- Use the **retainers graph** (what's holding a reference) rather than searching by class name — object *shape* (its own property names, which often survive minification differently than class names depending on config) and its position in the retainer chain are more reliable than the mangled constructor name.
- Take a snapshot from a *locally rebuilt, unminified* (or `optimization: false`) version of the exact same commit when you need name-searchability, then correlate object counts/shapes back to the minified prod snapshot.
- Angular component instances specifically retain a predictable *shape* — look for objects with `__ngContext__` properties still present (they usually are, even in minified prod builds, since Ivy needs them at runtime) as an anchor to identify "this is some Angular component instance" before further narrowing by shape.
- Comparison snapshots (snapshot A before an action, snapshot B after, diff the "objects allocated between A and B and not yet freed") are far more effective than single-snapshot browsing regardless of minification, since you're diffing counts/shapes rather than reading names.

### 5.3 Debugging OnPush components not updating

This is the single most common "gotcha" interview scenario, and it has a specific decision tree, not one root cause:

1. **Check whether the bound `@Input()` reference actually changed.** `ng.getComponent(el).someInput` — compare object identity, not just apparent value, to what you expect. If you mutated a nested field on the same array/object reference, OnPush's reference check sees nothing new — this is the #1 cause.
2. **Check whether the mutation happened inside the Angular zone at all.** If the state update came from a raw DOM API, a third-party widget's callback, or a WebSocket handler that isn't zone-patched (`Zone.js` zone-patches vary by API and configuration, and modern Angular increasingly runs in "zoneless" or zone-avoidance modes), then no CD tick was ever scheduled — you can confirm by wrapping the callback in `NgZone.run(() => ...)` as a test and seeing whether the UI now updates; if so, the fix is either running that code inside the zone or manually calling `ChangeDetectorRef.markForCheck()`/`ApplicationRef.tick()`.
3. **Check whether the component is actually a descendant of the CD path that ran.** An OnPush component detached via `ChangeDetectorRef.detach()` (common in custom virtual-scroll or performance-tuned components) will never be checked again until `reattach()` or a manual `detectChanges()` — a very easy thing to forget you did three files away.
4. **Check `async` pipe usage vs manual subscription.** The `async` pipe calls `markForCheck()` internally on each emission — a manual `.subscribe()` that sets a component field does *not*, so with OnPush the component won't refresh even though the field is genuinely updated, unless you call `markForCheck()` yourself in the subscribe callback.
5. **Signals change this calculus**: a component reading a `signal()` directly in its template gets fine-grained reactivity that doesn't depend on OnPush's input-reference check at all — if you're debugging a "mixed" component using both classic `@Input`/mutable-state patterns and signals, make sure you're not misattributing a signal-driven update path to the OnPush input-check path (or vice versa) when reasoning about why something did or didn't refresh.

---

## 6. Interview Questions & Answers

**Q1. What's the fastest way to find the component instance backing a specific DOM element, without adding any code?**
Right-click the element → Inspect (so it becomes `$0` in the console), then run `ng.getComponent($0)`. This works in dev builds where Angular's debug globals are registered. For the template context of an embedded view (e.g., `*ngFor` row), use `ng.getContext($0)` instead, since `getComponent` returns `null` for nodes that aren't component hosts.

**Q2. Why does Angular DevTools sometimes show nothing / say it can't detect Angular on a production site?**
**Interviewer intent:** checks whether the candidate understands that DevTools depends on runtime cooperation, not magic external inspection.
Production builds by default don't register the `window.ng` debug globals (`getComponent`, `getContext`, etc.) that both DevTools and manual console debugging rely on — this is deliberate, to avoid the bundle-size and introspection/tamper surface cost in production. Angular DevTools reads the component tree via these same underlying APIs (walking `LView`s via a node's `__ngContext__` marker), so if they aren't registered, there's nothing for the extension to hook into. You either debug in a dev build, or you deliberately opt into keeping debug utilities available in that particular build/environment.

**Q3. A component using `ChangeDetectionStrategy.OnPush` isn't updating when you know the underlying data changed. Walk through your debugging steps.**
First, confirm the *reference* actually changed — inspect the current bound input via `ng.getComponent(el)` and compare identity, not just value, since in-place mutation of an object/array is the single most common cause and reads as "changed" logically but not by reference. Second, check whether the state change happened inside the Angular zone — updates from raw DOM listeners, some third-party libraries, or manually-scheduled work outside `NgZone.run()` never trigger a CD tick. Third, check whether the component was detached (`ChangeDetectorRef.detach()`) elsewhere in the codebase. Fourth, if the value arrives via a manually-subscribed Observable rather than the `async` pipe, remember the `async` pipe calls `markForCheck()` on your behalf — a manual subscription doesn't, so you need to call it yourself.

**Q4. What's the difference between `ng.getComponent()` and `ng.getContext()`?**
`getComponent(element)` returns the *component instance* whose host is that element, or `null` if the element isn't a component host. `getContext(element)` returns the *template context* for the view containing that element — for a component's own view that's effectively the component instance again, but for an embedded view created by a structural directive (`*ngFor`, `*ngIf` with an `else` template, custom `*ngTemplateOutlet` usage), it returns the context object holding things like `$implicit`, `index`, `first`, `last` that aren't accessible from `getComponent` at all, since those nodes aren't component hosts.

**Q5. Explain `ExpressionChangedAfterItHasBeenCheckedError`. Is it a real bug or a false alarm?**
**Interviewer intent:** tests whether the candidate treats this as noise to suppress versus a genuine correctness signal about production behavior.
It's Angular's dev-mode-only second verification pass catching a bound value that changed *after* it was already rendered within the same change-detection cycle — commonly caused by setting a parent-bound value from a child's lifecycle hook (e.g., `ngAfterViewInit`) or a getter/computed value that isn't stable across two consecutive reads. It never throws in production because that verification pass doesn't run there — but the underlying issue (state still settling after the "final" render of a cycle) is real and can cause one-frame flicker or a subtly stale UI in production even without any visible error. Treat it as a canary pointing at real timing bugs, not just something to silence.

**Q6. How do source maps let you debug a minified production bundle, and what are hidden source maps for?**
A source map encodes, via VLQ-compressed segments, a mapping from each position in the generated (minified/bundled) file back to a file/line/column in the original source. For Angular this composes across three stages — template compilation (mapping generated instruction calls back to `.html` bindings), TypeScript compilation (mapping JS back to `.ts`), and bundling/minification (composing the prior two and accounting for renaming/code motion) — into one final map so a browser or error-tracking tool can resolve a minified stack frame straight back to your original source line, sometimes down to the exact template expression. Hidden source maps are generated (so your error-tracking service can use them) but not referenced via a `//# sourceMappingURL=` comment in the shipped bundle, so end users can't pull your original source through the browser's Sources panel — you upload the map privately to something like Sentry, which resolves stack traces server-side.

**Q7. You suspect a memory leak from RxJS subscriptions that outlive their component. How would you find and fix it?**
Take a heap snapshot before and after repeatedly navigating to and away from the suspect component/route, and diff retained object counts — growing counts of the component's class shape (or, in a minified build, matching by retained-property shape/`__ngContext__` presence rather than mangled class name) that should have been garbage collected indicates a leak. Confirm via the retainers graph what's holding the reference — typically an Observable/Subject the component subscribed to without teardown. Fix by using `takeUntilDestroyed()` (or the classic `takeUntil(this.destroy$)` + `ngOnDestroy` pattern) on every subscription that isn't automatically torn down (the `async` pipe *does* auto-unsubscribe; manual `.subscribe()` calls don't).

**Q8. What does the Angular DevTools Profiler distinguish between "checked" and "changed," and why does that distinction matter?**
**Interviewer intent:** probes whether the candidate actually understands what change detection *does* per node versus just knowing "OnPush skips checks."
"Checked" means Angular's change-detection walk visited that component/view and ran its template-check function (evaluating bindings). "Changed" means that check actually produced a DOM mutation because a binding's value differed from last time. A component can be checked (walked, bindings evaluated) without changing (nothing was different, so no DOM write happened). This distinction matters because a performance problem is usually "too many components checked," which is fixable with OnPush/zone-avoidance, whereas "many components changed" might just reflect genuinely large, legitimate UI updates that CD is correctly performing — the profiler lets you tell those two situations apart instead of just seeing "CD is slow" as an undifferentiated blob.

**Q9. Your app has a race condition: a slower earlier request's response overwrites a faster later request's response. How do you diagnose this with RxJS-focused tooling, and what's the fix?**
Add `tap` observers with timestamps at the point of dispatch and at the point of the inner observable's `next`/`complete` for each request, so you can see the actual completion order rather than assuming start order equals completion order — the logs will show "started A, started B, completed B, completed A," proving that A's late response is clobbering B's already-rendered result. The fix is almost always to swap the flattening operator: `mergeMap` (concurrent, no cancellation) is often mistakenly used where `switchMap` (cancels the previous inner observable when a new source value arrives) is what's actually wanted for "latest search wins" semantics; for cases where you need all responses but must apply them in request order, `concatMap` (sequential) or explicit sequencing/indexing is the fix instead.

**Q10. How does Angular DevTools' Dependency Injection tab actually determine which provider satisfies a token for a given component, and why is that hard to answer by reading code alone?**
**Interviewer intent:** distinguishes candidates who understand the hierarchical injector resolution algorithm from those who only know `providedIn: 'root'` by rote.
It walks the actual runtime injector chain starting from the selected node's element injector (`NodeInjector`, populated from that element's and its ancestors' `providers`/`viewProviders` arrays), and if not found there, continues up through the module/environment injector hierarchy (lazy-loaded module injectors, the platform injector, `root`). It reports which injector in that walked chain actually held a matching provider record. This is hard to determine by static code reading alone because Angular's DI resolution is a *runtime* hierarchical lookup — the same token can be legitimately provided at multiple levels (component, module, root) simultaneously, and which one "wins" for a specific component instance depends on that component's actual position in the live injector tree, which especially with lazy-loaded feature modules or per-route providers, isn't always obvious just from reading each file in isolation.

**Q11. You're debugging a production issue and have no source maps available at all. What's your approach?**
Prioritize signal sources that don't require source maps: error-monitoring breadcrumbs (route changes, HTTP calls, prior console output, user clicks) to reconstruct the sequence of events leading to the failure; the *shape* of the minified stack trace (same chunk/line/column recurring across many affected sessions) to at least group/bucket occurrences even without readable names; and reproducing locally against the exact deployed commit/build so you have a debuggable copy under your control. If it's a systemic problem worth deeper investigation, temporarily (and safely — behind a flag, in staging, or for a sampled percentage of users) re-enable source maps or ship a debug build to reproduce with full tooling, rather than trying to permanently reverse-engineer minified code by hand. Session replay tools are especially valuable here for rendering/CSS bugs where the stack trace wouldn't help even with a map.

**Q12. Why might `console.log`-based debugging of an Observable mislead you about *when* something actually happened?**
**Interviewer intent:** checks for awareness of microtask/macrotask timing and multicast subtleties, not just "logging works."
A plain `console.log` inside `subscribe()`/`tap()` tells you when *that particular callback* ran, but it doesn't tell you: whether the observable is cold (each subscriber triggers independent execution/side effects — so two logs from two subscribers can mean two separate underlying HTTP calls, not two observers of one call) or hot/multicast; whether you're seeing synchronous emission during subscription setup versus a later asynchronous emission (easy to misread ordering when several `of()`/synchronous sources are interleaved with genuinely async ones); or whether zone.js's task tracking has already scheduled a change-detection tick relative to when your log fired, which matters when you're trying to correlate "value logged" with "UI actually updated." Using the observer-object form of `tap` (`subscribe`/`next`/`error`/`complete`/`unsubscribe`) and logging `performance.now()` alongside each event, rather than a single bare `next` log, avoids most of these misreadings.

**Q13. How would you use `ng.getInjector()` to debug a `NullInjectorError` or an unexpectedly-wrong service instance?**
Select the failing element (or its nearest component ancestor) and run `ng.getInjector(el)`, then call `.get(TheToken)` against it to see directly what that specific injector resolves — if it throws `NullInjectorError` there too, you've confirmed the token genuinely isn't providable from that position in the tree (versus a typing/import issue elsewhere), and you can walk up manually via `injector.get(Injector).parent` (or use the DevTools DI tab's visualization instead of doing this by hand) to find the first ancestor where it *does* resolve, and compare that against where you expected it to be provided (e.g., discovering a component-level `providers: [SomeService]` you forgot about is shadowing the intended root singleton, so every component under it gets its own instance instead of sharing state).

---

## 7. Quick Revision Cheat Sheet

- **Angular DevTools panels**: Component tree (inspect props/signals, jump to DOM), Profiler (checked vs. changed, per-tick timing), DI tab (visualize injector resolution chain).
- **Console globals** (dev builds only): `ng.getComponent(el)`, `ng.getContext(el)`, `ng.getDirectives(el)`, `ng.getOwningComponent(el)`, `ng.getInjector(el)`, `ng.getListeners(el)`, `ng.applyChanges(cmp)`.
- **OnPush not updating — check in order**: (1) input reference actually changed? (2) update happened inside the zone? (3) component detached via `ChangeDetectorRef.detach()`? (4) manual subscribe missing `markForCheck()` (the `async` pipe does this for you)?
- **`ExpressionChangedAfterItHasBeenCheckedError`**: dev-only canary for "value changed after render within one cycle" — real timing bug, not just noise; doesn't throw in prod but can still cause flicker/staleness there.
- **Source maps**: compose template→JS, TS→JS, and bundler/minifier mappings into one chain; `sourceMap` build option controls emission; **hidden source maps** = generated but not referenced from the shipped file, uploaded privately to error tracking.
- **RxJS debugging**: `tap` with the full observer object (`subscribe`/`next`/`error`/`complete`/`unsubscribe`) beats a bare `next`-only log; timestamp logs to catch completion-order races; `mergeMap` vs `switchMap` vs `concatMap` mix-ups are the #1 race-condition source.
- **No source maps in prod**: lean on error-monitoring breadcrumbs, minified-stack-trace bucketing by chunk/line/col, exact-commit local rebuilds, session replay, and temporary flagged/staged debug builds — don't hand-reverse-engineer minified code as a first resort.
- **Common prod bug categories**: async race conditions, environment/config drift, zone.js not covering third-party callbacks, chunk-loading failures after redeploy, long-session memory leaks, locale/timezone drift, device-specific rendering issues.
- **Debugging methodology**: reproduce → characterize precisely → bisect (commits/environments/code paths/data) → instrument at the narrowest boundary → hypothesize before logging more → fix at the root-cause layer → add a regression guard.
- **Internals**: `LView` = the per-view state array; `__ngContext__` on a DOM node is the hook `ng.getComponent`/DevTools use to recover the owning `LView`/`TView`; production builds by default skip registering these debug globals (`publishDefaultGlobalUtils`) for size/security reasons — DevTools needs them present to function.
