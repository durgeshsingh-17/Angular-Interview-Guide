# Chapter 32: Server-Side Rendering (SSR)

## 1. Overview

By default, an Angular application is a **client-side rendered (CSR)** single-page application (SPA). The server ships an almost-empty `index.html` plus a large JavaScript bundle; the browser downloads the bundle, boots Angular, and only then paints real content. Until that happens, the user sees a blank page (or a static loading shell), and any crawler that doesn't execute JavaScript sees nothing meaningful.

**Server-Side Rendering (SSR)** flips part of this process: Angular runs the same application on a Node.js server, renders the component tree to a fully-formed HTML string for the *specific requested URL*, and sends that HTML to the browser immediately. The browser can paint real content before a single line of application JavaScript has executed. Once the JS bundle arrives, Angular "boots" on the client and takes over the already-rendered DOM (historically by re-rendering it; in modern Angular, by **hydrating** it — reusing the existing DOM nodes rather than tearing them down).

Angular's SSR implementation is historically called **Angular Universal** (the project that made "render once, run anywhere — server, browser, mobile" possible). As of Angular 17+, Universal's capabilities have been folded directly into the core Angular CLI and framework — `ng add @angular/ssr` (or the `--ssr` flag at `ng new` time) scaffolds everything, and SSR is now a first-class, non-experimental part of the framework rather than a bolted-on package.

### Why SSR matters (the three pillars)

1. **SEO** — Search engine crawlers (and social media link-preview bots) historically had inconsistent JavaScript execution support. Even now that Googlebot can execute JS, it does so in a secondary, delayed rendering pass, and other crawlers (Bing, LinkedIn, Twitter/X, Facebook's scraper) often don't execute JS at all. SSR guarantees a fully populated HTML document is available on the very first response, with all your `<title>`, `<meta>`, and content tags present, which is the safest and fastest path to correct indexing and rich link previews.

2. **Perceived performance (First Contentful Paint)** — SSR dramatically improves **First Contentful Paint (FCP)** and **Largest Contentful Paint (LCP)**, because the browser can paint server-rendered markup while the JS bundle is still downloading/parsing/executing in the background. The user *sees* the page much sooner, even though the page isn't interactive yet.

3. **Core Web Vitals** — Google's Core Web Vitals (LCP, INP/FID, CLS) directly factor into search ranking. SSR helps LCP a lot (content paints early) and can help CLS (server-rendered layout is stable before client JS starts moving things around), though it does nothing by itself for **TTI (Time to Interactive)** — the page may look ready but not yet respond to clicks until hydration completes. This gap between "looks done" and "is done" is the central SSR trade-off, and it's exactly what Angular's hydration feature is designed to shrink.

## 2. Core Concepts

### 2.1 Angular Universal architecture

"Universal" describes an app that can run in multiple JavaScript environments: the browser (with a real DOM) and Node.js (with no DOM at all). Angular achieves this via the **platform abstraction**:

- **`platform-browser`** — the default platform, bootstraps the app against a real browser `window`/`document`.
- **`platform-server`** — bootstraps the *same* application code inside Node.js, using a lightweight, headless DOM implementation (Domino, historically) to give `document`, `Node`, `Element`, etc. enough of an implementation that component templates can be rendered to a string. No real browser APIs (canvas, real layout, `IntersectionObserver`, etc.) exist here.

Both platforms sit on top of the platform-agnostic Angular core (change detection, DI, template compiler output). Your component/service/module code is written once; only the **bootstrap entry point** differs between browser and server. This is the essence of "Universal" — one codebase, two runtimes.

Structurally a modern (standalone-API) Angular SSR app has:

- `src/main.ts` — client bootstrap (`bootstrapApplication(AppComponent, appConfig)`), unchanged from a CSR app.
- `src/main.server.ts` — server bootstrap, using `bootstrapApplication(AppComponent, mergeApplicationConfig(appConfig, serverConfig))` referencing a server-specific config.
- `src/app/app.config.server.ts` — calls `provideServerRendering()` and any server-only providers.
- `src/server.ts` — an Express (or other Node HTTP) app that, for each incoming request, invokes the Angular server renderer (via `@angular/ssr`'s `AngularNodeAppEngine` / `CommonEngine`) to produce HTML and writes it to the HTTP response.

### 2.2 `provideServerRendering()` and the `platform-server` package

`provideServerRendering()` (from `@angular/platform-server`, or `@angular/ssr` in newer CLI-scaffolded apps) is the provider function that wires up everything needed for server rendering into your application's DI tree: the server-side renderer, DOM shim bindings, and (in modern Angular) **hydration support** so the client knows how to reconcile with server-rendered DOM instead of blowing it away.

It is added only to the *server* application config, never the browser one:

```typescript
// app.config.server.ts
import { mergeApplicationConfig, ApplicationConfig } from '@angular/core';
import { provideServerRendering } from '@angular/platform-server';
import { appConfig } from './app.config';

const serverConfig: ApplicationConfig = {
  providers: [
    provideServerRendering(),
  ],
};

export const config = mergeApplicationConfig(appConfig, serverConfig);
```

The `platform-server` package itself provides:
- `renderApplication()` / `renderModule()` — programmatic APIs that take a bootstrap function/module and a document string, and return a Promise of the rendered HTML.
- A DOM shim (Domino) so `document.createElement`, `querySelector`, etc. work without a browser.
- Server-safe implementations/no-ops for browser-only concerns (e.g., style/script tag insertion tracking so styles get correctly emitted into the rendered HTML `<head>`).

### 2.3 TransferState — avoiding duplicate data fetching

Without special handling, SSR causes **double data fetching**: the server renders the component tree (which triggers an HTTP call to your API to get, say, a product list), and then when the client boots and re-renders/hydrates, the *same* component fires the *same* HTTP call again — wasting an API round-trip the user's browser didn't need, and potentially causing a content flash if the second response differs even slightly.

`TransferState` solves this: it's a key/value store that gets **serialized into the server-rendered HTML** (as a `<script id="ng-state" type="application/json">` blob) and **rehydrated on the client** before bootstrapping. The pattern:

1. On the server, after fetching data, store it in `TransferState` under a `StateKey`.
2. On the client, before making the HTTP call, check `TransferState` for that key. If present, use it and skip the HTTP call (and remove/clear the key so subsequent navigations still fetch fresh data). If absent (pure CSR run, no SSR), fall back to the normal HTTP call.

Angular's `HttpClient` actually has this built in automatically via `withHttpTransferStateCache()` (provided by `provideClientHydration(withHttpTransferStateCache())` or as part of `provideHttpClient(withFetch())` transfer-state interop) for apps using `HttpClient` — but understanding the manual `TransferState` API is essential for interviews and for non-`HttpClient` data sources (e.g., a raw `fetch`, a GraphQL client, or a third-party SDK call).

### 2.4 `isPlatformBrowser` / `isPlatformServer`

Because server code has no `window`, `document`, `localStorage`, `navigator`, etc., any code that touches these directly will **throw a `ReferenceError` on the server** and crash SSR rendering for that request. Angular provides the `PLATFORM_ID` injection token plus two helper functions from `@angular/common` to branch behavior safely:

```typescript
import { isPlatformBrowser, isPlatformServer } from '@angular/common';
import { PLATFORM_ID, inject } from '@angular/core';

const platformId = inject(PLATFORM_ID);

if (isPlatformBrowser(platformId)) {
  // safe to touch window/document/localStorage here
}
if (isPlatformServer(platformId)) {
  // server-only logic (e.g., setting response status codes, reading request headers)
}
```

`PLATFORM_ID` is a DI token whose value differs depending on which platform bootstrapped the app (`platform-browser` vs `platform-server`). `isPlatformBrowser`/`isPlatformServer` are just readable, type-safe comparisons against that token's known values — always prefer them over manually checking `typeof window !== 'undefined'`, because they integrate with Angular's DI and testing (you can inject a mock `PLATFORM_ID` in unit tests), whereas raw `typeof window` checks are a plain JS idiom that works but doesn't compose with Angular's architecture (e.g., it can't be overridden per test, and doesn't communicate intent as clearly in code review).

### 2.5 Hydration (non-destructive rebootstrap)

**Before Angular 16's hydration feature**, when the client-side app booted on top of server-rendered HTML, Angular did not know the DOM was already correct — it destroyed the server-rendered DOM entirely and re-rendered everything from scratch on the client. This caused:
- A visible **flicker/flash** as content briefly disappeared and reappeared.
- Wasted rendering work (defeats part of the performance benefit of SSR).
- Loss of state in things like focused inputs, video playback position, or CSS animations that were already running.

**With hydration** (`provideClientHydration()`, stable since Angular 16/17), the client instead **walks the existing DOM tree and the existing Angular component tree in parallel**, matching them up node-by-node, attaching event listeners and component instances to the *already-rendered* elements, without regenerating markup. This is why hydration requires the server-rendered HTML structure to exactly match what client-side rendering would produce — any mismatch triggers Angular's hydration-mismatch detection, which logs a warning/error and falls back to destroy-and-rerender for the affected subtree (defeating the benefit locally, but not crashing the whole app).

```typescript
// app.config.ts (client)
import { provideClientHydration, withEventReplay } from '@angular/platform-browser';

export const appConfig: ApplicationConfig = {
  providers: [
    provideClientHydration(withEventReplay()),
  ],
};
```

`withEventReplay()` is a companion feature: it captures user interactions (e.g., a button click) that happen *during* the hydration window — before Angular's event listeners are actually attached — and replays them once hydration completes, so an eager click isn't silently dropped.

## 3. Code Examples

### 3.1 TransferState usage to prevent duplicate HTTP calls

```typescript
// product.service.ts
import { Injectable, PLATFORM_ID, inject, makeStateKey, TransferState } from '@angular/core';
import { isPlatformServer } from '@angular/common';
import { HttpClient } from '@angular/common/http';
import { Observable, of, tap } from 'rxjs';

interface Product {
  id: number;
  name: string;
  price: number;
}

const PRODUCTS_KEY = makeStateKey<Product[]>('products');

@Injectable({ providedIn: 'root' })
export class ProductService {
  private http = inject(HttpClient);
  private transferState = inject(TransferState);
  private platformId = inject(PLATFORM_ID);

  getProducts(): Observable<Product[]> {
    // 1. On the client, if the server already fetched this data, use it and stop.
    if (this.transferState.hasKey(PRODUCTS_KEY)) {
      const cached = this.transferState.get(PRODUCTS_KEY, [] as Product[]);
      // Remove the key so a later re-fetch (e.g., pull-to-refresh) isn't
      // permanently stuck serving stale server-rendered data.
      this.transferState.remove(PRODUCTS_KEY);
      return of(cached);
    }

    // 2. Otherwise fetch normally (this runs on the server during SSR,
    //    or on the client for a pure-CSR/lazy-loaded scenario).
    return this.http.get<Product[]>('/api/products').pipe(
      tap((products) => {
        if (isPlatformServer(this.platformId)) {
          // 3. Stash the result so it gets serialized into the
          //    <script id="ng-state"> blob sent to the browser.
          this.transferState.set(PRODUCTS_KEY, products);
        }
      }),
    );
  }
}
```

```typescript
// product-list.component.ts
import { Component, inject, signal } from '@angular/core';
import { ProductService } from './product.service';

@Component({
  selector: 'app-product-list',
  standalone: true,
  template: `
    <ul>
      @for (p of products(); track p.id) {
        <li>{{ p.name }} — {{ p.price | currency }}</li>
      }
    </ul>
  `,
})
export class ProductListComponent {
  private productService = inject(ProductService);
  products = signal<Product[]>([]);

  constructor() {
    this.productService.getProducts().subscribe((products) => this.products.set(products));
  }
}
```

Result: the server fetches `/api/products` once, embeds the JSON in the HTML response. The browser's first render of `ProductListComponent` reads that embedded JSON instead of firing a second HTTP request — eliminating a redundant network round-trip and any risk of the list "flickering" if the API returns slightly different data on the second call.

### 3.2 `isPlatformBrowser` guard around a browser-only API

```typescript
// scroll-position.service.ts
import { Injectable, PLATFORM_ID, inject } from '@angular/core';
import { isPlatformBrowser } from '@angular/common';

@Injectable({ providedIn: 'root' })
export class ScrollPositionService {
  private platformId = inject(PLATFORM_ID);

  restoreScrollPosition(key: string): void {
    if (!isPlatformBrowser(this.platformId)) {
      // On the server there is no window/localStorage/scroll position at all —
      // returning early here is what prevents a ReferenceError from crashing
      // the SSR render for this request.
      return;
    }

    const saved = window.localStorage.getItem(key);
    if (saved) {
      window.scrollTo(0, Number(saved));
    }
  }

  saveScrollPosition(key: string): void {
    if (!isPlatformBrowser(this.platformId)) {
      return;
    }
    window.localStorage.setItem(key, String(window.scrollY));
  }
}
```

```typescript
// A common variant: guarding a third-party chart library that manipulates the DOM directly.
import { AfterViewInit, Component, ElementRef, PLATFORM_ID, inject, viewChild } from '@angular/core';
import { isPlatformBrowser } from '@angular/common';

@Component({
  selector: 'app-chart',
  standalone: true,
  template: `<div #chartHost></div>`,
})
export class ChartComponent implements AfterViewInit {
  private platformId = inject(PLATFORM_ID);
  private chartHost = viewChild.required<ElementRef<HTMLDivElement>>('chartHost');

  async ngAfterViewInit(): Promise<void> {
    if (!isPlatformBrowser(this.platformId)) {
      return; // Chart.js/D3/etc. assume a real Canvas/SVG DOM — never run on server.
    }

    // Dynamic import keeps the library out of the server bundle entirely.
    const { Chart } = await import('chart.js/auto');
    new Chart(this.chartHost().nativeElement, { type: 'bar', data: {/* ... */} });
  }
}
```

### 3.3 `server.ts` setup

```typescript
// server.ts (Express-based, Angular 17+ style using @angular/ssr)
import 'zone.js/node';
import express from 'express';
import { AngularNodeAppEngine, createNodeRequestHandler, isMainModule, writeResponseToNodeResponse } from '@angular/ssr/node';
import { join } from 'node:path';

const browserDistFolder = join(import.meta.dirname, '../browser');

const app = express();
const angularApp = new AngularNodeAppEngine();

// Serve static files (JS/CSS/images) generated by the client build, with long
// cache headers for hashed assets.
app.use(
  express.static(browserDistFolder, {
    maxAge: '1y',
    index: false,
    redirect: false,
  }),
);

// All remaining requests are handed to Angular's server renderer.
app.use('/**', (req, res, next) => {
  angularApp
    .handle(req)
    .then((response) =>
      response ? writeResponseToNodeResponse(response, res) : next(),
    )
    .catch(next);
});

// Start the Node server only when this file is the entry point
// (as opposed to being imported by, e.g., a serverless wrapper).
if (isMainModule(import.meta.url)) {
  const port = process.env['PORT'] || 4000;
  app.listen(port, () => {
    console.log(`Node server listening on http://localhost:${port}`);
  });
}

export const reqHandler = createNodeRequestHandler(app);
```

Build/run pipeline that goes with this file:

```bash
ng build                     # produces dist/<app>/browser (client bundle) and
                              #   dist/<app>/server (server bundle, incl. server.ts output)
node dist/<app>/server/server.mjs   # boots the Express app above
```

## 4. Internal Working

### 4.1 Rendering the component tree to an HTML string, on Node

1. A request hits `server.ts`. The Express handler delegates to Angular's `AngularNodeAppEngine` (or, in older Universal setups, `CommonEngine`/`renderApplication`), passing the request URL.
2. Angular bootstraps the application **fresh, per request** using `main.server.ts`'s bootstrap function and the server `ApplicationConfig` (the one with `provideServerRendering()`). This is a full DI container + component tree instantiation — the server does not keep one "warm" app instance shared across requests; instead each request gets an isolated Angular application instance so that per-request state (route, request-scoped data) doesn't leak between concurrent users.
3. Angular's router matches the requested URL against your routes (this is why routes/guards/resolvers *do* run during SSR — they must, so that the rendered HTML corresponds to the correct page).
4. Change detection runs against the component tree, exactly as it would in the browser, except the "DOM" being written to is the Domino-based shim provided by `platform-server` rather than a real browser DOM. Component templates call the same `createElement`/`setAttribute`/`appendChild`-style rendering instructions the Ivy compiler always generates; on the server these instructions build up an in-memory shim-DOM tree instead of visible pixels.
5. Angular waits for the application to reach a **stable** state — specifically, it uses `ApplicationRef.isStable` (backed by Zone.js's `NgZone.onStable`, or increasingly by pending-task tracking for zoneless apps) so that pending async work (HTTP calls, timers registered via Angular's task tracking, resolvers) has a chance to complete before serialization happens. This is precisely why an HTTP call made in `ngOnInit` is reflected in the rendered HTML: SSR doesn't serialize the *initial* empty template, it waits for the app to settle first.
6. Once stable, `platform-server` serializes the shim-DOM tree to an HTML string, injects the `TransferState` JSON blob as a `<script>` tag, and (with hydration enabled) annotates key DOM nodes with hydration markers (`ngh` attributes) that tell the client renderer how to reconnect to this exact tree.
7. That HTML string is written to the HTTP response and sent to the browser. The Node process's per-request application instance is then discarded/garbage-collected.

### 4.2 How the client "picks up" — hydration vs. the old destroy-and-rerender model

Historically (pre-hydration Universal), step 7's HTML was purely cosmetic — a "skeleton" for the browser to paint quickly. When the client's JS bundle finished loading:
- `bootstrapApplication` ran exactly as it would for a pure CSR app.
- Angular built a brand-new component tree and a brand-new set of DOM nodes, then **replaced** the server-rendered DOM outright (typically the root element's `innerHTML` was cleared and Angular's normal creation-mode rendering took over).
- The visible symptom: a flash where server-rendered content disappears for a frame (or several) before client-rendered content reappears — sometimes called the "SSR flicker."

**With hydration** (`provideClientHydration()`), the client bootstrap process changes qualitatively:
- Angular reads the `ngh` hydration annotations left in the HTML (matching each component/view to its corresponding DOM subtree).
- Instead of calling DOM-creation instructions, Angular's hydration runtime walks the *existing* DOM nodes and **claims** them — attaching component instances, bindings, and event listeners to nodes that are already there, verifying (in dev mode) that the structure matches what client-side rendering would have produced.
- No DOM is destroyed and rebuilt. The only "new" work the client does is: wire up reactivity (signals/change detection) and event listeners on top of nodes that already exist, and reconcile `TransferState` data so components don't refetch.
- The `TransferState` payload embedded in step 6 above is read synchronously during this bootstrap, before any component's `ngOnInit`/constructor runs its own HTTP fetch, which is what makes the "check TransferState first" pattern from Section 3.1 actually work.

The net effect: users see one continuous render — the server-painted content simply "becomes interactive" rather than disappearing and being redrawn.

## 5. Edge Cases & Gotchas

### 5.1 Direct `window`/`document`/`localStorage` access crashes the server

The single most common SSR bug: any code path executed during rendering (constructors, `ngOnInit`, template getters, computed signals evaluated eagerly, etc.) that references `window`, `document`, `localStorage`, `navigator`, or DOM APIs like `IntersectionObserver`/`ResizeObserver` will throw `ReferenceError: window is not defined` (or similar) **on the server only** — it will work perfectly in local dev with `ng serve` (pure CSR) and only break once SSR is enabled or in production where SSR is live. This is why SSR bugs frequently slip past casual manual testing: the developer never ran `ng serve` against the SSR build.
- Fix: wrap with `isPlatformBrowser(platformId)`, or better, defer the logic to `afterNextRender()`/`afterRender()` (Angular 16+), which is guaranteed to run **only in the browser, after rendering** — no `isPlatformBrowser` check needed inside it.
- A more subtle variant: libraries that check `typeof window !== 'undefined'` themselves but still get *imported* at the module level with side effects that run on import (e.g., a library that does `window.foo = ...` at the top of its file, unconditionally). The crash happens before your own guard code even runs, since it's triggered by the `import` statement itself. Fix: dynamic `import()` deferred inside a browser-only branch (see Section 3.2's chart example).

### 5.2 Third-party DOM-manipulating libraries breaking SSR

Many popular UI libraries (charting libraries, rich-text editors, drag-and-drop libraries, some carousel/slider libraries, jQuery plugins) assume a full browser environment exists at *import time*, not just at usage time. Symptoms:
- Build succeeds, but `node dist/.../server.mjs` throws on the first request.
- Works fine in dev CSR but the production SSR build 500s.
- Fix strategies, in order of preference:
  1. Check if the library has an official SSR-safe / "isomorphic" build or a documented Angular Universal recipe.
  2. Lazy-load it via dynamic `import()` gated by `isPlatformBrowser`/inside `afterNextRender()`, so its module code never even loads in the Node process.
  3. As a last resort, provide a no-op/stub implementation of the library's entry point on the server (via a custom Webpack/`esbuild` alias or DI token swap) so the import resolves to a harmless stub during SSR builds.

### 5.3 Memory leaks from long-lived server processes

A CSR app's "process" is a single browser tab — if it leaks memory, closing the tab reclaims everything. An SSR Node server, by contrast, is a **long-lived process handling thousands of requests** without restarting. Any subscription, timer, or event listener that isn't properly cleaned up per request accumulates across every request the process ever serves, eventually causing:
- Gradually increasing memory usage (visible in server monitoring) until the process crashes or gets OOM-killed by the orchestrator (Kubernetes, PM2, etc.).
- Increasingly slow responses as garbage collection has to work harder.

Common causes specific to SSR: singleton services (`providedIn: 'root'`) that start an `setInterval`/`setTimeout` or subscribe to a long-lived `Observable` in their constructor — in a browser this runs once per tab load, but on the server a fresh application (and thus a fresh instance of that singleton) is created **per request**, so an unguarded `setInterval` effectively spawns a new interval timer on every single request forever, none of which are ever cleared, since there's no "component destroyed" moment analogous to closing a browser tab. Mitigation: ensure request-scoped resources are torn down (Angular's server rendering does call `ApplicationRef.destroy()`/`ngOnDestroy` on the per-request app instance after rendering completes, so put cleanup in `ngOnDestroy`/`DestroyRef.onDestroy()` rather than relying on process exit), and avoid creating global timers in services that get reinstantiated per SSR request.

### 5.4 Flash of content before/after hydration ("flash of unhydrated content" and hydration mismatches)

Even with hydration enabled, two related visual glitches remain possible:
- **Un-styled/partially-styled flash**: if critical CSS isn't inlined or the stylesheet `<link>` loads slower than the HTML paints, users can briefly see unstyled server-rendered markup (a variant of FOUC — flash of unstyled content) before CSS applies.
- **Hydration mismatch**: if the server-rendered HTML structure doesn't exactly match what the client-side component tree would produce (e.g., a component conditionally renders different content based on `Math.random()`, `Date.now()`, or a value read from `window` that differs between server and client, or a template uses `@if` guarded by something server/client-divergent), Angular's hydration logic detects the DOM doesn't match, logs an `NG0500`-family hydration error to the console, and falls back to destroying and re-rendering that subtree — reintroducing the exact flicker hydration was meant to eliminate, but scoped to the mismatched component rather than the whole page. Common real-world triggers: rendering `new Date().toLocaleString()` directly in a template (timezone/locale differences between server and browser), or any A/B-test/feature-flag value that's resolved differently per environment.
- Directly related: content that legitimately *should* differ between server and client (e.g., "Welcome back, {{ userName }}" once a client-only auth token is read from `localStorage`) is expected to swap after hydration — this is normal and not a bug, but it does mean a user may see a one-frame content change after the page becomes interactive; this is unavoidable for genuinely client-only data.

## 6. Interview Questions & Answers

**Q1: What is SSR in the context of Angular, and how does it differ from CSR?**
A: In client-side rendering (CSR), the server sends a mostly empty HTML shell and a JS bundle; the browser executes the bundle to build the DOM and fetch data, so the user sees a blank page until JS finishes loading and running. In server-side rendering (SSR), Angular runs the application on a Node.js server for each request, renders the component tree into a complete HTML string reflecting that specific page's content, and sends that fully-formed HTML to the browser immediately — so there's visible content before any client JS executes. The client then hydrates (or, historically, re-renders) on top of that HTML once its bundle loads.

**Q2: Why would a team choose to add SSR to an Angular app? What concrete metrics improve?**
A: Three main reasons: (1) SEO — crawlers and social-media link-preview bots get a complete HTML document immediately rather than relying on JS execution, which some crawlers don't do at all; (2) perceived performance — First Contentful Paint and Largest Contentful Paint improve because the browser paints server-provided markup while the JS bundle is still downloading/parsing; (3) Core Web Vitals — LCP specifically improves (content is visible sooner) which is a direct Google ranking signal. It's worth noting SSR does *not* automatically improve Time to Interactive/INP — the page can look ready before it can actually respond to input, which is exactly the gap hydration is designed to narrow.
**Interviewer intent:** Checking whether the candidate understands SSR is a perceived-performance and discoverability tool, not a magic "make everything faster" switch, and can distinguish between the metrics it actually helps.

**Q3: What is `provideServerRendering()` and where does it belong in an Angular app's configuration?**
A: `provideServerRendering()` (from `@angular/platform-server`) is a provider function that registers everything the server-side renderer needs — the platform-server DOM shim wiring and hydration support — into the application's DI providers. It belongs exclusively in the server-side `ApplicationConfig` (conventionally `app.config.server.ts`), merged on top of the shared client config via `mergeApplicationConfig()`. It must never be added to the browser config, since it configures server-only rendering behavior.

**Q4: What is the `platform-server` package responsible for?**
A: It's the package that lets the exact same Angular application code run inside Node.js instead of a browser. It supplies a headless DOM implementation (so `document.createElement`, `querySelector`, etc. work without a real browser), the `renderApplication`/`renderModule` APIs to bootstrap an app and serialize its rendered output to an HTML string, and the plumbing that captures component styles so they get correctly emitted into the rendered `<head>`. It's the server-side counterpart to `platform-browser`.

**Q5: What problem does `TransferState` solve, and how does it work?**
A: Without it, SSR causes duplicate data fetching: the server fetches data to render the page, then the client re-fetches the same data when it boots/hydrates, wasting a network round-trip and risking a flash if the two responses differ. `TransferState` is a key/value store that the server serializes into the rendered HTML as a JSON `<script>` tag; on the client, before firing an HTTP call, code checks whether that key already exists in `TransferState` and uses the cached value instead of calling the API again. Angular's `HttpClient` does this automatically for standard HTTP requests when transfer-state caching is enabled; for other data sources (raw `fetch`, GraphQL, third-party SDKs) you use the `TransferState`/`makeStateKey` API manually, as shown by calling `.set()` on the server after a fetch and `.get()`/`.hasKey()` on the client before issuing one.

**Q6: Why does directly referencing `window` or `document` in a component often work fine in development but crash in production once SSR is enabled?**
A: `ng serve` runs pure CSR in a real browser, so `window`/`document` always exist. Once SSR is turned on, the same component code also executes inside Node.js during server rendering, where there is no `window`, `document`, or `localStorage` global at all — referencing them throws a `ReferenceError` that aborts rendering for that request. This is why SSR-specific bugs are easy to miss without explicitly testing the SSR build (`npm run build` + running the generated `server.mjs`) rather than only running `ng serve`.
**Interviewer intent:** Testing whether the candidate has actually hit this in practice — it's the single most common real-world SSR bug, and a candidate who's shipped SSR apps will have a war story here.

**Q7: How do `isPlatformBrowser` and `isPlatformServer` work, and why prefer them over `typeof window !== 'undefined'`?**
A: They're helper functions from `@angular/common` that compare the injected `PLATFORM_ID` DI token's value against Angular's known platform identifiers, returning a boolean for whether the code is currently executing in a browser or on the server (via `platform-server`) respectively. They're preferred over a raw `typeof window` check because they integrate with Angular's dependency injection — `PLATFORM_ID` can be overridden in unit tests to simulate either platform, whereas a hardcoded `typeof window` check can't be substituted — and because they communicate architectural intent explicitly in code review (this is a platform branch, not an incidental feature-detection check).

**Q8: What is hydration in Angular, and what problem did it solve relative to older Angular Universal behavior?**
A: Before hydration (pre-Angular 16), when the client's JS bundle booted on top of server-rendered HTML, Angular didn't know that DOM was already correct, so it destroyed it entirely and re-rendered the whole component tree from scratch on the client — causing a visible flicker as content disappeared and reappeared, wasting the rendering work SSR had already done, and losing any live state (focused input, running CSS animation, video playback position) in the discarded nodes. Hydration (`provideClientHydration()`) instead has the client walk the existing server-rendered DOM and match it node-by-node against the client-side component tree, attaching component instances and event listeners to the already-existing DOM nodes rather than recreating them — eliminating both the flicker and the duplicated rendering work.

**Q9: What is `withEventReplay()` and why is it needed alongside hydration?**
A: Hydration takes a short but non-zero amount of time to attach event listeners to the server-rendered DOM. If a user clicks a button in that window — after the HTML is visible but before hydration has wired up the corresponding click handler — that click would normally be silently lost, because no listener was attached yet. `withEventReplay()` (used with `provideClientHydration(withEventReplay())`) captures such early interactions and replays them once hydration finishes attaching the real listeners, so an eager user's click still registers correctly instead of appearing to do nothing.

**Q10: Walk through what happens, end-to-end, when a browser requests a page from an SSR-enabled Angular app.**
A: The request hits the Node server (e.g., `server.ts`'s Express app), which delegates to Angular's server-rendering engine with the request URL. Angular bootstraps a fresh application instance for that request using the server `ApplicationConfig` (with `provideServerRendering()`), the router matches the URL to a route (running guards/resolvers), and change detection runs, writing to the platform-server DOM shim rather than a real browser DOM. Angular waits for the application to reach a stable state (pending HTTP calls/timers/resolvers resolved) before serializing. The shim-DOM tree is then serialized to an HTML string, `TransferState` data and hydration annotations are embedded, and that HTML is sent as the HTTP response. The per-request server-side application instance is then destroyed. In the browser, the HTML paints immediately; once the JS bundle loads, `bootstrapApplication` runs on the client, reads the hydration annotations and `TransferState` payload, and reconciles with the existing DOM (hydrating) instead of rebuilding it, then wires up interactivity.

**Q11: Your team added a Chart.js-based dashboard component and the app now crashes on the SSR build with `ReferenceError: HTMLCanvasElement is not defined`. Walk through how you'd diagnose and fix it.**
A: First confirm the error only happens with SSR (run the SSR build/server, not `ng serve`) to isolate it as a platform issue rather than a general bug. The root cause is almost certainly that Chart.js (or a module it imports) expects `HTMLCanvasElement`/Canvas APIs which don't exist in the Node/Domino shim used by `platform-server`. Two things can trigger it: either the chart-creation code itself running during SSR, or merely *importing* the library at the module top-level triggering side effects. Fix: guard the code that instantiates the chart with `isPlatformBrowser(platformId)` (or move it into `afterNextRender()`, which only runs client-side), and critically, use a dynamic `import('chart.js/auto')` *inside* that guarded block rather than a static top-level import, so the library's module code — and any DOM assumptions it makes at load time — never executes in the Node process at all. Rebuild and re-run the SSR server to confirm the error is gone and the chart still renders correctly once the client boots.
**Interviewer intent:** Practical debugging skill — checking the candidate knows the difference between "usage-time" and "import-time" DOM access, since a naive `isPlatformBrowser` guard around only the `new Chart(...)` call, with a static import left at the top of the file, would still crash.

**Q12: An Angular SSR server has been running for a few days under load and its memory usage keeps climbing until it OOM-crashes. What SSR-specific causes would you investigate first?**
A: Unlike a CSR app where a leak is scoped to a single browser tab, an SSR Node server is a single long-lived process handling many requests, and Angular creates a fresh application instance per request — so anything not cleaned up accumulates across every request the process ever serves. Prime suspects: a `providedIn: 'root'` singleton service whose constructor starts a `setInterval`/`setTimeout` or subscribes to a long-lived observable without ever clearing it — since a new instance of that "singleton" is created per SSR request, each request spawns another timer that's never torn down. Also check for subscriptions to global/static event emitters that outlive the per-request `ApplicationRef`, and confirm cleanup logic actually lives in `ngOnDestroy`/`DestroyRef.onDestroy()` (Angular does call `ApplicationRef.destroy()` on the per-request instance after rendering, so correctly-placed cleanup there will run) rather than assuming process-level cleanup that never happens because the process itself doesn't restart between requests.
**Interviewer intent:** Distinguishing candidates who've only ever run SSR locally/in a demo from those who've operated it in production long enough to hit process-level resource exhaustion.

**Q13: What is the role of `ApplicationRef.isStable` (or NgZone stability) in SSR rendering, and why does it matter for data fetching?**
A: SSR needs to know *when* to serialize the rendered DOM to HTML — too early, and async data (HTTP responses, resolved route resolvers) won't have arrived yet, producing HTML with empty/loading states baked in. Angular's server renderer waits for the application to become "stable" — meaning Zone.js reports no pending macrotasks/microtasks it's tracking (or, in zoneless apps, Angular's own pending-task tracking reports no outstanding work) — before serializing the DOM. This is why data fetched in `ngOnInit` via `HttpClient` correctly appears in the server-rendered HTML: the renderer effectively waits for that request to resolve before considering the app "done" and taking the HTML snapshot.
**Interviewer intent:** Tests understanding of the mechanism that makes SSR "just work" with async data — many candidates use SSR without knowing why the timing works out correctly.

**Q14: Two components in the same app need genuinely different content on server vs. client (e.g., one reads a `localStorage`-based feature flag). Does this break hydration, and how should it be handled?**
A: Content that is legitimately client-only (data unavailable during server rendering, like `localStorage` values or `window`-derived state) is expected to differ between the server-rendered HTML and the eventual client-rendered state, and this alone doesn't "break" hydration in the sense of crashing — Angular's hydration mismatch detection is about structural DOM mismatches (different elements/text at the same position), not about content that's designed to update after the client determines its real value. The correct pattern is to render a server-safe default/placeholder during SSR (guarded with `isPlatformServer`) and update to the real client-only value inside `isPlatformBrowser` logic (ideally in `afterNextRender()`), accepting that the user may see a brief, deliberate content swap right after hydration — as opposed to trying to force identical output that isn't actually knowable on the server, which isn't achievable and isn't the problem hydration mismatch detection is meant to catch.

## 7. Quick Revision Cheat Sheet

- **SSR** = Angular renders the app to an HTML string on Node.js per request, so the browser gets full content before JS loads/runs. "Angular Universal" = the historical/architectural name for this "runs on server and browser" capability, now built into core Angular (`ng add @angular/ssr`, `--ssr` flag).
- **Benefits**: SEO (crawlers get complete HTML immediately), better FCP/LCP (perceived performance), improved Core Web Vitals (mainly LCP) — does **not** by itself improve TTI/INP; that gap is what hydration narrows.
- **`provideServerRendering()`** (`@angular/platform-server`) — goes only in the server `ApplicationConfig` (`app.config.server.ts`); wires up server rendering + hydration support.
- **`platform-server` package** — lets Angular run in Node via a headless DOM shim (Domino); exposes `renderApplication`/`renderModule`.
- **`TransferState`** — `makeStateKey()` + `.set()` on server, `.hasKey()`/`.get()`/`.remove()` on client — avoids re-fetching data the server already fetched; `HttpClient` does this automatically with transfer-state caching enabled.
- **`isPlatformBrowser`/`isPlatformServer`** (`@angular/common`, driven by `PLATFORM_ID` token) — the DI-integrated, test-friendly way to branch platform-specific code; prefer over `typeof window !== 'undefined'`. Prefer `afterNextRender()`/`afterRender()` for browser-only side effects when possible — no manual guard needed.
- **Hydration** (`provideClientHydration()`) — client reconnects to existing server-rendered DOM instead of destroying/re-rendering it; eliminates flicker and duplicate rendering work; requires structural server/client markup match or falls back per-subtree.
- **`withEventReplay()`** — captures clicks that happen before hydration finishes attaching listeners, replays them afterward.
- **Common pitfalls**: direct `window`/`document`/`localStorage` access crashing the server; third-party libraries touching the DOM at import time (fix: dynamic `import()` behind a platform guard); long-lived server process memory leaks from uncleared timers/subscriptions in per-request singleton instances; hydration mismatches from server/client-divergent content (`Date.now()`, `Math.random()`, locale-dependent formatting).
- **Pipeline**: `main.ts` (client bootstrap) + `main.server.ts` (server bootstrap) + `app.config.server.ts` (`provideServerRendering()`) + `server.ts` (Express/Node HTTP handler using `AngularNodeAppEngine`) → `ng build` emits both `browser` and `server` output → `node dist/<app>/server/server.mjs` runs it.
- Server creates a **fresh application instance per request** (not one shared warm instance) — rendering waits for `ApplicationRef` stability (pending HTTP/timers resolved) before serializing to HTML.
