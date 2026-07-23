# Chapter 70: SSR Scenarios

## Scenario 1: Third-party library crashes the server with "window is not defined"

**Situation:** Your Angular app uses `provideServerRendering()` for SSR. After integrating a third-party analytics/carousel library that references `window`, `document`, and `navigator` directly at import time, the Node server process throws `ReferenceError: window is not defined` on every request, and the app never renders on the server.

**Question:** How do you diagnose and fix a crash caused by a browser-only third-party library during SSR?

**Answer:** The root cause is that the library executes browser-global-dependent code either at module-load time (top-level code) or during `ngOnInit`/constructor without checking the execution platform. Angular runs the exact same component tree on the server (Node) and the browser, so any code path that unconditionally touches `window`, `document`, `navigator`, or `localStorage` will crash server-side.

Fix strategy, in order of preference:

1. **Guard with `isPlatformBrowser`/`isPlatformServer`** so the library only initializes in the browser.
2. **Defer loading to `afterNextRender`/`afterRender`** (Angular 16+ lifecycle hooks that only run in the browser, replacing manual `isPlatformBrowser` + `ngAfterViewInit` combos).
3. If the library **cannot even be imported** on the server (crashes at import, not just at call time), lazy-load it dynamically with `import()` inside a browser-only guard so its module code never executes in Node at all.

```typescript
import {
  Component,
  ElementRef,
  PLATFORM_ID,
  afterNextRender,
  inject,
} from '@angular/core';
import { isPlatformBrowser } from '@angular/common';

@Component({
  selector: 'app-carousel',
  standalone: true,
  template: `<div #carouselHost></div>`,
})
export class CarouselComponent {
  private elementRef = inject(ElementRef);
  private platformId = inject(PLATFORM_ID);

  constructor() {
    // afterNextRender only ever runs in the browser, after the first render.
    // It's the modern replacement for manually checking isPlatformBrowser
    // inside ngAfterViewInit.
    afterNextRender(async () => {
      // Dynamic import ensures the library's module-level code
      // (which touches `window` at import time) never runs on the server.
      const { initCarousel } = await import('third-party-carousel');
      initCarousel(this.elementRef.nativeElement);
    });
  }
}
```

For cases where you don't control the component (e.g., it comes from a shared UI library), wrap it in a `*ngIf="isBrowser"` structural check backed by a signal set from `isPlatformBrowser(this.platformId)`, or defer it entirely with `@defer (on viewport; hydrate on interaction)` combined with `SsrSkip`-style patterns so the block only renders client-side content.

As a last resort, some teams alias problematic packages to no-op stub modules in `server.ts`'s build config (via `tsconfig.server.json` path mapping) so bundlers never even bundle the offending code into the server bundle — but prefer the `afterNextRender` approach since it keeps a single codebase without build-time forking.

**Interviewer intent:** Tests whether the candidate understands that SSR executes the *same* component code in Node, and knows the current idiomatic tools (`afterNextRender`, `isPlatformBrowser`) rather than legacy workarounds like checking `typeof window !== 'undefined'` scattered everywhere.

---

## Scenario 2: Duplicate API calls on server and client

**Situation:** A product listing page calls `this.http.get('/api/products')` inside `ngOnInit`. In the network tab during a hard reload, the team sees the API called once from the Node server (to render the initial HTML) and then called *again* from the browser immediately after hydration — doubling backend load and causing a visible content flash if the second response differs.

**Question:** Why does this duplication happen, and how do you eliminate it?

**Answer:** SSR produces static HTML by running the app once on the server. When that HTML reaches the browser, Angular bootstraps the app again client-side to attach event listeners and make it interactive (hydration). Without any special handling, `HttpClient` doesn't know the server already fetched the data, so it fetches it again on the client — this is the classic "double data fetching" problem.

The fix is **`TransferState`**, which Angular's `HttpClient` uses automatically as of Angular's built-in HTTP transfer cache when you enable `withHttpTransferCacheOptions()`/`provideClientHydration(withHttpTransferCacheOptions())`, or manually via the `TransferState` API for non-HTTP data.

```typescript
// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideHttpClient, withFetch } from '@angular/common/http';
import {
  provideClientHydration,
  withHttpTransferCacheOptions,
  withEventReplay,
} from '@angular/platform-browser';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(withFetch()),
    provideClientHydration(
      withEventReplay(),
      withHttpTransferCacheOptions({
        includePostRequests: false, // GET is cached by default; opt in to POST carefully
        includeHeaders: [],         // avoid caching sensitive headers
      }),
    ),
  ],
};
```

With this enabled, the server's `HttpClient` responses are automatically serialized into a `<script id="ng-state">` JSON blob embedded in the HTML. On the client, `HttpClient` checks the transfer store *before* issuing a request; if a matching cached entry exists (matched by URL + method + params), it resolves from that cache synchronously instead of hitting the network — then evicts the entry so subsequent calls (e.g., a manual refresh button) go to the network as normal.

For custom, non-HttpClient async work (e.g., a direct `fetch()` call or a third-party SDK call), use `TransferState` manually:

```typescript
import { Injectable, TransferState, makeStateKey, inject, PLATFORM_ID } from '@angular/core';
import { isPlatformServer } from '@angular/common';

const PRODUCTS_KEY = makeStateKey<Product[]>('products');

@Injectable({ providedIn: 'root' })
export class ProductsService {
  private transferState = inject(TransferState);
  private platformId = inject(PLATFORM_ID);

  async getProducts(): Promise<Product[]> {
    if (this.transferState.hasKey(PRODUCTS_KEY)) {
      const cached = this.transferState.get(PRODUCTS_KEY, []);
      this.transferState.remove(PRODUCTS_KEY); // consume once
      return cached;
    }
    const data = await this.fetchFromSdk();
    if (isPlatformServer(this.platformId)) {
      this.transferState.set(PRODUCTS_KEY, data);
    }
    return data;
  }
}
```

Tradeoff: transfer-cached state inflates the initial HTML payload (it's embedded JSON), so for very large responses you should cache only what's needed for the first paint and lazily fetch the rest client-side after hydration.

**Interviewer intent:** Confirms the candidate understands SSR's "render twice" model and knows the concrete API (`TransferState` / HTTP transfer cache) rather than vaguely saying "cache it somehow."

---

## Scenario 3: Hydration mismatch error in production only

**Situation:** In production, users occasionally see a console error: `NG0500: Hydration was requested but no host node was found` or `NG0752: The server-rendered content was not matched` (destroy-and-recreate happens for a whole subtree), and the page visibly flickers on load. It doesn't reproduce locally under `ng serve`, only after a real SSR build.

**Question:** What causes hydration mismatches, and how would you track down and fix this one?

**Answer:** Hydration works by Angular walking the DOM the server produced and "claiming" existing nodes instead of re-creating them, matching them against the DOM the client-side render *would have produced*. A mismatch error means the client's expected DOM shape differs from what the server actually emitted. Common causes:

1. **Non-deterministic rendering** — code using `Math.random()`, `Date.now()`, or `new Date()` directly in a template/computed value produces different output server vs. client.
2. **Direct DOM manipulation** outside Angular's control (e.g., a directive doing `this.el.nativeElement.innerHTML = ...` or a jQuery plugin mutating the DOM) — Angular's hydration doesn't know about those changes and expects the "clean" DOM shape.
3. **Invalid HTML nesting** that the browser silently "corrects" (e.g., a `<div>` inside a `<p>`, or a `<table>` without `<tbody>`) — the server emits the invalid HTML, the browser parser reshapes it before Angular ever sees it, and hydration then sees a DOM tree that doesn't match.
4. **`@if`/`@for` conditions relying on browser-only state** (e.g., `isPlatformBrowser` conditionally rendering different template branches) producing different structure server vs client on the very first render.

Debugging approach:

```typescript
// app.config.ts — enable verbose hydration logging in non-prod to catch these early
import { provideClientHydration, withHttpTransferCacheOptions, ɵwithHydrationErrorHandling } from '@angular/platform-browser';
```

In practice, Chrome DevTools + `ng.getComponent()` plus comparing `view-source:` (server HTML) against the rendered DOM in a diff tool quickly surfaces which subtree changed. For this specific bug, the culprit was a `<span>{{ formatDate(item.createdAt) }}</span>` where `formatDate` used `new Date().getTimezoneOffset()` — the server (running in UTC in the Docker container) and the client browser (running in the user's local timezone) produced different formatted strings, so the text node's initial value mismatched.

Fix: make the value deterministic between server and client, or explicitly skip hydration for content that legitimately differs by rendering it only after hydration:

```typescript
@Component({
  selector: 'app-item-date',
  standalone: true,
  template: `
    @if (isBrowser()) {
      <span>{{ localDate() }}</span>
    } @else {
      <span ngSkipHydration>{{ utcDate() }}</span>
    }
  `,
})
export class ItemDateComponent {
  private platformId = inject(PLATFORM_ID);
  isBrowser = signal(isPlatformBrowser(this.platformId));
  // ...
}
```

`ngSkipHydration` (an attribute Angular recognizes) tells the hydration algorithm to skip diffing that subtree and just re-render it client-side — a valid escape hatch for legitimately-non-deterministic or third-party-mutated regions, used sparingly since it reintroduces a flash for that subtree.

**Interviewer intent:** Tests deep understanding of *why* hydration mismatches happen (determinism + DOM shape matching), not just "add ngSkipHydration everywhere," which is an anti-pattern if overused.

---

## Scenario 4: SEO metadata must be set per-route, dynamically, from API data

**Situation:** A blog/e-commerce app needs each article/product page's `<title>`, meta description, Open Graph tags, and canonical URL to reflect the specific article/product fetched from the API — and crawlers must see this in the initial SSR response, not after client-side JS runs.

**Question:** How do you implement per-route dynamic SEO metadata correctly in an SSR Angular app?

**Answer:** Use Angular's `Title` and `Meta` services from `@angular/platform-browser`, called from a route-level resolver or the component itself, so the values are set *during* the server render pass (before Angular serializes the HTML) — this is what makes them crawler-visible without JS execution.

```typescript
// product.resolver.ts
import { ResolveFn } from '@angular/router';
import { inject } from '@angular/core';
import { Title, Meta } from '@angular/platform-browser';
import { ProductsService } from './products.service';

export const productSeoResolver: ResolveFn<Product> = async (route) => {
  const productsService = inject(ProductsService);
  const title = inject(Title);
  const meta = inject(Meta);

  const product = await productsService.getById(route.paramMap.get('id')!);

  title.setTitle(`${product.name} | Acme Store`);
  meta.updateTag({ name: 'description', content: product.shortDescription });
  meta.updateTag({ property: 'og:title', content: product.name });
  meta.updateTag({ property: 'og:image', content: product.imageUrl });
  meta.updateTag({ property: 'og:type', content: 'product' });
  meta.updateTag({ rel: 'canonical', href: `https://acme.com/products/${product.slug}` }, 'rel="canonical"');

  return product;
};

// routes.ts
export const routes: Routes = [
  {
    path: 'products/:id',
    loadComponent: () => import('./product-detail.component').then(m => m.ProductDetailComponent),
    resolve: { product: productSeoResolver },
  },
];
```

For canonical `<link>` tags specifically (not a `Meta` API concern), you typically manipulate the `<head>` via `DOCUMENT` injection since `Meta` only handles `<meta>` tags:

```typescript
import { DOCUMENT } from '@angular/common';

const doc = inject(DOCUMENT);
let link: HTMLLinkElement | null = doc.querySelector("link[rel='canonical']");
if (!link) {
  link = doc.createElement('link');
  link.setAttribute('rel', 'canonical');
  doc.head.appendChild(link);
}
link.setAttribute('href', `https://acme.com/products/${product.slug}`);
```

Key considerations:
- Do this in a **resolver**, not `ngOnInit`, so metadata is set *before* the router renders the component and before the server flushes HTML — avoids a flash of default meta tags getting captured by fast crawlers or SSR snapshotting tools.
- For structured data (JSON-LD), inject a `<script type="application/ld+json">` the same way via `DOCUMENT`, populated with the fetched entity data, so rich results (price, rating stars) work in Google Search.
- Verify with `curl https://yoursite.com/products/123 | grep '<title>'` (not a browser) to confirm the *raw* SSR response — not the post-hydration DOM — contains the right tags, since browser dev tools show the live DOM which can mask issues that crawlers using raw HTML would hit.

**Interviewer intent:** Checks that the candidate knows metadata must be resolved *before* the render completes (resolver timing) and that `Title`/`Meta` services are platform-aware (no-op safe on server) — a common real-world SSR/SEO task.

---

## Scenario 5: Slow Time-to-First-Byte from data-fetching-heavy SSR

**Situation:** A dashboard route fetches five independent REST endpoints sequentially in `ngOnInit`/a resolver before rendering. Client-side this was tolerable (progressive spinners), but under SSR the entire HTML response is held up until *all* five calls finish, pushing TTFB to 3-4 seconds and hurting Core Web Vitals (TTFB, LCP).

**Question:** How do you reduce TTFB in a data-heavy SSR route without abandoning SSR?

**Answer:** TTFB balloons because Angular's SSR render is synchronous-per-request: the server won't emit bytes until the app's async rendering (zoneless/zone-based change detection settling, plus any awaited resolvers) completes. The fixes fall into a few categories:

1. **Parallelize independent requests** — the most common culprit is `await`-ing in sequence instead of `Promise.all`/`forkJoin`.

```typescript
// Bad: sequential, ~4x latency
const user = await this.userService.get();
const orders = await this.orderService.get();
const stats = await this.statsService.get();

// Good: parallel
const [user, orders, stats] = await Promise.all([
  this.userService.get(),
  this.orderService.get(),
  this.statsService.get(),
]);
```

2. **Move non-critical data out of the SSR-blocking path using `@defer`.** Only data required for above-the-fold / LCP content should block the initial render; secondary widgets can be deferred to render client-side after hydration.

```typescript
@Component({
  selector: 'app-dashboard',
  standalone: true,
  imports: [RecentActivityComponent],
  template: `
    <app-summary [stats]="stats()" /> <!-- critical, SSR-blocking -->

    @defer (on viewport; hydrate on interaction) {
      <app-recent-activity />
    } @placeholder {
      <div class="skeleton"></div>
    }
  `,
})
export class DashboardComponent {
  stats = input.required<Stats>();
}
```

3. **Cache upstream responses at the edge/CDN or in-process (short-TTL in-memory cache in the Node server)** so repeated SSR requests for the same data (e.g., a shared "top products" widget hit by every visitor) don't re-fetch from the backend on every single request.

4. **Consider SSG or ISR-style pre-rendering for pages whose data doesn't change per-request** (see Scenario 6) — if the dashboard has a mostly-static "reference data" section, pre-render that fragment at build time and only SSR the truly dynamic parts.

5. **Set realistic budgets and add server-side timeouts/fallbacks**: wrap slow calls in a timeout so one flaky downstream service doesn't stall the entire page's TTFB indefinitely; fall back to cached/stale data or an empty state.

```typescript
import { timeout, catchError, of } from 'rxjs';

this.statsService.get$().pipe(
  timeout(800),
  catchError(() => of(null)), // render with graceful fallback rather than hang
);
```

Tradeoff: parallelizing and deferring reduces perceived and actual TTFB but shifts some data-fetching to the client, meaning that content appears slightly later there and isn't in the initial crawler-visible HTML — acceptable for below-the-fold or non-SEO-critical widgets, not for primary content.

**Interviewer intent:** Evaluates whether the candidate can reason about SSR performance holistically — parallelism, deferred loading, caching, timeouts — rather than a single silver-bullet answer.

---

## Scenario 6: Choosing SSR vs. SSG vs. CSR per route in one app

**Situation:** The application has a marketing homepage and blog (content rarely changes), a product catalog (changes daily, needs SEO), and an authenticated user dashboard (highly dynamic, no SEO need, behind login). Leadership wants "the fastest possible experience everywhere" and asks the team to justify a per-route rendering strategy rather than one blanket approach.

**Question:** How do you decide which routes should use SSR, SSG (prerendering), or CSR, and how is this configured in a modern Angular app?

**Answer:** The decision matrix is driven by two questions per route: **(a) does it need to be crawlable/fast-first-paint for SEO or Core Web Vitals?** and **(b) how often does the underlying data change, and does it vary per-user?**

| Route type | Strategy | Why |
|---|---|---|
| Marketing home, static blog posts | **SSG (prerender at build time)** | Content is identical for all users and rarely changes; prerendering gives instant TTFB (served as static HTML from CDN) with zero server compute per request. |
| Product catalog / product detail | **SSR** | Needs SEO + up-to-date data (price/stock) per request; content varies but not per-user, so it can also be paired with a CDN cache with short TTL (stale-while-revalidate) to approximate SSG-like speed. |
| Authenticated dashboard | **CSR** | No SEO benefit behind auth; content is per-user and highly dynamic; SSR would add server compute cost and complexity (auth cookies, personalized data) for no crawlability gain. |

Angular's application builder (esbuild-based, `@angular/build:application`) supports this natively via **route-level render mode** configuration (`server.routes.ts` / `app.routes.server.ts` using `RenderMode`):

```typescript
// app.routes.server.ts
import { RenderMode, ServerRoute } from '@angular/ssr';

export const serverRoutes: ServerRoute[] = [
  { path: '', renderMode: RenderMode.Prerender },              // home: SSG
  { path: 'blog/:slug', renderMode: RenderMode.Prerender },     // build-time list of slugs, or getPrerenderParams
  { path: 'products/:id', renderMode: RenderMode.Server },      // SSR per request
  { path: 'dashboard/**', renderMode: RenderMode.Client },      // CSR only, server just ships the shell
];
```

For blog posts where slugs aren't known statically (or number in the thousands), supply `getPrerenderParams` to enumerate them at build time from a CMS API, or fall back to `RenderMode.Server` with edge caching if the list is too large/volatile for build-time generation.

```typescript
{
  path: 'blog/:slug',
  renderMode: RenderMode.Prerender,
  async getPrerenderParams() {
    const posts = await fetchAllPublishedSlugsFromCms();
    return posts.map(slug => ({ slug }));
  },
},
```

This route-level configuration means a single Angular application and build produces a hybrid deployment: static files for prerendered routes, a Node (or edge) function for SSR routes, and a client-only bundle fallback for CSR routes — without needing three separate apps.

**Interviewer intent:** Tests architectural judgment — can the candidate map business/SEO/performance requirements to the right rendering strategy per route, and do they know Angular's actual `RenderMode` API rather than only knowing SSR exists as a monolith setting.

---

## Scenario 7: Deploying SSR to a serverless platform (AWS Lambda / Vercel / Cloud Functions)

**Situation:** The team currently runs the Angular SSR Node server (`server.ts` using Express) on a long-running EC2/VM instance and wants to move to a serverless platform (AWS Lambda via API Gateway, or Vercel Functions) to cut idle costs and simplify ops.

**Question:** What changes are needed to run Angular Universal/SSR on a serverless platform, and what pitfalls should you watch for?

**Answer:** Serverless functions are stateless, short-lived, and cold-start-sensitive, which conflicts with a few assumptions baked into a traditional long-running Express SSR server:

1. **Replace the persistent Express listener with a request handler export.** Angular's `@angular/ssr/node` exposes `AngularNodeAppEngine`/`createNodeRequestHandler`, which can be adapted to the platform's expected handler signature instead of `app.listen(port)`.

```typescript
// server.ts (Vercel-style handler)
import { AngularNodeAppEngine, createNodeRequestHandler, writeResponseToNodeResponse } from '@angular/ssr/node';
import { isMainModule } from '@angular/ssr/node';

const angularApp = new AngularNodeAppEngine();

export async function handler(req: Request): Promise<Response> {
  const response = await angularApp.handle(req);
  return response ?? new Response('Not found', { status: 404 });
}

// Keep a local dev server entry too, gated so it's excluded from the serverless bundle:
if (isMainModule(import.meta.url)) {
  const { createServer } = await import('node:http');
  createServer(createNodeRequestHandler(angularApp)).listen(process.env['PORT'] ?? 4000);
}
```

2. **Cold starts** — every "cold" invocation re-bootstraps the Angular platform (module graph parsing, DI container setup) from scratch, which is heavier than a typical Lambda's usual JSON-in/JSON-out handler. Mitigate by:
   - Keeping the server bundle as small as possible (avoid importing unused providers into the server config; use `@defer` client-only for heavy widgets so their code isn't in the server bundle).
   - Provisioned concurrency (AWS) / edge functions with faster cold boot (Vercel Edge Runtime, though Edge Runtime doesn't support full Node APIs, which may itself block certain Node-only SSR dependencies).
   - Prerendering (SSG) whatever routes you can, per Scenario 6, so fewer requests hit the cold Lambda at all.

3. **No shared in-memory state between invocations.** Any pattern that assumed a warm, long-lived process (in-memory caches, DB connection pools kept open across requests) is unreliable — a given Lambda instance might be reused (warm) or thrown away (cold) unpredictably. Move shared caching to an external store (Redis/DynamoDB) or rely on CDN-layer caching instead of process memory.

4. **Timeouts** — serverless platforms enforce hard execution time limits (e.g., 10s default on Vercel, configurable but capped on Lambda). A slow-data-fetching SSR route (Scenario 5) that was merely "slow" on a VM can now outright fail with a `504`. Apply the same timeout/fallback and parallelization fixes, and treat serverless as forcing function to fix latency issues you might have tolerated before.

5. **Filesystem is read-only (or ephemeral) in most serverless runtimes** — if any part of the app (or a third-party library) writes temp files, cache files, or logs to disk expecting persistence, it will break or silently lose data between invocations; redirect logging to stdout (captured by the platform) and any required file writes to `/tmp` (ephemeral, per-invocation) at most.

**Interviewer intent:** Assesses whether the candidate has real deployment experience beyond `ng serve`/local Node — cold starts, statelessness, and timeout constraints are the recurring gotchas in serverless SSR interviews.

---

## Scenario 8: Debugging a memory leak in the long-running Node SSR server

**Situation:** The SSR server (a long-running Node process behind a load balancer, not serverless) shows steadily increasing RSS memory in Grafana over several hours until it OOMs and gets restarted by the orchestrator (Kubernetes liveness probe). It happens under sustained traffic, not immediately at startup.

**Question:** How would you investigate and fix a memory leak in a long-running Angular SSR Node process?

**Answer:** Unlike a browser tab that's destroyed on navigation, an SSR Node process serves *many requests over its lifetime* — so anything that accumulates per-request without cleanup compounds. Investigation steps:

1. **Capture heap snapshots over time** using `node --inspect` + Chrome DevTools Memory tab, or `node --heapsnapshot-signal=SIGUSR2` to dump snapshots under load without restarting, then diff two snapshots (e.g., after 100 requests vs. after 10,000 requests) to see which retained object types grow unboundedly.

2. **Common Angular-SSR-specific leak sources:**
   - **Not destroying the platform/module ref per request.** Each SSR request must bootstrap a fresh application instance and then be properly disposed; if using a custom `server.ts` that manually calls `renderApplication`/`bootstrapApplication` in a loop without calling `.destroy()` on the resulting application ref, injector trees, RxJS subscriptions, and zone contexts accumulate.

```typescript
import { renderApplication } from '@angular/platform-server';

export async function render(url: string): Promise<string> {
  const html = await renderApplication(bootstrap, {
    document: indexHtml,
    url,
  });
  // renderApplication internally creates + destroys the app ref per call when
  // used as documented; if you hold a manual reference (e.g., for TransferState
  // inspection) make sure you don't retain it in a module-level array/cache.
  return html;
}
```

   - **Module-level singletons that accumulate per-request data** — e.g., a service using `providedIn: 'root'` that pushes into a static array (`private static requestLog: Request[] = []`) for "debugging," never cleared. On the server, `root`-provided services can be recreated per-request (new injector per SSR render) *unless* you've accidentally hoisted state to a JS module-level `let`/`const` outside any Angular DI scope — that state is truly global to the Node process and leaks by definition.
   - **Unclosed RxJS subscriptions in services bootstrapped on the server** — e.g., a service that does `interval(1000).subscribe(...)` in its constructor for polling; this makes sense in a browser tab (destroyed on navigation) but on the server, if the injector isn't destroyed correctly, the interval keeps firing and holding closures/memory forever, and repeats once per request over the process lifetime.
   - **Global caches without eviction** — an in-memory `Map` used as a manual cache (e.g., for the "parallelize + cache" fix from Scenario 5) that never expires entries; under high-cardinality URLs (e.g., caching by full query string on a search page) this grows unboundedly.

3. **Fix pattern — bound any server-side cache and ensure DI scope correctness:**

```typescript
import { Injectable } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class ServerCacheService {
  // Bounded LRU-style cache instead of an unbounded Map; evict on TTL.
  private cache = new Map<string, { value: unknown; expiresAt: number }>();
  private readonly maxEntries = 500;

  get<T>(key: string): T | undefined {
    const entry = this.cache.get(key);
    if (!entry || entry.expiresAt < Date.now()) {
      this.cache.delete(key);
      return undefined;
    }
    return entry.value as T;
  }

  set(key: string, value: unknown, ttlMs = 30_000): void {
    if (this.cache.size >= this.maxEntries) {
      const oldestKey = this.cache.keys().next().value;
      this.cache.delete(oldestKey);
    }
    this.cache.set(key, { value, expiresAt: Date.now() + ttlMs });
  }
}
```

4. **Mitigate blast radius operationally** regardless of the code fix: configure `--max-old-space-size` appropriately, keep the Kubernetes liveness/readiness probes and memory limits tuned so a leaking pod is recycled gracefully under low traffic rather than OOM-killing mid-request, and roll deployments with enough replicas that one recycling pod doesn't cause a user-facing outage — but treat these as safety nets, not the fix.

**Interviewer intent:** Distinguishes candidates who've only run SSR locally from those who've operated it in production — probes for awareness of per-request lifecycle, DI scoping on the server, and Node memory profiling tools.

---

## Scenario 9: Hydration breaks because a component uses `Math.random()` for keys

**Situation:** A component renders a list of cards and uses `Math.random()` to generate a unique `id` attribute for each card (for anchor-linking). After enabling `provideClientHydration()`, every card's DOM gets destroyed and recreated on hydration, causing a visible flash and defeating the purpose of hydration (no CLS/flicker was the goal).

**Question:** Why does randomness break hydration, and what's the correct way to generate stable per-item identifiers?

**Answer:** Hydration's whole value proposition is *reusing* server-rendered DOM nodes instead of discarding and recreating them. To do that, Angular must produce the *same* attribute values, text content, and structural output when it does its internal reconciliation pass as it produced during the server render. `Math.random()` (or `Date.now()`, `crypto.randomUUID()` called at render time) generates a *different* value each time the function runs — once on the server, once again during the client's hydration-time re-evaluation — so Angular sees a mismatched `id` attribute and, unable to trust the rest of that node's identity, discards and rebuilds the subtree (a hydration "downgrade" for that node), which reintroduces the flicker/CLS hydration was meant to prevent.

The fix is to derive the identifier from **stable, deterministic input** — typically the item's own data (its database ID, slug, or index), never from a random or time-based source computed at render time:

```typescript
@Component({
  selector: 'app-card-list',
  standalone: true,
  template: `
    @for (card of cards(); track card.id) {
      <div [id]="'card-' + card.id" class="card">
        {{ card.title }}
      </div>
    }
  `,
})
export class CardListComponent {
  cards = input.required<Card[]>();
}
```

If you truly need a client-generated random/unique value (e.g., a one-time CSRF-safe token for a form, not meant to be crawlable or stable), generate it only in the browser via `afterNextRender`, and mark that specific element/subtree with `ngSkipHydration` so Angular doesn't attempt to match it against server output at all — an explicit opt-out rather than an accidental mismatch.

```typescript
constructor() {
  afterNextRender(() => {
    this.clientOnlyToken.set(crypto.randomUUID());
  });
}
```

**Interviewer intent:** Probes understanding that hydration requires *determinism*, not just "no errors" — a subtler failure mode than an outright crash, since the app still "works" but silently loses hydration's performance benefit.

---

## Scenario 10: `localStorage`/`sessionStorage` access crashes SSR

**Situation:** An `AuthService` reads a JWT from `localStorage` in its constructor to initialize an `isLoggedIn` signal. This works fine client-side, but SSR throws `ReferenceError: localStorage is not defined`, crashing every server render.

**Question:** How do you safely handle browser storage APIs (`localStorage`, `sessionStorage`, `cookies`) in an SSR-compatible way?

**Answer:** `localStorage`/`sessionStorage` are browser-only Web Storage APIs with no Node.js equivalent, so any unguarded access during SSR's server-side execution throws. There are two correct handling strategies depending on *why* you need the data:

1. **If the data isn't needed for the initial render (pure client-side UX state)** — guard with `isPlatformBrowser` and provide a safe default on the server:

```typescript
import { Injectable, PLATFORM_ID, inject, signal } from '@angular/core';
import { isPlatformBrowser } from '@angular/common';

@Injectable({ providedIn: 'root' })
export class AuthService {
  private platformId = inject(PLATFORM_ID);
  private isBrowser = isPlatformBrowser(this.platformId);

  isLoggedIn = signal<boolean>(this.readInitialLoginState());

  private readInitialLoginState(): boolean {
    if (!this.isBrowser) {
      return false; // safe server-side default; corrected once hydrated in browser
    }
    return !!localStorage.getItem('authToken');
  }

  login(token: string): void {
    if (this.isBrowser) {
      localStorage.setItem('authToken', token);
    }
    this.isLoggedIn.set(true);
  }
}
```

2. **If the data IS needed for the initial render to be correct (e.g., rendering a logged-in nav bar server-side, so there's no post-hydration flash from "logged out" to "logged in")** — you cannot use `localStorage` at all, because it isn't sent with the HTTP request the way cookies are. Instead, read the **auth cookie** from the incoming request on the server (via `REQUEST` injection token / Express `req.cookies`) and mirror that same cookie-reading logic in the browser via `document.cookie`, so both platforms agree on the initial value without relying on a storage mechanism that only exists client-side.

```typescript
import { REQUEST } from '@angular/core'; // request-scoped token available during SSR
import { DOCUMENT, isPlatformServer } from '@angular/common';

@Injectable({ providedIn: 'root' })
export class AuthService {
  private platformId = inject(PLATFORM_ID);
  private doc = inject(DOCUMENT);
  private request = isPlatformServer(this.platformId) ? inject(REQUEST, { optional: true }) : null;

  isLoggedIn = signal<boolean>(this.readTokenFromCookie() !== null);

  private readTokenFromCookie(): string | null {
    const cookieHeader = isPlatformServer(this.platformId)
      ? this.request?.headers.get('cookie') ?? ''
      : this.doc.cookie;
    const match = cookieHeader.match(/authToken=([^;]+)/);
    return match ? match[1] : null;
  }
}
```

This way, the server renders the *correct* logged-in/logged-out UI on the first response (good for perceived correctness and avoiding a hydration mismatch per Scenario 3, since cookie-derived state is identical between server and client for the same request), whereas `localStorage`-derived state is fundamentally invisible to the server and should only drive client-only, non-SSR-critical UI.

**Interviewer intent:** Tests whether the candidate knows the difference between "guard it so it doesn't crash" (Band-Aid, still causes flash-of-wrong-state) versus "use a mechanism visible to the server" (cookies) for state that must be correct on first paint.

---

## Scenario 11: `@defer` blocks never hydrate their interactive parts

**Situation:** A team adopted `@defer` blocks extensively to shrink server-rendered payload and speed up TTFB. However, they notice `@defer (on viewport)` blocks show their server-rendered placeholder-then-content correctly, but click handlers inside those deferred blocks don't fire until the user scrolls again or interacts twice — the first click is "swallowed."

**Question:** Why do interactions inside `@defer` blocks feel unresponsive right after they appear, and how do you fix it with hydrate triggers?

**Answer:** By default, `@defer` blocks are rendered on the server as static HTML (so content is visible and SEO-friendly immediately), but the JavaScript needed to make that block *interactive* (event listeners, change detection wiring) doesn't load until the deferred trigger condition fires **on the client** — e.g., `on viewport` means "load the component's JS when it scrolls into view," which is a *separate* client-side trigger from server rendering. If the developer only specified a render trigger (`on viewport`) without a matching **hydrate trigger**, Angular defaults hydration timing to the same trigger, but there's often a gap: the static HTML is visible (and looks clickable) before the JS chunk has finished downloading and the component has actually hydrated, so early clicks land on inert DOM.

The fix is to be explicit about **when to hydrate** versus **when to render**, using the hydrate-specific trigger syntax introduced for `@defer` + hydration:

```typescript
@Component({
  selector: 'app-product-page',
  standalone: true,
  template: `
    <app-product-summary [product]="product()" /> <!-- eagerly SSR'd, no defer -->

    @defer (on viewport; hydrate on viewport) {
      <app-reviews-widget [productId]="product().id" />
    } @placeholder {
      <div class="skeleton reviews-skeleton"></div>
    }

    @defer (on viewport; hydrate on interaction) {
      <app-live-chat-widget />
    } @placeholder {
      <button class="chat-fab">Chat with us</button>
    }
  `,
})
export class ProductPageComponent {
  product = input.required<Product>();
}
```

Key hydrate-trigger patterns:
- **`hydrate on interaction`** — ship the static placeholder HTML immediately (fast, non-blocking, SEO-neutral) but *don't* hydrate/download the component's JS until the user actually clicks/focuses it — ideal for widgets like chat, comment boxes, or "load more" buttons that many users never touch.
- **`hydrate on viewport`** — hydrate as soon as it scrolls into view (matches the visual "appears and should immediately work" expectation) — appropriate for content the user is clearly about to engage with, like a reviews section.
- **`hydrate on idle`** — hydrate once the browser is idle (`requestIdleCallback`), a good default for below-the-fold, non-critical widgets that should eventually become interactive without competing with more important startup work.
- **`hydrate never`** — explicitly keep a block server-rendered-only/static forever (e.g., a footer that's genuinely non-interactive), skipping hydration cost entirely for content that never needs JS.

The specific bug in this scenario (perceived "swallowed first click") is fixed by switching from the implicit default (hydrate timing following the render trigger, `on viewport`, which can lag behind visibility due to chunk download time) to `hydrate on interaction` for anything genuinely click-driven — this guarantees the click event itself is what triggers hydration (Angular uses a lightweight global capture-phase listener to catch that first interaction and queue it for replay once hydration completes), rather than the user needing to click twice.

**Interviewer intent:** Tests knowledge of the newer, more granular `@defer` hydrate-trigger API — a frequently-tested "latest Angular" feature distinguishing candidates current on post-hydration-stable-release features from those who only know basic `@defer (on viewport)`.

---

## Scenario 12: Event replay is disabled and users' early clicks are lost

**Situation:** QA reports that on slower devices, if a user taps a "Add to Cart" button within the first ~500ms of the page appearing (before JS has fully hydrated), the click does nothing — no error, no add-to-cart, no feedback. The team has `provideClientHydration()` configured but without `withEventReplay()`.

**Question:** What does `withEventReplay()` do, and why is omitting it a common source of this exact "lost first click" bug?

**Answer:** Between the moment the server-rendered HTML becomes visible and the moment Angular finishes hydrating (JS parsed, executed, event listeners attached, component tree reconciled), there's an unavoidable window where the page *looks* interactive but *isn't yet*. Without event replay, any click, input, or other DOM event a user triggers during that gap is handled only by the *native* DOM (if there's a default browser behavior, e.g., a real `<a href>` navigating) or is simply dropped if it needed an Angular event binding (`(click)="addToCart()"`) that hasn't been wired up yet.

`withEventReplay()` fixes this by installing a lightweight, framework-agnostic global event listener (based on the JSAction library, the same technique Google uses) *before* Angular's own bootstrapping finishes. This listener captures early user interactions (clicks, input, etc.) on any element, and once Angular finishes hydrating and the real event bindings are wired, it **replays** those captured events against the now-hydrated component tree — so the "Add to Cart" click that happened at 400ms is dispatched to the actual `addToCart()` handler once it exists at 600ms, instead of being lost.

```typescript
// app.config.ts
import { provideClientHydration, withEventReplay } from '@angular/platform-browser';

export const appConfig: ApplicationConfig = {
  providers: [
    provideClientHydration(
      withEventReplay(), // captures + replays pre-hydration DOM events
    ),
  ],
};
```

As of recent Angular versions, `withEventReplay()` is included by default when you call `provideClientHydration()` (it graduated from opt-in to default-on), but it's still worth explicitly stating in interviews/code review since some codebases pin an Angular version or explicitly opted out with `withNoHttpTransferCache()`/removed it during a migration and never noticed regressions until a bug report like this one surfaced.

Tradeoffs: event replay adds a small amount of JS (the capture listener) that must load early (it's part of the hydration runtime, not lazy), and it can occasionally replay an event against slightly different state if the component's data changed between capture and replay (e.g., a stock-limited "Add to Cart" where inventory hit zero in that window) — acceptable in virtually all real-world cases, but worth knowing as an edge case if asked "is it ever wrong?"

**Interviewer intent:** Confirms familiarity with a specific, easy-to-miss hydration configuration flag and the exact user-facing symptom (dropped early interactions) it's designed to solve — a very concrete, real production bug pattern.

---

## Scenario 13: Zone.js removal (zoneless) breaks SSR change detection timing

**Situation:** The team migrated to zoneless change detection (`provideExperimentalZonelessChangeDetection()` / the stable zoneless provider) for bundle-size and performance reasons. After the migration, SSR output is sometimes captured *before* an async data fetch inside a component resolves, so the server emits HTML with an empty/loading state instead of the final data — even though the equivalent zone-based app rendered correctly.

**Question:** Why can zoneless change detection cause incomplete SSR output, and how do you ensure the server waits for all pending work before serializing HTML?

**Answer:** With Zone.js, the SSR renderer could lean on Zone's ability to track "is there any pending async work" (macrotasks/microtasks patched by Zone) as a heuristic for "is the app stable/done rendering" before serializing. Without Zone, Angular has no automatic global signal of "all my async work is finished" — it relies instead on explicit **application stability** signals: `ApplicationRef.isStable`, and specifically for pending SSR-relevant async work, patterns like `PendingTasks` (an injectable service, `pendingTasks.run()`/`add()`) that the framework and your own code use to tell Angular "don't consider the app stable yet, I have async work in flight."

If a component fetches data via a raw `fetch()` or a non-Angular-tracked promise (e.g., a wrapped SDK) without registering it with `PendingTasks`, Angular's stability tracker doesn't know to wait, and the SSR renderer serializes the DOM as soon as *it thinks* the app is stable — which, in zoneless mode, might be before that untracked promise resolves.

Fix: explicitly register unmanaged async work with `PendingTasks` so the app-stability/SSR-completion signal correctly waits for it.

```typescript
import { Injectable, PendingTasks, inject } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class LegacySdkDataService {
  private pendingTasks = inject(PendingTasks);

  async fetchDashboardData(): Promise<DashboardData> {
    // Wrap the untracked async operation so Angular's stability/SSR
    // completion detection knows to wait for it.
    return this.pendingTasks.run(async () => {
      return legacySdk.fetchData(); // a raw callback/promise-based SDK, not HttpClient
    });
  }
}
```

Note that `HttpClient` calls, `async`/`await` inside component lifecycle hooks called from Angular's own scheduling, and signal-based reactive graphs are already correctly tracked by Angular's zoneless scheduler and don't need manual `PendingTasks` wrapping — this is specifically needed for "escape hatch" async work the framework can't see (raw timers, third-party SDK callbacks, manual `fetch()` calls not routed through `HttpClient`).

Broader guidance for zoneless SSR correctness:
- Prefer `HttpClient` (tracked) over raw `fetch()` for anything that must complete before SSR serialization.
- Avoid `setTimeout`-based "fake delay" patterns in resolvers; they resolve on the wall clock, not on real completion, and zoneless mode won't magically wait longer just because Zone used to patch timers.
- When debugging, check `applicationRef.isStable` (an observable) directly in a `console.log` during a manual server-side test render to see exactly when Angular considers the app "done" versus when your data actually arrived.

**Interviewer intent:** Tests currency with zoneless Angular (a major, actively-rolling-out change) and specifically its interaction with SSR completion timing — a nuanced, forward-looking topic that separates candidates who've only used the zone-based default.

---

## Scenario 14: `provideClientHydration()` causes a full re-render flash on complex forms

**Situation:** A multi-step reactive form (with `FormGroup`s bound via `[formGroup]`, custom `ControlValueAccessor` components, and a rich third-party date-picker) renders correctly via SSR, but on hydration the entire form subtree flashes and resets to its initial (empty) state momentarily before repopulating — worse than not having SSR at all for this page.

**Question:** Why do complex form components sometimes fail to hydrate cleanly, and what's the resolution path?

**Answer:** Angular's hydration reuses DOM nodes but form control *state* (the `FormGroup`/`FormControl` object graph, validators, and their current values) is JavaScript object state, not DOM state — it doesn't exist until the client-side bootstrap re-runs the component's constructor/`ngOnInit`, which re-creates the `FormGroup` via `this.fb.group({...})`. If that initial construction only sets *default* values (not the actual data-driven values the server used to fill the form, e.g., prefilled from a "resume your application" API call), there's a real gap: the DOM shows the server-rendered filled values, but the newly-constructed client-side `FormGroup` briefly reports empty/default values until the same data-fetch that populated the server-side form re-resolves on the client and patches the form via `patchValue()`.

Additionally, some third-party `ControlValueAccessor` implementations (especially ones wrapping non-Angular-aware widgets like certain date pickers) directly manipulate their host DOM node's attributes/classes during their own internal initialization in a way that doesn't match what the server produced, triggering the same DOM-shape mismatch as Scenario 3, causing Angular to blow away and recreate that control's DOM (visible flash) rather than hydrate it.

Resolution:

1. **Ensure the same data source populates the form identically on both platforms**, using `TransferState` (Scenario 2) so the client doesn't have to refetch-then-patch — it should construct the `FormGroup` with the *already-known* values from the very first tick, eliminating the empty-then-filled flash entirely.

```typescript
export class ApplicationFormComponent {
  private transferState = inject(TransferState);
  private fb = inject(FormBuilder);

  form = this.fb.group({
    firstName: [''],
    lastName: [''],
    // ...
  });

  constructor() {
    const initialData = this.transferState.get(APPLICATION_DATA_KEY, null);
    if (initialData) {
      this.form.patchValue(initialData); // synchronous, before first paint attempt
    }
  }
}
```

2. **For third-party form widgets that mutate their own DOM unpredictably**, mark that specific control's host element with `ngSkipHydration` so Angular deliberately skips DOM-reuse for it (accepting a small, isolated re-render for just that widget) rather than letting the mismatch cascade and force a re-render of the *entire* parent form subtree, which is the more damaging failure mode described in this scenario.

```html
<div ngSkipHydration>
  <app-third-party-datepicker [formControl]="form.controls.startDate" />
</div>
```

3. **Verify with Angular's hydration DevTools overlay / console warnings** (enabled by default in dev mode, listing which components were hydrated vs. "skipped/re-rendered") to precisely scope which control is causing the cascade rather than guessing.

The key architectural lesson: hydration correctness for forms is really a *data availability timing* problem as much as a DOM-shape problem — get the values into the client's initial construction pass via `TransferState`, and isolate truly incompatible third-party widgets with `ngSkipHydration` rather than letting one bad actor blow away an entire well-behaved form.

**Interviewer intent:** Tests whether the candidate can combine multiple prior concepts (TransferState + ngSkipHydration + mismatch root-causing) to solve a compound, realistic production bug rather than reciting hydration facts in isolation.

---

## Scenario 15: `provideServerRendering` app returns different content for the same URL depending on request headers, breaking CDN caching

**Situation:** The app is deployed behind a CDN in front of the SSR Node servers. The team enabled edge caching keyed only by URL to reduce Node compute load, but soon after, some users report seeing *other users'* personalized "Welcome back, Alex" banner, or seeing content in the wrong language despite their `Accept-Language` header.

**Question:** What went wrong with the caching strategy, and how should SSR output be cached correctly when content varies by request?

**Answer:** This is a caching-key mismatch, not strictly an Angular bug, but it's a scenario Angular SSR engineers must reason about: the CDN cached the *rendered HTML* keyed only by URL path, while the actual rendered content also depended on **request-specific inputs** — an auth cookie (for the personalized name) and the `Accept-Language` header (for localized content). Because those inputs weren't part of the cache key, the *first* request for a given URL got cached, and every subsequent request for that same URL — regardless of who made it or what language they wanted — received that first, now-stale-and-wrong, cached response.

Fixes, layered:

1. **Expand the cache key (Vary header) to include every input that affects output.** Set `Vary: Cookie, Accept-Language` (or more precisely, vary on a normalized/hashed version of just the relevant cookie, not the entire cookie header, to avoid a combinatorial explosion of cache entries) so the CDN treats requests with different auth/locale as genuinely different cache entries.

```typescript
// server.ts — set Vary explicitly on the SSR response
res.setHeader('Vary', 'Accept-Language');
```

2. **Never cache authenticated/personalized responses at a shared (CDN) layer at all.** For the "Welcome back, Alex" banner specifically, the correct architecture is usually: cache the *page shell* (product info, layout, marketing content) as anonymous/shared-cacheable HTML, and render the personalized banner as a client-only, post-hydration fetch (using the user's cookie/session, called from the browser, never hitting the shared CDN cache) — i.e., architect the personalized fragment out of the cached HTML entirely rather than trying to cache-key around it.

```typescript
@Component({
  selector: 'app-welcome-banner',
  standalone: true,
  template: `
    @defer (on immediate; hydrate on immediate) {
      @if (userName(); as name) {
        <p>Welcome back, {{ name }}</p>
      }
    } @placeholder {
      <p>Welcome</p> <!-- safe, cacheable, generic shell -->
    }
  `,
})
export class WelcomeBannerComponent {
  private authService = inject(AuthService);
  userName = signal<string | null>(null);

  constructor() {
    afterNextRender(() => {
      // Client-only, session-aware fetch — never part of the shared SSR cache.
      this.userName.set(this.authService.currentUserName());
    });
  }
}
```

3. **For genuinely cacheable-but-varying content (locale), cache per-variant explicitly** rather than relying on `Vary` alone (which many CDNs support only partially/inconsistently) — e.g., route locale into the URL path (`/en/products/1`, `/fr/products/1`) so each cached variant has its own unique cache key by construction, which is both more predictable and debuggable than header-based `Vary` caching.

4. **Set explicit, conservative `Cache-Control` headers from the SSR response itself** (`Cache-Control: private, no-store` for anything personalized; `Cache-Control: public, max-age=60, stale-while-revalidate=300` for genuinely shared content) rather than leaving the CDN's default caching heuristics to guess — this is the single most important fix, since the underlying bug was effectively "the CDN was allowed to cache something it should have been told not to."

**Interviewer intent:** Tests whether the candidate thinks about SSR in a real infrastructure context (CDN, cache keys, `Vary`, personalization) rather than treating SSR as purely an in-app concern — a distinctly senior-level scenario about caching correctness and security (data leaking between users).

---

## Scenario 16: SSR build fails only in CI with "Cannot find module" for a Node-only dependency

**Situation:** `ng build` succeeds locally and the app SSRs correctly in local `npm run serve:ssr`. In the CI pipeline (a clean Docker image), the server bundle build step fails with `Error: Cannot find module 'fsevents'` (a macOS-only optional native dependency pulled in transitively by a dev tool), and separately, a runtime error `Cannot find module 'sharp'` occurs on the deployed server in production.

**Question:** How do you resolve Node-native/platform-specific dependency issues that only surface in CI/production SSR environments?

**Answer:** Two distinct problems bundled in this scenario, both stemming from the fact that the SSR *build* and *runtime* environment (Linux container, typically) differs from the developer's local machine (often macOS/Windows):

1. **`fsevents` build-time failure** — `fsevents` is a macOS-only optional dependency of Chokidar (used by many dev/watch tools). It's marked `optional: true` in its own `package.json`, so `npm install` correctly skips it on Linux — the CI failure usually means a *lockfile* was generated on macOS with `fsevents` resolved as a mandatory-looking entry, or the CI is using a package manager/flag (`npm ci` with a stale/mismatched lockfile, or a monorepo tool not respecting `optionalDependencies`) that fails to skip it. Fix: regenerate the lockfile in a Linux environment (or in Docker) so optional-dependency resolution matches the target platform, and ensure CI uses `npm ci` against that Linux-generated lockfile rather than a locally (macOS) committed one that encodes different optional resolutions.

2. **`sharp` runtime failure** — `sharp` (an image-processing library, common in Angular apps for server-side image optimization/the `NgOptimizedImage` server loader) ships **prebuilt native binaries per platform/architecture** (e.g., `linux-x64`, `darwin-arm64`). If `npm install` runs on a developer's ARM Mac and the resulting `node_modules` (including `sharp`'s native `.node` binary) is copied verbatim into a Linux production Docker image (rather than reinstalling inside the image), the binary is architecture-incompatible and fails to load at runtime. Fix:

```dockerfile
# Dockerfile — install dependencies INSIDE the target Linux image,
# never copy a host-machine node_modules into the container.
FROM node:20-slim AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-slim AS runtime
WORKDIR /app
COPY --from=build /app/dist ./dist
COPY package.json package-lock.json ./
RUN npm ci --omit=dev  # reinstall inside the Linux runtime image, correct native binaries
CMD ["node", "dist/my-app/server/server.mjs"]
```

General principles for the interview answer:
- **Never copy `node_modules` between machines/images** for packages with native bindings — always run `npm ci`/`npm install` *inside* the final target OS/architecture.
- **Commit the lockfile and generate it consistently** (ideally via CI itself or a container matching CI, not ad hoc on individual developers' machines with different OSes).
- **Use multi-stage Docker builds** so the build toolchain (large, includes dev dependencies) doesn't bloat the final runtime image, while still ensuring native modules are compiled/installed for the runtime's actual platform.
- For truly cross-platform-sensitive native deps, check if a pure-JS or WASM alternative exists (e.g., some teams swap `sharp` for a WASM image library specifically to sidestep native-binary platform issues in multi-arch deployments).

**Interviewer intent:** Tests DevOps/build-pipeline maturity around SSR specifically — Angular SSR's Node runtime pulls in the full Node ecosystem's platform-native-dependency issues, which is a very real, frequently-encountered category of "it works on my machine" SSR bugs.

---

## Scenario 17: Redirects and 404s during SSR don't return the correct HTTP status code

**Situation:** An SSR e-commerce app's Angular Router redirects `/old-product/:id` to `/products/:id` client-side via `redirectTo`, and shows a "Product Not Found" component when an ID doesn't exist. QA/SEO audit flags that in *both* cases, the server actually responds with `HTTP 200 OK` — search engines are indexing "not found" pages as valid content, and redirect chains aren't recognized as true 301/302 redirects by crawlers or `curl -I`.

**Question:** How do you make SSR return correct HTTP status codes (301/302 for redirects, 404 for not-found, 500 for errors) instead of always 200?

**Answer:** By default, Angular's SSR pipeline resolves whatever route/component the Router lands on and serializes it to HTML with a `200 OK`, because from the Router's perspective a client-side `redirectTo` or an `@if` branch showing a "not found" template is just... a successfully rendered component — there's no inherent signal to the HTTP layer that "this should have been a 404. The fix is to explicitly set the response status code during SSR using Angular's `RESPONSE_INIT`/response token (or the Express `res` object when using a custom `server.ts`), driven from within the component/resolver that detects the not-found/redirect condition.

```typescript
// product-detail.component.ts
import { RESPONSE_INIT } from '@angular/ssr'; // token providing access to shape the SSR response
import { inject, PLATFORM_ID } from '@angular/core';
import { isPlatformServer } from '@angular/common';

@Component({ /* ... */ })
export class ProductDetailComponent {
  private responseInit = inject(RESPONSE_INIT, { optional: true });
  private platformId = inject(PLATFORM_ID);
  product = signal<Product | null>(null);

  constructor() {
    const productsService = inject(ProductsService);
    const route = inject(ActivatedRoute);

    effect(async () => {
      const id = route.snapshot.paramMap.get('id')!;
      const found = await productsService.tryGetById(id);
      if (!found && isPlatformServer(this.platformId) && this.responseInit) {
        this.responseInit.status = 404;
      }
      this.product.set(found);
    });
  }
}
```

For router-level redirects, prefer a resolver/guard-based redirect (via `Router.navigate`/a `CanActivate` guard returning a `UrlTree`) combined with setting the actual HTTP redirect status in the Express layer, since `redirectTo` alone is a *client-side* Router concept and doesn't automatically translate to a server HTTP redirect:

```typescript
// server.ts (custom Express middleware layer, before Angular's request handler)
app.get('/old-product/:id', (req, res) => {
  res.redirect(301, `/products/${req.params['id']}`); // true HTTP 301, no Angular render needed at all
});
```

Handling this at the Express/routing layer *before* Angular even renders is actually preferable for permanent redirects — it avoids the cost of bootstrapping Angular just to immediately redirect, and it produces an unambiguous, crawler-correct 301 rather than relying on Angular's render path to intuit the right status.

For a global catch-all 404 (unmatched routes), configure a wildcard route (`{ path: '**', component: NotFoundComponent }`) and set the 404 status the same way via `RESPONSE_INIT` inside that component's constructor, ensuring the *outer* HTTP response matches what's shown to the user, which matters for SEO (Google won't index 404-status pages) and for any monitoring/alerting that keys off status codes.

**Interviewer intent:** Tests whether the candidate understands that Angular Router's internal navigation state (which page is "shown") is a separate concern from the actual HTTP response status the server sends — a subtle but SEO-critical distinction unique to SSR.

---

## Scenario 18: Signals-based component re-renders correctly client-side but "freezes" its initial SSR snapshot

**Situation:** A component uses a `computed()` signal that depends on an `input()` signal to display a countdown timer (`computed(() => this.deadline() - Date.now())`). SSR renders one static value (correct for that instant), but the team is confused why, post-hydration, the value doesn't seem to "catch up" or feel laggy immediately, and separately asks whether signals need any special SSR handling at all.

**Question:** Do Angular signals require special SSR/hydration handling, and what's happening with this specific countdown timer?

**Answer:** Signals themselves don't require special SSR plumbing — they're just synchronous reactive primitives that participate in change detection like any other Angular reactivity mechanism, and Angular's SSR renderer reads their current value at render time just as it would read a plain property. There's no `TransferState`-equivalent needed purely *because* something is a signal.

The actual issue in this scenario is the same determinism problem as Scenario 9, just with a time-based value instead of a random one: `Date.now()` evaluated inside a `computed()` produces a value tied to the exact instant of evaluation. The server evaluates it once, at server-render time (e.g., "10:00:00.000"); the client re-evaluates the same `computed()` during its own initial change-detection pass a moment later — after network transfer, parsing, hydration — by which point real time has moved on (e.g., "10:00:02.400"). This isn't a mismatch *error* (hydration doesn't fail, because `computed()` output isn't part of DOM-shape reconciliation the same rigid way static attributes are, and text content differences are typically hydrated/patched rather than causing a structural re-render) — but it does mean the number the user sees can visibly "jump" once client-side JS takes over, which reads as janky/laggy rather than a smooth countdown.

Fix: don't derive continuously-changing values from wall-clock reads inside a signal that's expected to be identical across server/client; instead, treat "now" as an explicit, updating signal that's initialized once (consistently) and then driven forward by a client-only ticking mechanism after hydration:

```typescript
import { Component, input, signal, computed, afterNextRender } from '@angular/core';

@Component({
  selector: 'app-countdown',
  standalone: true,
  template: `<span>{{ remainingSeconds() }}s remaining</span>`,
})
export class CountdownComponent {
  deadline = input.required<number>(); // epoch ms, from server-fetched data (via TransferState)
  private now = signal(this.deadline() > 0 ? Date.now() : 0); // initial snapshot, same conceptually on both platforms

  remainingSeconds = computed(() =>
    Math.max(0, Math.floor((this.deadline() - this.now()) / 1000)),
  );

  constructor() {
    // Ticking is a client-only concern; never runs on the server, so
    // there's no server/client disagreement about a "moving" value —
    // the server simply shows one static snapshot, which is expected and fine.
    afterNextRender(() => {
      setInterval(() => this.now.set(Date.now()), 1000);
    });
  }
}
```

This way, the server correctly renders one static, "frozen" snapshot (which is the *correct* SSR behavior for a live-updating value — SSR output is inherently a snapshot in time, not a live feed), and the client picks up ticking smoothly from wherever that snapshot left off, without a jarring jump, because the transition point is explicit (an interval starting fresh client-side) rather than an implicit re-evaluation of a wall-clock-dependent `computed()`.

**Interviewer intent:** Tests conceptual clarity on signals + SSR (no magic integration needed, but same determinism discipline applies) and distinguishes "hydration mismatch error" from "correct-but-visually-jarring snapshot staleness," which are different failure classes with different fixes.

---

## Scenario 19: `provideClientHydration` vs. full page reload — testing SSR locally gives misleading results

**Situation:** A developer tests hydration behavior locally by editing code and using the browser's Live Reload (webpack-dev-server-style HMR) or by navigating client-side between routes, then concludes "hydration seems to work fine," but QA later finds real hydration mismatch errors that only appear on an actual **hard, full-page load** of an SSR route in a deployed environment.

**Question:** Why can local development testing give a false sense of confidence about SSR/hydration correctness, and how should you properly test it?

**Answer:** Hydration is, by definition, a **one-time process that happens only on the very first full page load** of an SSR-rendered route — the moment the browser parses server-delivered HTML and Angular's client bundle takes over that specific DOM. Once the app is running client-side, all subsequent navigation (client-side routing via `routerLink`, HMR-triggered recompiles, SPA-style route changes) is ordinary client-side rendering with no hydration involved at all — Angular is simply creating and destroying components in an already-bootstrapped, already-hydrated app. So testing exclusively via client-side navigation or dev-server HMR can never surface a hydration bug, because hydration never runs in that testing path.

Correct local testing methodology:

1. **Always test via a genuine SSR build**, not `ng serve`'s default dev server (which, depending on config, may or may not even be doing real SSR + hydration versus just CSR) — run the actual `npm run build` + `npm run serve:ssr:<project-name>` (or equivalent Node server start script) pipeline that mirrors production.
2. **Force a true hard reload** (Ctrl+Shift+R / disable cache in DevTools' Network tab) on the specific route under test — a regular reload can sometimes be served from bfcache/HTTP cache in ways that skip the real server round-trip.
3. **Check the browser console for Angular's hydration warnings**, which in dev-mode builds log explicit messages when a component/subtree couldn't be hydrated and was re-rendered instead (Angular's hydration devtools instrumentation is designed exactly for this — don't rely on "no visible flicker" as your only signal, since some mismatches are subtle).
4. **View source (`Ctrl+U` / `curl`) to inspect the RAW server HTML**, comparing it against the final hydrated DOM in the Elements panel — this is the only reliable way to see exactly what the server produced before any client-side JS touched it, since DevTools' Elements panel always shows the live (post-hydration) DOM, which can look identical even when a mismatch caused a silent subtree replacement.
5. **Add automated e2e coverage specifically for first-load hydration** (e.g., Playwright test that does a fresh `page.goto()` — a true full navigation, not an SPA transition — asserts no console errors matching `NG05` hydration error codes, and optionally screenshots to catch layout shift) so this class of regression is caught in CI rather than relying on manual hard-reload testing, which is easy for a team to forget under deadline pressure.

**Interviewer intent:** Tests whether the candidate understands hydration's "happens once, on real navigation" nature deeply enough to know *why* common dev workflows (HMR, SPA nav) systematically fail to exercise it — a meta-scenario about testing methodology rather than a specific code fix.

---

## Scenario 20: Deciding whether to keep a legacy AngularJS-style server-rendered page or migrate fully, given SSR overhead concerns

**Situation:** Leadership is concerned that maintaining Angular SSR (Node server fleet, extra deployment complexity, the entire class of bugs in Scenarios 1-19) is expensive, and proposes reverting the marketing site to plain CSR with a "loading spinner" for simplicity, citing "our competitors just use CSR."

**Question:** How do you evaluate whether SSR's operational cost is actually justified for a given app, and what would you tell leadership?

**Answer:** This is fundamentally a cost/benefit tradeoff question, and the right answer depends on quantifying *why* SSR was adopted in the first place and whether those reasons still hold — rather than treating SSR as an unconditionally "more advanced/better" default.

SSR's concrete benefits:
- **SEO/crawlability** — search engines and social-media link-preview crawlers historically execute JS poorly/inconsistently or not at all; SSR guarantees fully-formed HTML content is present regardless of crawler JS support. This is the single biggest justification for public, content-driven, SEO-dependent routes (marketing site, blog, product pages) — CSR-with-a-spinner is a real SEO risk for these.
- **First Contentful Paint / Largest Contentful Paint** — SSR delivers visible content in the initial HTML response, before any JS has downloaded/parsed/executed, which matters enormously on slow networks/low-end devices and directly affects Core Web Vitals (a ranking factor and a real user-experience metric) — CSR's "blank page then spinner then content" pattern is strictly worse for these metrics, independent of SEO.
- **Perceived performance / bounce rate** — users on flaky connections see *something* immediately with SSR rather than a blank white screen, which measurably affects conversion/bounce rates on marketing/e-commerce pages specifically.

SSR's real costs (validating leadership's concern isn't baseless):
- Node server fleet/serverless function operational overhead (Scenario 7, 8) — compute cost and ops burden that CSR-with-a-CDN-only doesn't have.
- An entire additional class of bugs (hydration mismatches, browser-API guarding, TransferState plumbing) that the team demonstrably has been spending real engineering time on (per this very chapter's scenarios).
- Slower iteration/debugging loop in some cases (needing to test full SSR builds, not just `ng serve`, per Scenario 19).

The recommendation, using the per-route decision framework from Scenario 6, is almost never "all SSR" or "all CSR" but a **hybrid**:
- Keep **SSG (prerendering)** for the truly static marketing/blog content — this captures *all* of SSR's SEO and performance benefits with *none* of its operational cost (no Node server needed at request time at all; it's just static files on a CDN) — directly addressing leadership's cost concern without sacrificing SEO.
- Keep **SSR** only for routes with genuinely dynamic, per-request, SEO-relevant content (e.g., product pages with live pricing/stock) where SSG can't apply because content changes too frequently to prebuild.
- Use **CSR** for authenticated, non-SEO app sections (dashboards) where SSR provides no benefit and only adds cost.

Concretely: "our competitors just use CSR" is not, by itself, a valid technical argument — it depends entirely on whether their business model needs organic search traffic to marketing/content pages the way this app's does; the right response to leadership is to bring **data** (current organic search traffic %, Core Web Vitals scores with/without SSR on a sample route, actual hosting cost delta between the SSR fleet and a pure static/CDN setup) rather than settling the debate on philosophy alone — and to propose the hybrid SSG/SSR/CSR split as very likely satisfying both the cost concern and the SEO/performance requirements simultaneously, rather than treating it as an all-or-nothing choice.

**Interviewer intent:** A capstone/architecture-judgment scenario testing whether the candidate can step back from implementation details and make (and defend) a business-aligned technical recommendation — distinguishing senior engineers who reason about tradeoffs holistically from those who only know how to implement whatever's asked.

---

## Quick Revision Cheat Sheet

- **SSR runs your exact component code in Node** — any unguarded `window`/`document`/`localStorage`/`navigator` access will crash the server; guard with `isPlatformBrowser`/`isPlatformServer`, or better, defer browser-only work to `afterNextRender`.
- **The app "renders twice" (server then client)** — without `TransferState` (or the built-in HTTP transfer cache via `withHttpTransferCacheOptions()`), you get duplicate API calls; cache server-fetched data into the transfer store and consume it once on the client.
- **Hydration requires determinism** — `Math.random()`, `Date.now()`, wall-clock reads, and any code path producing different output server vs. client causes hydration mismatches (`NG0500`/`NG0752`); derive keys/IDs from stable data, and use `ngSkipHydration` only as a deliberate, scoped escape hatch, not a blanket fix.
- **SEO metadata (`Title`, `Meta`, canonical links, JSON-LD) must be set in a resolver**, before the component renders, so it's present in the raw server HTML crawlers see — verify with `curl`/view-source, not browser DevTools' live DOM.
- **TTFB in data-heavy SSR routes is fixed by parallelizing requests (`Promise.all`), deferring non-critical data with `@defer`, caching upstream calls, and applying timeouts/fallbacks** so one slow dependency doesn't stall the whole response.
- **Not every route needs SSR** — use Angular's route-level `RenderMode` (`Prerender`/`Server`/`Client`) to mix SSG for static content, SSR for dynamic-but-shared content, and CSR for authenticated/no-SEO sections in a single app and build.
- **Serverless SSR deployment requires statelessness awareness** — no reliance on warm in-memory caches/process state across invocations, watch cold-start bundle size, and respect hard execution timeouts.
- **Long-running Node SSR servers can leak memory** — audit for undestroyed per-request application refs, module-level singletons holding unbounded arrays/Maps, unclosed RxJS subscriptions in server-bootstrapped services, and unbounded in-memory caches; use heap snapshots to confirm.
- **`withEventReplay()` (default-on with `provideClientHydration()`) prevents dropped early user interactions** during the gap between visible HTML and completed hydration.
- **`@defer` needs explicit hydrate triggers** (`hydrate on interaction`/`viewport`/`idle`/`never`) distinct from render triggers, or interactive widgets can appear clickable before their JS has actually hydrated.
- **Zoneless SSR needs explicit `PendingTasks` registration** for any async work Angular can't natively track (raw `fetch`, third-party SDK callbacks), or the server may serialize HTML before that work resolves.
- **CDN/edge caching of SSR output must key on every input that affects the response** (cookies, `Accept-Language`) via `Vary` and explicit `Cache-Control`, and personalized fragments should be architected out of the shared cache entirely (client-only post-hydration fetch) rather than cached per-user.
- **Native Node dependencies (`sharp`, etc.) must be installed inside the target runtime OS/architecture**, never copied from a developer's machine — use multi-stage Docker builds and Linux-generated lockfiles.
- **HTTP status codes (301/302/404/500) don't happen automatically from Router redirects or "not found" templates** — set them explicitly via the SSR response-init token (or an Express layer redirect before Angular even renders) so crawlers and monitoring see the truth.
- **Signals don't need special SSR plumbing, but wall-clock-derived `computed()` values will "jump" post-hydration** — treat continuously-changing values as client-only ticking state seeded from one consistent initial snapshot.
- **Hydration only happens on a genuine full-page load** — testing via HMR or client-side navigation can never surface hydration bugs; always test against a real SSR build with a hard reload, and add first-load e2e coverage in CI.
- **SSR is a cost/benefit decision, not a default** — justify it per route against SEO/Core-Web-Vitals needs, and prefer prerendering (SSG) wherever content doesn't require per-request freshness, to capture SSR's benefits without its full operational cost.

**Created By - Durgesh Singh**
