# Chapter 40: Security

## 1. Overview

Angular is designed to be secure by default. Unlike frameworks that leave XSS prevention entirely to the developer, Angular treats untrusted values as hostile by default: every value interpolated into a template or bound to a property, attribute, style, or URL passes through a **security context**, and Angular's built-in `DomSanitizer` strips or rejects anything it cannot prove safe. This is the single biggest reason Angular apps are, on average, harder to XSS than hand-rolled DOM manipulation or naive template engines.

This chapter is not about authentication or authorization (those get their own chapters). It is about the mechanics of **how Angular defends the DOM and the network channel**:

- How Angular's automatic sanitization works, and the four contexts it recognizes (HTML, Style, URL, Resource URL).
- The `bypassSecurityTrust*` escape hatches — what they do, why they exist, and why they are the number one source of real-world Angular XSS bugs.
- Content Security Policy (CSP), Angular's **strict CSP mode**, and nonce-based inline style/script allowances.
- CSRF/XSRF defense — the double-submit-cookie pattern and how `HttpClientXsrfModule` / `withXsrfConfiguration` implement it.
- Practical hygiene: avoiding `innerHTML`/`document.write`/`eval`-like APIs, template injection risks, and dependency vulnerability scanning (`npm audit`, Snyk, etc.) as part of the SDLC.

The mental model to carry through the chapter: **Angular sanitizes values, not code paths.** Sanitization happens at the point a value is written into the DOM via Angular's binding system. If you write to the DOM through a side door (`ElementRef.nativeElement.innerHTML = ...`, jQuery, a third-party widget, `renderer2.setProperty(el, 'innerHTML', ...)`), Angular's sanitizer is bypassed entirely because Angular never sees that write.

---

## 2. Core Concepts

### 2.1 Why Angular sanitizes automatically

Cross-site scripting (XSS) happens when attacker-controlled data is interpreted as executable code or as markup that can execute code (a `<script>` tag, an `onerror` handler, a `javascript:` URL). Angular's template compiler knows, at compile time, *where* a piece of dynamic data lands — is it inside `innerHTML`? Is it the `href` of an anchor? Is it the `src` of an `<iframe>`? Because it knows the destination, it can apply the right sanitizer for that destination automatically, without you writing any code.

This is fundamentally different from a templating library that just concatenates strings into HTML — there, the destination is not statically known, so nothing can auto-sanitize.

### 2.2 The four security contexts

Angular defines an enum, `SecurityContext`, with these members relevant to this chapter:

| Context | Enum value | Applies to | What it guards against |
|---|---|---|---|
| **None** | `SecurityContext.NONE` | Plain text interpolation `{{ }}`, most property bindings | Nothing needed — text nodes can't execute script |
| **HTML** | `SecurityContext.HTML` | `[innerHTML]` | Injected `<script>`, `<img onerror>`, `<svg onload>`, event handler attributes, `<style>` with `expression()`/behaviors |
| **Style** | `SecurityContext.STYLE` | `[style]` / `[style.xxx]` bindings that take a full CSS value from a variable | CSS-based attacks — `expression()` in old IE, `url(javascript:...)`, data exfiltration via `background: url(https://evil.com/steal?...)` |
| **URL** | `SecurityContext.URL` | `[href]`, `[src]` on `<a>`, `<area>`, most navigational attributes | `javascript:alert(1)` URLs, `data:text/html,<script>...` in navigable contexts |
| **Resource URL** | `SecurityContext.RESOURCE_URL` | `[src]` on `<script>`, `<iframe>`, `<embed>`, `<object>`, `[href]` on `<link rel="stylesheet">` | Loading and *executing* attacker-supplied code/resources — this is the most dangerous context because the loaded resource runs with the page's privileges |

Key nuance: **plain interpolation `{{ expr }}` is always safe** because it is rendered as a text node (`textContent`), never parsed as markup. You never need to sanitize a value going into `{{ }}` — there is no security context to bypass there. Sanitization only matters for **property bindings** that write into HTML/CSS/URL "sinks."

`RESOURCE_URL` has no sanitizer implementation — Angular cannot safely rewrite a URL that is about to be treated as code (a script src, an iframe src). For that context Angular **throws** unless you explicitly mark the value as trusted via `bypassSecurityTrustResourceUrl`. This is a "fail closed" design: rather than silently stripping something and possibly still being unsafe, Angular refuses to render it at all.

### 2.3 What each sanitizer actually does

- **HTML sanitizer**: Parses the HTML string using an inert DOM parser (a detached document/DOM), walks it, and removes any element/attribute not on Angular's allow-list. Removed: `<script>`, `<style>` (in some cases), event handler attributes (`onclick`, `onerror`, `onload`, etc.), `javascript:` URLs inside `href`/`src` found within that markup, `<iframe>`, `<object>`, `<embed>`, and comments (to prevent conditional-comment tricks in old IE). Kept: common formatting tags like `<b>`, `<i>`, `<p>`, `<a href="https://...">`, `<img src="https://...">` (with `src` itself further sanitized as a URL).
- **Style sanitizer**: As of recent Angular versions, style sanitization was significantly relaxed/removed for many cases because modern CSS in evergreen browsers isn't a script-execution vector the way it was in IE (no more `expression()`). But Angular still restricts style values in older/legacy sanitization paths and disallows constructing arbitrary style strings that could smuggle behavior. In practice, treat style binding sanitization as a secondary defense, not your primary one.
- **URL sanitizer**: Parses the URL and checks its scheme against an allow-list (`http`, `https`, `ftp`, `mailto`, `tel`, and relative URLs). Anything else — most importantly `javascript:` — is replaced with the string `unsafe:<value>`, which renders as an inert, non-navigating string.
- **Resource URL "sanitizer"**: There isn't one. Angular cannot know whether an arbitrary resource URL is safe to execute, so the framework simply refuses (throws `Error: unsafe value used in a resource URL context`) unless the value arrives already wrapped as a `SafeResourceUrl`.

### 2.4 `DomSanitizer` and the `bypassSecurityTrust*` family

`DomSanitizer` (from `@angular/platform-browser`) is the injectable service that performs this work. It exposes:

```typescript
sanitize(context: SecurityContext, value: any): string | null;

bypassSecurityTrustHtml(value: string): SafeHtml;
bypassSecurityTrustStyle(value: string): SafeStyle;
bypassSecurityTrustScript(value: string): SafeScript;
bypassSecurityTrustUrl(value: string): SafeUrl;
bypassSecurityTrustResourceUrl(value: string): SafeResourceUrl;
```

Each `bypassSecurityTrust*` method wraps the raw string in a branded wrapper object (e.g., `SafeHtml`). When Angular's binding machinery sees a value that is already one of these branded "Safe" types for the matching context, it **skips sanitization entirely** and writes the raw value straight to the DOM. This is the officially sanctioned escape hatch for cases where you, the developer, have verified the value is safe through some other means (e.g., it's static content you authored, or it came from a trusted, signed source) — and it is also the most common way real Angular apps get XSS'd, because "I verified this is safe" is very often wrong once user input touches the value anywhere upstream.

### 2.5 CSP and Angular's strict mode

Content Security Policy is a browser-enforced allow-list for script/style/resource origins, delivered via the `Content-Security-Policy` HTTP header (or a `<meta>` tag). A strict policy like:

```
Content-Security-Policy: script-src 'nonce-<random>'; style-src 'nonce-<random>'; object-src 'none'; base-uri 'none';
```

blocks inline `<script>` tags and `eval`, and only allows scripts/styles carrying the correct per-request nonce. This is a second, independent layer of defense: even if an attacker manages to inject markup (bypassing Angular's sanitizer somehow, or via a dependency), CSP can still stop the injected script from executing because it lacks the nonce.

Angular has first-class support for this via **`ngCspNonce`**:

```html
<html>
  <head>...</head>
  <body>
    <app-root ngCspNonce="{{ serverGeneratedNonce }}"></app-root>
  </body>
</html>
```

Angular's JIT compiler (used at runtime for things like dynamically-created components, and historically for style tags Angular injects for component styles) reads this attribute and applies the same nonce to any `<style>` elements it creates, so a strict `style-src 'nonce-...'` policy doesn't break component styling. Without this, a strict CSP would block Angular's own injected `<style>` tags for `ViewEncapsulation.Emulated` components.

Angular CLI also supports baking a placeholder nonce into the build (`"security": { "autoCsp": true }` in newer CLI versions, or manually inserting `ngCspNonce`) so that inline styles generated by Angular's style compiler carry a valid nonce without weakening the policy with `'unsafe-inline'`.

### 2.6 CSRF/XSRF protection

CSRF (Cross-Site Request Forgery) tricks a logged-in user's browser into firing a request to your API using their existing session cookie, without their consent. Angular does not prevent CSRF outright (that requires server cooperation), but `HttpClient` implements the client half of the **double-submit cookie pattern**:

1. The server sets a cookie (default name `XSRF-TOKEN`) containing a random token, readable by JavaScript (i.e., **not** `HttpOnly`).
2. `HttpClient`, via `HttpClientXsrfModule` (or `withXsrfConfiguration()` with the functional `provideHttpClient()` API), reads that cookie's value on every outgoing request that isn't a "safe" method (Angular applies this to same-origin requests, skipping GET/HEAD by default is *not* the actual gate — Angular attaches the header to all requests to relative/whitelisted URLs regardless of method, but the server should only *check* it for state-changing methods) and copies it into a request header (default name `X-XSRF-TOKEN`).
3. The server, on receiving the request, compares the cookie value to the header value. Because a cross-site attacker's page can trigger a cookie-carrying request but **cannot read the cookie value** (cross-origin) to also set the matching header, the forged request fails validation.

Angular only sends the header for requests to the app's own origin (or explicitly configured trusted origins) — it deliberately avoids leaking the token to third-party domains.

### 2.7 Template injection

"Template injection" in Angular usually does **not** mean attacker-controlled Angular template syntax (`{{ }}`) gets compiled — Angular templates are compiled ahead of time (AOT) from your source files, not from runtime strings, so there is no direct equivalent of server-side template injection (SSTI) in normal Angular usage. The real risks are:

- Dynamically compiling components/templates from strings at runtime using the JIT compiler with attacker-influenced content (rare, but some CMS-style Angular apps do this).
- `[innerHTML]` bound to attacker content that includes Angular-*looking* syntax is irrelevant — Angular's HTML sanitizer just treats it as text/markup, it does not re-interpret `{{ }}` inside `innerHTML`-bound strings as a live Angular expression. The real danger there is plain HTML/JS injection, handled by the HTML sanitizer.
- Using `eval`, `new Function(...)`, or feeding user strings into `Function`-based expression evaluators (some dynamic-forms/expression libraries do this) — this is where genuine "template/expression injection" risk lives in Angular apps.

### 2.8 Dependency vulnerability scanning

Angular apps pull in hundreds of transitive npm packages. A vulnerable dependency (a `lodash` prototype-pollution bug, a vulnerable `marked`/HTML-parsing library, a compromised package) can undermine everything above. Standard practice:

- `npm audit` / `npm audit fix` in CI, failing the build above a severity threshold.
- `ng update` to stay current with Angular's own security patches.
- Tools like Snyk, Dependabot, Renovate, or GitHub's built-in security advisories for continuous monitoring.
- Special attention to any library that touches `innerHTML`, Markdown rendering, rich text editors, or PDF/SVG parsing — these are disproportionately common sources of DOM XSS in dependencies.

---

## 3. Code Examples

### 3.1 Safe vs. unsafe binding

```typescript
import { Component } from '@angular/core';

@Component({
  selector: 'app-comment',
  standalone: true,
  template: `
    <!-- SAFE: plain interpolation, always rendered as text -->
    <p>{{ userComment }}</p>

    <!-- RISKY: innerHTML goes through the HTML security context.
         Angular sanitizes it automatically — script tags/handlers stripped.
         Still prefer interpolation unless you truly need rendered markup. -->
    <div [innerHTML]="userComment"></div>
  `,
})
export class CommentComponent {
  // Imagine this came from user input, e.g. "<img src=x onerror=alert(1)>"
  userComment = '<img src=x onerror=alert(1)>';
  // Interpolation renders the literal text "<img src=x onerror=alert(1)>"
  // [innerHTML] renders "<img src="x">" — Angular strips the onerror handler.
}
```

### 3.2 `bypassSecurityTrustHtml` — misuse vs. correct usage

```typescript
import { Component, Input } from '@angular/core';
import { DomSanitizer, SafeHtml } from '@angular/platform-browser';

// ── MISUSE: trusting user-controlled content ──────────────────────────────
@Component({
  selector: 'app-bad-preview',
  standalone: true,
  template: `<div [innerHTML]="trustedHtml"></div>`,
})
export class BadPreviewComponent {
  @Input() set rawUserHtml(value: string) {
    // DANGER: this disables Angular's sanitizer for attacker-controlled input.
    // Any <script>, onerror=, or javascript: URL the user typed now executes.
    this.trustedHtml = this.sanitizer.bypassSecurityTrustHtml(value);
  }
  trustedHtml!: SafeHtml;
  constructor(private sanitizer: DomSanitizer) {}
}

// ── CORRECT: bypass only for content you control/author, never user input ─
@Component({
  selector: 'app-good-preview',
  standalone: true,
  template: `<div [innerHTML]="marketingHtml"></div>`,
})
export class GoodPreviewComponent {
  // This HTML is authored by your own CMS team / build pipeline, not end users,
  // and has already been through a server-side sanitizer (e.g., DOMPurify) as
  // defense in depth. Bypassing here is a deliberate, reviewed decision.
  marketingHtml: SafeHtml;

  constructor(private sanitizer: DomSanitizer) {
    const trustedFromCms = '<b>Welcome</b> to our <em>trusted</em> promo banner';
    this.marketingHtml = this.sanitizer.bypassSecurityTrustHtml(trustedFromCms);
  }
}

// ── BETTER STILL: don't bypass at all — let Angular sanitize ───────────────
@Component({
  selector: 'app-best-preview',
  standalone: true,
  template: `<div [innerHTML]="content"></div>`, // no bypass; Angular sanitizes
})
export class BestPreviewComponent {
  @Input() content = ''; // rendered through the HTML sanitizer automatically
}
```

### 3.3 `bypassSecurityTrustResourceUrl` for embeds — with validation

```typescript
import { Component, Input } from '@angular/core';
import { DomSanitizer, SafeResourceUrl } from '@angular/platform-browser';

@Component({
  selector: 'app-video-embed',
  standalone: true,
  template: `<iframe [src]="embedUrl" allow="autoplay" width="560" height="315"></iframe>`,
})
export class VideoEmbedComponent {
  embedUrl!: SafeResourceUrl;

  constructor(private sanitizer: DomSanitizer) {}

  @Input() set videoId(id: string) {
    // Validate BEFORE bypassing — never bypass a URL built from raw user input.
    if (!/^[A-Za-z0-9_-]{11}$/.test(id)) {
      throw new Error('Invalid video id');
    }
    const safeConstructedUrl = `https://www.youtube.com/embed/${id}`;
    this.embedUrl = this.sanitizer.bypassSecurityTrustResourceUrl(safeConstructedUrl);
  }
}
```

### 3.4 Strict CSP with a nonce (`ngCspNonce`)

Server-rendered `index.html` (e.g., from an Express/Node SSR layer or a templated shell), generating a fresh nonce per request:

```typescript
// server.ts (Express-style SSR bootstrap)
import { randomBytes } from 'crypto';

app.get('*', (req, res) => {
  const nonce = randomBytes(16).toString('base64');

  res.setHeader(
    'Content-Security-Policy',
    `default-src 'self'; script-src 'nonce-${nonce}'; style-src 'nonce-${nonce}'; object-src 'none'; base-uri 'self';`
  );

  res.render('index', { nonce }); // template interpolates {{nonce}} below
});
```

```html
<!-- index.html template -->
<!doctype html>
<html>
  <head>
    <script nonce="{{nonce}}" src="runtime.js"></script>
    <script nonce="{{nonce}}" src="main.js"></script>
  </head>
  <body>
    <app-root ngCspNonce="{{nonce}}"></app-root>
  </body>
</html>
```

Angular reads `ngCspNonce` off the host element and stamps the same nonce onto every `<style>` tag it dynamically injects for component view encapsulation, so `style-src 'nonce-...'` doesn't force you to add `'unsafe-inline'`.

### 3.5 XSRF configuration with `HttpClient`

Default cookie/header names (`XSRF-TOKEN` / `X-XSRF-TOKEN`) usually need no configuration — just make sure the server sets that cookie. Custom names, with the modern functional API:

```typescript
// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideHttpClient, withXsrfConfiguration, withInterceptorsFromDi } from '@angular/common/http';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(
      withXsrfConfiguration({
        cookieName: 'CSRF-TOKEN',      // matches your backend's cookie name
        headerName: 'X-CSRF-TOKEN',    // matches your backend's expected header
      }),
      withInterceptorsFromDi(),
    ),
  ],
};
```

Legacy `NgModule`-based equivalent:

```typescript
@NgModule({
  imports: [
    HttpClientModule,
    HttpClientXsrfModule.withOptions({
      cookieName: 'CSRF-TOKEN',
      headerName: 'X-CSRF-TOKEN',
    }),
  ],
})
export class AppModule {}
```

Disabling XSRF handling entirely (only for APIs that use a different CSRF scheme, e.g., pure bearer-token auth with no cookies involved):

```typescript
provideHttpClient(withXsrfConfiguration({ cookieName: '', headerName: '' }));
// or, older API:
HttpClientXsrfModule.disable();
```

---

## 4. Internal Working

### 4.1 How the compiler decides which context applies

Angular ships a static table mapping `[element, property]` pairs (and some ARIA/SVG variants) to a `SecurityContext`, built into `packages/compiler/src/schema/dom_security_schema.ts`. When the Ivy template compiler processes a property binding like `[href]` on an `<a>`, it consults this schema at **compile time** (not runtime) and determines: this binding lands in `SecurityContext.URL`. It then emits an instruction (`ɵɵsanitizeUrl` or similar host-bound sanitization call) that both the JIT and AOT pipelines wire into the generated `update` function for that binding. This is why sanitization is "free" — the compiler pre-selects the sanitizer function per binding once, and it runs on every change-detection pass without further lookups.

For custom attributes/directives where Angular can't infer the destination (e.g., an `@Input()` on your own component that internally uses the value unsafely), no automatic sanitization occurs — the schema only covers native DOM element/attribute bindings, not your own directive's internal handling of an `@Input()`.

### 4.2 How the HTML sanitizer strips dangerous markup

Angular's HTML sanitizer (`packages/platform-browser/src/security/html_sanitizer.ts`) does not use regex string surgery (a classic source of sanitizer bypass bugs). Instead it:

1. Parses the input string into a DOM tree using an **inert** document (an element/document created via `DOMParser` or a detached `<template>`/document fragment) so nothing executes during parsing.
2. Walks the resulting node tree recursively.
3. For each element, checks its tag name against an allow-list (`SANITIZABLE_ELEMENTS` roughly: formatting/structural tags like `a, b, blockquote, br, div, em, h1-h6, hr, i, li, ol, p, pre, span, table, ul, ...`). Disallowed elements (`script`, `style` in strict mode, `iframe`, `object`, `embed`, `svg` in some configurations) are dropped, though in some cases their *children*/text content may be kept depending on the tag.
4. For each attribute on a kept element, checks the attribute name against an allow-list (`href, src, alt, title, class, ...`) and drops event-handler attributes (`on*`) outright — they are never on the allow-list.
5. For attributes that carry URLs (`href`, `src`), additionally runs the URL sanitizer's scheme check.
6. Serializes the cleaned tree back into an HTML string, which is what actually gets assigned via `innerHTML`.

Because this is a real DOM walk rather than string pattern matching, most classic regex-sanitizer-bypass tricks (odd casing, null bytes, malformed tag soup that browsers "fix up" differently than a regex expects) don't apply the same way — the browser's own inert parser normalizes the markup first.

### 4.3 How the URL sanitizer decides "safe"

The URL sanitizer (`url_sanitizer.ts`) uses a regex to validate the URL's scheme against an allow-list: `^(?:(?:https?|mailto|ftp|tel|file|sms):|[^&:/?#]*(?:[/?#]|$))`. Effectively: allowed absolute schemes, or a relative URL with no scheme at all. Anything else — most importantly `javascript:`, `data:` in some contexts, `vbscript:` — fails the match and the sanitizer substitutes `unsafe:<original-value>` as the output string, which browsers do not treat as a navigable/executable URL.

### 4.4 How `HttpClient` reads and attaches the XSRF token

`HttpXsrfInterceptor` (an `HttpInterceptor` registered internally when you enable `withXsrfConfiguration`/`HttpClientXsrfModule`) runs on every outgoing `HttpRequest`:

1. It calls an injected `HttpXsrfTokenExtractor`, whose default browser implementation reads `document.cookie`, parses out the cookie matching the configured `cookieName`, and URL-decodes its value.
2. If a non-empty token is found **and** the request is same-origin (or targets an explicitly allowed origin) **and** the request is not a plain `GET`/`HEAD` in the older gating logic (current implementations attach it broadly for same-origin requests, leaving the *enforcement* decision — which methods actually require the header — to the server), the interceptor clones the request and sets the configured `headerName` to the token value.
3. Because reading `document.cookie` only works for **non-`HttpOnly`** cookies, the XSRF cookie must intentionally be readable by JS — this is safe *for this specific purpose* because the security guarantee comes from same-origin policy (a cross-site attacker page cannot read your cookies to forge the header), not from cookie secrecy.
4. The actual CSRF defense is enforced **server-side**: the backend must independently verify the header value matches the cookie value (or matches a server-side session-bound token) on every state-changing request. Angular's role is purely "faithfully echo the token back as a header" — if the server doesn't check it, none of this does anything.

---

## 5. Edge Cases & Gotchas

- **`bypassSecurityTrust*` is the #1 real-world source of Angular XSS.** The moment any bypassed value's construction path includes user input — even indirectly, e.g., concatenating a user's display name into an "authored" HTML string before bypassing — the bypass becomes an attacker-controlled injection point. Treat every `bypassSecurityTrust*` call as a standing security liability requiring a code-review comment justifying why the input is provably non-attacker-controlled, and revisit it whenever upstream data sources change.
- **Sanitization applies to bindings, not to raw DOM API calls.** `elementRef.nativeElement.innerHTML = userInput`, `renderer2.setProperty(el, 'innerHTML', userInput)` in some Renderer2 usages, jQuery's `.html()`, or any third-party widget that does its own DOM writes all bypass Angular's sanitizer completely, because Angular's Ivy-generated update instructions never run for those code paths. `Renderer2.setProperty` for `innerHTML` specifically *does* still go through Angular's sanitizer as of recent versions when set on a *bound* property, but any direct `nativeElement` manipulation does not.
- **Host bindings and directive `@HostBinding` are not always covered the same way.** The DOM security schema is keyed by literal element+attribute names known to the compiler. A custom directive that internally does something dangerous with an `@Input()` value (e.g., builds a `style` string and assigns it via `renderer.setStyle` in a way that isn't a standard style binding) is entirely outside Angular's automatic sanitization — the responsibility shifts to the directive author.
- **Third-party libraries silently injecting raw HTML.** Rich text editors, chart libraries with custom tooltip renderers, Markdown-to-HTML converters, and PDF viewers frequently accept a "render as HTML" option or do so by default. If you wire user-provided content into these libraries, you inherit whatever sanitization (or lack thereof) that library does — audit them independently; don't assume "it's an Angular component, so it's sanitized."
- **CSP breaking Angular's own JIT-compiled styles/templates.** A strict `script-src`/`style-src` without `'unsafe-inline'` and without a matching nonce will silently fail to apply component styles (or throw console errors) because Angular injects `<style>` tags at runtime for `ViewEncapsulation.Emulated`. Forgetting `ngCspNonce` on the root element (or forgetting to propagate a *fresh* nonce per request rather than a hardcoded one, which defeats the purpose of CSP) is a very common deployment bug once teams turn on strict CSP.
- **`'unsafe-inline'` as a nonce fallback is not actually a fallback.** Browsers that support nonces ignore `'unsafe-inline'` when a nonce or hash is also present in the same directive, per the CSP3 spec — but older browsers that don't understand nonces *do* honor `'unsafe-inline'`. Some teams add both, intentionally, purely as graceful degradation for legacy browsers; know this is deliberate, not an oversight, when you see it in a real config.
- **Interpolation vs. property binding confusion.** `<div title="{{ userInput }}">` and `<div [title]="userInput">` are equivalent and both safe for `title` (not an executable sink) — but `<a href="{{ userInput }}">` written as *attribute interpolation* is still subject to the URL sanitizer, same as `[href]="userInput"`, because Angular desugars attribute interpolation into the equivalent property binding internally. Don't assume interpolation syntax alone means "no sanitization" — it depends on which attribute/property you're interpolating into.
- **Angular Universal / SSR and `document.cookie`.** On the server, there is no browser cookie jar in the same sense; XSRF token extraction needs a server-aware token extractor or must be handled via a transfer-state/cookie-forwarding mechanism, otherwise the first SSR-rendered request may not carry a token correctly.
- **`SafeValue` types don't survive serialization.** If you store a `SafeHtml`/`SafeUrl` object in `localStorage`, a signal, or pass it through `JSON.stringify`, you get an opaque wrapper object, not a string — attempting to rehydrate it later as if it were still "safe" will fail or silently misbehave. Bypass results should be constructed fresh at render time from the trusted source, not cached/serialized.
- **Dependency vulnerabilities defeat everything above.** A vulnerable version of a Markdown parser or DOMPurify itself (yes, sanitizer libraries have had their own bypass CVEs) undermines your own careful `bypassSecurityTrust*` discipline. Automated scanning (`npm audit`, Snyk/Dependabot) needs to run continuously, not just at project kickoff.

---

## 6. Interview Questions & Answers

**Q1. Why don't you need to sanitize a value bound with `{{ }}` interpolation?**
A: `{{ }}` interpolation is always rendered into a text node (`Node.textContent` under the hood), never parsed as markup. A text node cannot contain executable script or trigger handler attributes regardless of its content, so there is no security context to violate. Sanitization only matters for property bindings that write into DOM "sinks" that *are* interpreted as HTML/CSS/URLs — `[innerHTML]`, `[href]`, `[src]`, `[style]`, etc.

**Q2. Name the four Angular security contexts and give one example binding for each.**
A: HTML (`[innerHTML]`), Style (`[style]`/`[style.color]`), URL (`[href]` on `<a>`), Resource URL (`[src]` on `<iframe>`/`<script>`). Each has a distinct sanitizer (or, for Resource URL, no sanitizer — it throws unless explicitly trusted) because the risk profile differs: HTML injection risks arbitrary markup/script, URL injection risks `javascript:` navigation, Resource URL risks loading and executing a wholly attacker-controlled resource with full page privilege.

**Q3. What does `DomSanitizer.bypassSecurityTrustHtml()` actually do internally?**
A: It wraps the raw string in a branded object implementing `SafeHtml` (an internal class carrying a marker so Angular's binding instructions recognize it). When that binding executes, Angular's generated code checks "is this value already a `Safe*` type matching this context?" — if yes, it skips the sanitizer call and writes the raw string directly to the DOM. It does not sanitize, validate, or transform the string in any way; it is purely an assertion by the developer that the value is already safe.
**Interviewer intent:** distinguishes candidates who think `bypassSecurityTrustHtml` does *some* lightweight cleaning from those who understand it is a full opt-out with zero validation — a common misconception that leads to real vulnerabilities.

**Q4. Your team wants to render Markdown-converted HTML from user posts. What's the safe way to do this in Angular?**
A: Prefer *not* bypassing sanitization at all — bind the converted HTML to `[innerHTML]` directly and let Angular's built-in sanitizer strip dangerous markup automatically; this covers the vast majority of real needs (bold, links, lists, etc. survive; `<script>`/handlers are stripped). If the Markdown renderer needs to preserve tags Angular's sanitizer would strip (e.g., custom embeds), run the HTML through a dedicated, actively-maintained sanitizer library like DOMPurify server-side or client-side *before* the value ever reaches Angular, and only then consider whether a bypass is still needed (usually it isn't). Never call `bypassSecurityTrustHtml` directly on Markdown output from user-authored content — that's precisely the scenario the bypass methods are unsafe for.

**Q5. What's the difference between `bypassSecurityTrustUrl` and `bypassSecurityTrustResourceUrl`, and why does it matter which one you use?**
A: `bypassSecurityTrustUrl` marks a value safe for the URL context (navigational `href`/`src` on `<a>`, `<area>` — clicking navigates, but nothing auto-executes). `bypassSecurityTrustResourceUrl` marks a value safe for the Resource URL context (`<iframe src>`, `<script src>`, `<object data>` — the resource loads and **runs** automatically, no user click needed). Using the wrong one either won't compile/won't satisfy the binding's actual required context (Angular checks the wrapper's context tag), or, worse, if a developer reaches for the more permissive `bypassSecurityTrustResourceUrl` "just to make the error go away" on a binding that only needed URL-level trust, they've granted far more risk than necessary.
**Interviewer intent:** checks whether the candidate understands that these wrapper types are context-tagged, not interchangeable "trust me" tokens, and that over-scoping the bypass is a common shortcut that increases blast radius.

**Q6. How does Angular decide which security context applies to a given property binding — is it a runtime check?**
A: No, it's resolved at **compile time**. The Ivy compiler consults a static schema (mapping element+property pairs to a `SecurityContext`) while generating the component's update instructions. It emits the appropriate sanitizer call (e.g., `ɵɵsanitizeUrl`, `ɵɵsanitizeHtml`) baked into the compiled output for that specific binding. At runtime, that pre-selected sanitizer simply runs on every change-detection pass — there's no per-change lookup of "what context is this."

**Q7. What is Angular's strict CSP mode and what problem does `ngCspNonce` solve?**
A: Strict CSP means a `Content-Security-Policy` that avoids `'unsafe-inline'`/`'unsafe-eval'`, instead allowing only nonce- or hash-tagged scripts/styles. The problem: Angular itself injects `<style>` elements at runtime for component view encapsulation (and can JIT-compile templates in some scenarios), and a strict `style-src`/`script-src` would block Angular's own injected content as "inline" and un-nonced. `ngCspNonce="{{nonce}}"` on the root/host element tells Angular which nonce value to stamp onto every style/script tag it generates itself, so the app satisfies a strict CSP without weakening the policy.

**Q8. Explain how Angular's HttpClient participates in CSRF protection. Does it prevent CSRF by itself?**
A: No — Angular alone cannot prevent CSRF; the server must implement and enforce the check. `HttpClient`'s role, via `HttpClientXsrfModule`/`withXsrfConfiguration`, is to read a non-`HttpOnly` cookie (default `XSRF-TOKEN`) set by the server and copy its value into a request header (default `X-XSRF-TOKEN`) on same-origin requests. This implements the client side of the double-submit-cookie pattern. The actual protection comes from the server independently validating that the header matches the cookie/session on state-changing requests — a cross-site attacker can trigger a request carrying the cookie automatically but cannot read the cookie's value cross-origin to also forge the matching header.

**Q9. Why must the XSRF cookie be readable by JavaScript (not `HttpOnly`), and doesn't that weaken security?**
A: `HttpClient`'s browser-based token extractor reads the token via `document.cookie`, which is impossible for `HttpOnly` cookies. This doesn't weaken CSRF protection specifically, because the security guarantee here isn't "keep the token secret from JS" — it's "keep the token unreadable cross-origin," which the browser's same-origin policy already provides regardless of `HttpOnly`. (It would be a problem if this were the *session* cookie itself, which should remain `HttpOnly` to mitigate XSS-driven session theft — the XSRF token is a separate, disposable value specifically designed to be JS-readable.)
**Interviewer intent:** tests whether the candidate conflates XSS defenses (`HttpOnly` on session cookies) with CSRF defenses (JS-readable XSRF token) — a very common confusion.

**Q10. A component uses `ElementRef.nativeElement.innerHTML = someValue` directly instead of `[innerHTML]` binding. Is this still protected by Angular's sanitizer?**
A: No. Angular's sanitization is implemented in the compiled binding instructions the Ivy compiler generates for template bindings — it has no hook into arbitrary imperative DOM manipulation. Writing directly to `nativeElement.innerHTML` (or using `document.write`, jQuery's `.html()`, or similar) completely bypasses Angular's security pipeline. If `someValue` can contain attacker-influenced content, this is a direct DOM XSS vector with none of Angular's protections active — you'd need to call `DomSanitizer.sanitize(SecurityContext.HTML, someValue)` manually before assigning it, or better, refactor to use a template binding instead.

**Q11. What's "template injection" in the Angular context, and is it the same as server-side template injection (SSTI) in something like Jinja2 or Freemarker?**
A: Not quite the same. Angular templates are compiled ahead-of-time from your source `.html`/inline template files — there is no runtime path where untrusted user data becomes literal Angular template syntax (`{{ }}`, structural directives) that then gets compiled and evaluated, the way SSTI works in server template engines that interpolate user input directly into a template string before rendering. The closest real analogues in Angular are: (1) runtime JIT-compiling a component/template built from untrusted strings (uncommon, deliberately avoided in most apps), and (2) dynamic expression evaluators (some forms/rules-engine libraries) that use `new Function()` or `eval`-like mechanisms on user-supplied expression strings — that's genuine code-injection risk, just not "Angular template injection" in the SSTI sense.
**Interviewer intent:** separates candidates who understand Angular's AOT compilation model from those who assume every templating system shares SSTI-style risk; also surfaces whether they know about `eval`-adjacent risk in third-party dynamic-expression libraries.

**Q12. Angular's sanitizer already strips dangerous HTML automatically — why do dependency vulnerability scans (npm audit, Snyk) still matter for security?**
A: Angular's sanitizer only protects value flow through Angular's own binding system. It has no visibility into what a third-party library does internally — a chart library's custom tooltip renderer, a rich-text editor, a Markdown parser, or even a transitive dependency doing prototype pollution can introduce DOM XSS or other vulnerabilities entirely outside Angular's control. Dependency scanning catches known CVEs in these packages (including in sanitizer libraries themselves, which have had their own bypass vulnerabilities), and should run continuously in CI, not just once at project setup, since new CVEs are disclosed against existing, unchanged code constantly.

**Q13. If you must render truly untrusted, attacker-supplied HTML (e.g., a user-submitted rich-text post) with formatting intact, what's the recommended layered approach?**
A: Don't rely on any single layer. (1) Sanitize server-side on write, using a well-maintained library (DOMPurify, or a server-side equivalent) with an explicit allow-list of tags/attributes — never a deny-list. (2) Let Angular's own `[innerHTML]` sanitizer run as a second, independent layer on render — don't bypass it. (3) Serve the page under a CSP that at minimum sets `object-src 'none'` and restricts `script-src` so that even if some markup slips through, inline/injected scripts can't execute. (4) Never call `bypassSecurityTrustHtml` on this content — the whole point is that Angular's own sanitizer stays active as a safety net in case the upstream sanitizer has a bypass bug.

**Q14. What happens if you bind an attacker-controlled string like `"javascript:alert(1)"` to `[href]` without any bypass call?**
A: Angular's URL sanitizer checks the scheme against its allow-list regex (`https?`, `mailto`, `ftp`, `tel`, `file`, `sms`, or scheme-less/relative). `javascript:` fails that check, so the sanitizer replaces the entire value with `unsafe:javascript:alert(1)` as the string written to the `href` attribute. Browsers do not treat a value starting with `unsafe:` as a navigable scheme, so clicking the link is inert — no code executes, and no navigation to the crafted target occurs.

---

## 7. Quick Revision Cheat Sheet

- **`{{ }}` interpolation → always safe** (text node only, no security context to violate).
- **Four contexts**: HTML (`[innerHTML]`), Style (`[style]`), URL (`[href]`/`[src]` on navigational elements), Resource URL (`[src]` on `<iframe>`/`<script>`/`<object>` — no sanitizer, throws unless bypassed).
- **`DomSanitizer.sanitize(context, value)`** — manual sanitize; **`bypassSecurityTrust{Html,Style,Script,Url,ResourceUrl}`** — opt out entirely, only for values you fully control, never for raw user input.
- **HTML sanitizer** = real DOM walk over an inert-parsed tree + tag/attribute allow-lists, not regex string surgery.
- **URL sanitizer** = scheme allow-list (`http(s)`, `mailto`, `ftp`, `tel`, `file`, `sms`, relative); anything else → `unsafe:` prefix, inert.
- **Compiler picks the context at compile time** via a static DOM security schema — zero runtime lookup cost.
- **Bypassing DOM through `nativeElement`, `document.write`, jQuery, or a directive's internal handling of `@Input()` skips Angular's sanitizer entirely** — no protection outside Angular's own binding pipeline.
- **CSP** = independent, browser-enforced second layer; strict mode drops `'unsafe-inline'`/`'unsafe-eval'` in favor of nonces; use `ngCspNonce` on the root element so Angular's own injected `<style>` tags carry the nonce.
- **XSRF/CSRF**: `HttpClient` reads a JS-readable cookie (`XSRF-TOKEN` default) and echoes it as a header (`X-XSRF-TOKEN` default) on same-origin requests — configured via `withXsrfConfiguration()`/`HttpClientXsrfModule`. Angular only *transmits* the token; the **server must verify** it. This is unrelated to (and doesn't replace) `HttpOnly` protection on session cookies.
- **"Template injection" in Angular ≠ SSTI** — templates are AOT-compiled from source, not built from runtime user strings; real analogous risk lives in `eval`/`new Function`-based dynamic expression evaluators and runtime JIT-compiling of untrusted template strings.
- **Dependency scanning (`npm audit`, Snyk, Dependabot) is mandatory, continuous hygiene** — Angular's sanitizer can't protect against vulnerabilities inside third-party libraries that manipulate the DOM themselves.
- **Layered defense for untrusted rich content**: server-side allow-list sanitizer (e.g., DOMPurify) → Angular's own `[innerHTML]` sanitizer left active (no bypass) → restrictive CSP as a backstop.

**Created By - Durgesh Singh**
