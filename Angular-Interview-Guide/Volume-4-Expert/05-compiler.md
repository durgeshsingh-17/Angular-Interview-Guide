# Chapter 47: The Angular Compiler

## 1. Overview

Every other chapter in this book that touches AOT, JIT, or Ivy has been describing the *outputs* and *consequences* of compilation — smaller bundles, tree-shakable instructions, faster change detection. This chapter opens the box.

Angular's compiler, `ngtsc` ("Angular TypeScript Compiler"), is not a separate parser that reads your `.ts` files independently of TypeScript. It is a **TypeScript compiler plugin** — a set of custom `ts.TransformerFactory` hooks and a `ts.CompilerHost`/`ts.Program` wrapper — that rides on top of the real TypeScript compiler (`tsc`). It uses TypeScript's own type checker to read decorator metadata statically (no `reflect-metadata`, no runtime decorator execution), parses your HTML templates into a dedicated Angular Render3 AST (the "R3 AST"), lowers that AST into a sequence of `ɵɵinstruction()` calls backed by a **constant pool**, and finally hands a fully valid TypeScript AST back to `tsc` for ordinary emit to JavaScript.

The historical predecessor, `ngc` (the "old" View Engine compiler), worked differently: it used `.metadata.json` side files and static reflection over decorator arguments, generated `.ngfactory.ts`/`.ngsummary.ts` files as a *separate* compilation pass after `tsc` finished, and produced verbose factory code. `ngtsc`, introduced with Ivy, integrates directly into the `tsc` `Program` lifecycle as first-class transforms, enabling incremental, locality-based compilation and making the Angular Language Service's live type-checking of templates possible.

This chapter walks the pipeline end to end:

```
.ts source + .html template
        │
        ▼
 TypeScript Program creation (ts.createProgram / ngtsc wrapper)
        │
        ▼
 Decorator analysis via TypeChecker (no reflection)
        │
        ▼
 Template parsing → HTML AST → R3 AST (Angular Render3 AST)
        │
        ▼
 Binding parser (expressions, interpolations, pipes)
        │
        ▼
 Template type-checking (TCB generation, for `strictTemplates`)
        │
        ▼
 Instruction selection + Constant Pool construction
        │
        ▼
 TS AST synthesis (ɵɵdefineComponent, ɵɵtemplate, etc.)
        │
        ▼
 ts.Transformer injects synthesized AST into the Program
        │
        ▼
 Standard TypeScript emit → .js + .d.ts
```

## 2. Core Concepts

### 2.1 ngtsc vs. ngc vs. View Engine — why the architecture changed

**View Engine's `ngc`** ran as a wrapper *around* `tsc`:
1. Run `tsc` to get `.js`/`.d.ts` and `.metadata.json` (a JSON dump of decorator data, produced by a separate "metadata collector" that statically evaluated decorator expressions into JSON).
2. Run a second pass, `@angular/compiler-cli`, which read the `.metadata.json` files for the whole program, resolved `NgModule` graphs, and code-generated `.ngfactory.ts` files (component factories, `ViewDefinition` trees) as *new source files*.
3. Compile those factory files with `tsc` again.

This was slow (two/three compiler passes), fragile (metadata JSON couldn't represent arbitrary expressions), and produced non-tree-shakable code (every component pulled in the entire `ComponentFactory` machinery, and `NgModule`-scoped compilation meant a component's output depended on which module declared it).

**Ivy's `ngtsc`** collapses this into a *single* `tsc` invocation. It works by supplying TypeScript with a custom `CompilerHost` and injecting custom `ts.TransformerFactory<ts.SourceFile>` functions into the emit pipeline. Decorators are read via `ts.TypeChecker` symbol resolution (see §2.3), templates are compiled to per-component instruction streams (locality principle — a component's output depends only on itself and its own imports, not the module that declares it, which is what makes standalone components and tree-shaking work), and the generated code is spliced directly into the AST that TypeScript is about to emit — no intermediate `.ngfactory.ts` files touch disk.

### 2.2 The Program wrapper (`NgtscProgram`)

`@angular/compiler-cli` exposes `NgtscProgram`, which wraps a `ts.Program`. Key responsibilities:

- **`CompilerHost` augmentation**: intercepts `getSourceFile` to attach parsed template/style resources to component source files, and resolves `templateUrl`/`styleUrls` to actual file reads (so the Language Service and incremental builds can track them as dependencies).
- **Two logical passes over the `ts.Program`**:
  1. **Analysis phase** — walk every source file, find `@Component`/`@Directive`/`@Pipe`/`@NgModule`/`@Injectable` decorated classes (via the `DecoratorHandler` abstraction), extract metadata, parse templates into R3 AST, and run template type-checking.
  2. **Transform/emit phase** — for each decorated class, synthesize the `ɵɵdefineComponent(...)`/`ɵcmp` static property (and similarly `ɵdir`, `ɵpipe`, `ɵmod`, `ɵinj`, `ɵfac`) as real `ts.PropertyDeclaration` nodes, and hand them to a `ts.TransformerFactory` that TypeScript merges into the class declaration during `program.emit()`.

Each decorator type has its own `DecoratorHandler<D, A, R>` (detect → analyze → resolve → compile), e.g. `ComponentDecoratorHandler`, `DirectiveDecoratorHandler`, `NgModuleDecoratorHandler`, `PipeDecoratorHandler`, `InjectableDecoratorHandler`. This handler pattern is the extensibility seam — each handler is independently responsible for its slice of metadata and its own emitted instructions.

### 2.3 Reading decorators via the TypeChecker, not reflection

This is one of the most interview-relevant internals. `ngtsc` never executes your code. There is no `Reflect.getMetadata`, no running the `@Component({...})` call at compile time. Instead:

1. **Symbol resolution**: For each class declaration, `ngtsc` asks the TS `TypeChecker` for the class symbol, walks its `decorators` (or, for the new ECMAScript decorators/`experimentalDecorators` off, the equivalent AST shape), and identifies calls that resolve — via the checker's import/alias resolution — to `Component`, `Directive`, `NgModule`, etc. from `@angular/core`.
2. **Static expression evaluation (`PartialEvaluator`)**: The object literal passed to `@Component({...})` is evaluated by a purpose-built interpreter called the `PartialEvaluator` (in `packages/compiler-cli/src/ngtsc/partial_evaluator`). It walks the TS AST of the decorator argument and resolves values *statically*:
   - Literals (`'app-foo'`, `42`, `true`) resolve directly.
   - References to `const` variables/enums are traced to their declaration and resolved if that declaration is itself statically analyzable.
   - Object spreads, array literals, and simple template strings are supported.
   - Imported symbols are traced across module boundaries (following `.d.ts` re-exports) as far as static analysis allows.
   - Function calls are only resolvable if the function is a recognized "known function" the evaluator special-cases (e.g. simple built-ins) — arbitrary function calls generally produce a `DynamicValue` (see §5).
3. **Result**: a `ComponentMetadata`/`DirectiveMetadata` object entirely made of resolved TS types/values — this is what feeds the R3 AST template compiler and instruction emitter.

This static-evaluation strategy is *why* Angular can tree-shake, why `ng-packagr`/Ivy don't need `reflect-metadata` polyfills at runtime for compiled code, and why the compiler can catch configuration errors (e.g., non-analyzable `providers`) at build time rather than at runtime.

### 2.4 Template parsing: HTML AST → R3 AST

Template compilation itself is a mini-pipeline inside the bigger one:

1. **HTML lexing/parsing** (`packages/compiler/src/ml_parser`): the template string is tokenized and parsed into a generic HTML AST (`Element`, `Text`, `Comment`, `Attribute` nodes) — essentially an HTML5-ish parser, tolerant of Angular-specific syntax markers (`*ngIf`, `[prop]`, `(event)`, `#ref`, `{{ }}`).
2. **Template AST transform** (`packages/compiler/src/render3/r3_template_transform.ts`): the HTML AST is walked and converted into the **R3 AST** — Angular's own node types tailored to Ivy codegen: `Template` (structural-directive-bearing `<ng-template>`/desugared `*ngIf`), `Element`, `BoundText`, `BoundAttribute`, `BoundEvent`, `Reference`, `Variable`, `Content` (`<ng-content>`), `Icu` (i18n), `DeferredBlock`/`IfBlock`/`ForLoopBlock` (control flow syntax). Structural directives (`*ngIf="cond"`) are desugared here into nested `Template` nodes wrapping the element.
3. **Binding parsing** (`packages/compiler/src/expression_parser`): every binding string (`{{ user.name }}`, `[disabled]="isDisabled()"`, `(click)="save($event)"`) is handed to the **`Parser`/`Lexer`** for Angular expression syntax — a restricted expression grammar (no arbitrary JS: no `new`, no bitwise ops historically limited, no assignment inside interpolation, safe-navigation `?.` and pipes `|` are Angular-specific extensions). This produces `AST` expression nodes (`PropertyRead`, `MethodCall`, `Interpolation`, `BindingPipe`, `Conditional`, `KeyedRead`, etc.) — a *separate* small AST from both the HTML AST and the TS AST.
4. Each binding is also associated with a **security context** (`SecurityContext.HTML`, `.URL`, `.STYLE`, etc.) determined from the element/attribute name via `DomElementSchemaRegistry`, which is what lets the runtime auto-sanitize.

### 2.5 The Constant Pool

Ivy instructions frequently need "constant" data: attribute arrays (`['class', 'foo', 'id', 'bar']`), i18n message parts, `pureFunction` closures for object/array literals in templates, and the `consts` array of an element's static attributes/local refs. Rather than inlining these as fresh literals at every instruction call site (which would bloat and de-duplicate poorly), the compiler routes them through a **`ConstantPool`** (`packages/compiler/src/constant_pool.ts`).

The `ConstantPool` is a per-component (in newer versions, may hoist further) deduplicating cache: when the emitter needs a constant expression, it calls `constantPool.getConstLiteral(expr)` (or `getSharedConstant`, `.populate()`), which:
- Checks whether an *equivalent* expression already exists in the pool (structural equality on the generated `Expression` tree).
- If yes, returns a reference to the already-allocated variable/array-slot.
- If no, allocates a new statement in the pool (e.g. `const _c0 = ["class", "title"];`) and returns a reference to it.

At final emit, the pool's statements are prepended before the component's class declaration, and the `ɵɵdefineComponent()` call as well as per-element instructions (`ɵɵelementStart(0, "div", _c0)`) reference the pooled constants by index/identifier instead of re-literalizing them. This is why compiled output shows numbered `_c0`, `_c1`... arrays at the top of a component's generated JS — that's the constant pool made visible.

### 2.6 Instruction selection and emission

Once the R3 AST + resolved metadata exist, the **template compiler** (`packages/compiler/src/render3/view/template.ts`) walks the R3 AST and emits a **creation block** and an **update block** of `ɵɵ`-prefixed instruction calls:

- **Creation mode instructions** run once per view instance: `ɵɵelementStart`/`ɵɵelementEnd`, `ɵɵtext`, `ɵɵlistener`, `ɵɵtemplate` (for embedded views / structural directives / `@if`/`@for`), `ɵɵprojection`, `ɵɵi18nStart`.
- **Update mode instructions** run on every change-detection pass and are guarded by the `ɵɵadvance(n)` instruction, which moves the internal `LView` cursor forward by `n` nodes before applying bindings: `ɵɵproperty`, `ɵɵattribute`, `ɵɵclassProp`, `ɵɵstyleProp`, `ɵɵtextInterpolate1..8`/`ɵɵtextInterpolateV`, `ɵɵpipeBind1..4`/`ɵɵpipeBindV`.

Instruction *selection* is largely mechanical/table-driven: element vs. `ng-template` vs. text node vs. i18n block each map to a fixed instruction family; the compiler picks the arity-specialized variant (`ɵɵtextInterpolate1` vs `...2` vs `...V`) based on how many expression "holes" the interpolation has, because specialized arities avoid allocating an array at runtime. Pure/constant sub-expressions (object or array literals with no changing parts, e.g. `[config]="{active: true}"` where nothing varies) get hoisted through `ɵɵpureFunction0..8` wrapped closures registered in the constant pool, so they are computed once and memoized rather than reallocated every CD cycle.

The output of this stage is not text — it's a tree of `Statement`/`Expression` objects from `packages/compiler/src/output/output_ast.ts`, an output-AST abstraction that both the JS emitter (for calling from `tsc`) and (historically) other emitters could consume.

### 2.7 From output-AST to real TypeScript AST — the transformer hookup

`ngtsc`'s final step converts the compiler's own `output_ast` nodes into genuine `ts.Node`s (via `packages/compiler-cli/src/ngtsc/translator`), then a `ts.TransformerFactory<ts.SourceFile>` (registered as a `before` custom transformer passed to `program.emit(..., customTransformers)`) locates each decorated class declaration in the real `ts.SourceFile` and appends the synthesized static properties (`static ɵcmp = ɵɵdefineComponent({...})`, `static ɵfac = function ...`) directly onto the `ts.ClassDeclaration` node, and (depending on target) removes the original `@Component` decorator node so it doesn't survive into emitted JS (in non-experimental-decorator/ADAT mode) or leaves runtime-safe residue behind. TypeScript then emits this mutated AST exactly as it would emit any other program — Angular never touches `.js` text directly; it only ever manipulates TypeScript ASTs and lets `tsc`'s printer do the final stringification.

### 2.8 Template type-checking (the TCB) and `strictTemplates`

With `strictTemplates: true`, `ngtsc` additionally synthesizes, for every component, a synthetic **Type Check Block (TCB)** — a fictitious TypeScript function body that mirrors the template's data flow (e.g. `*ngIf="user"` becomes an `if (this.user) { ... }` guard; `{{ user.name }}` becomes a property access `this.user.name` typed against the component class). This TCB is a real (though never emitted) piece of TypeScript source, fed back through the *same* `ts.TypeChecker` used for the rest of the program. Type errors surfaced by the checker on the TCB are then mapped back, via source-span tracking, to the original template file/line/column and reported as template diagnostics. This is why template type errors (e.g. binding to a nonexistent property, wrong pipe argument types) show up with accurate template-relative line numbers even though, mechanically, they were caught by ordinary TS type-checking of synthetic code.

### 2.9 Incremental compilation

`ngtsc` supports incremental rebuilds (used by the CLI's watch mode and by IDE tooling) via a **dependency graph keyed by file** plus **semantic diffing**:
- On a file change, `ngtsc` doesn't re-analyze the whole program. It uses `ts.Program`'s own incremental "old program" reuse (`ts.createProgram({ oldProgram })`) to get unchanged `SourceFile` objects for free, and layers an Angular-specific `IncrementalDriver`/dependency-graph on top that tracks, per component, which template/style/decorator inputs it depends on.
- Changed files are re-analyzed; the **semantic reuse** logic asks "did the *public* API-relevant metadata of this file change?" (e.g. a component's selector, inputs/outputs list, exported symbols) rather than "did the file's bytes change?" — if only a method body changed and the component's public shape is unaffected, dependents (other components importing/using it) don't need re-analysis at all. This is what keeps `ng build --watch` fast on large repos.
- Locality: because Ivy compiles each component in isolation from its consumers (unlike View Engine's whole-module compilation), invalidation blast radius is naturally small — changing a component's template only forces re-emit of that component's own file, not every module that imports it.

### 2.10 The Angular Language Service

The Language Service (used by the VS Code extension, WebStorm, etc.) is effectively **`ngtsc` running inside an LSP host instead of a batch CLI**. It wraps a `ts.LanguageService` and layers Angular's own `LanguageService` class (`packages/language-service`) on top:

- It reuses the *exact same* `ngtsc` program construction, `PartialEvaluator`, template parser, and TCB generation used for full builds — this is deliberate: diagnostics, hovers, and autocomplete inside a template are backed by the same type-checking machinery that would run at build time, so "no red squiggly in the editor" reliably predicts "no compile error."
- For autocomplete inside `{{ user.| }}`, the Language Service locates the corresponding position in the synthetic TCB, asks the TS `LanguageService` for completions at that mapped TCB position, and maps the resulting completion list (with types) back to template coordinates.
- It runs incrementally per-keystroke by reusing the incremental `ts.Program`/`IncrementalDriver` machinery described above, re-analyzing only the single component whose template is being edited.

## 3. Code Examples

```typescript
// component.ts
@Component({
  selector: 'app-greeting',
  template: `
    <div class="card" *ngIf="user as u">
      Hello, {{ u.name }}!
      <button (click)="greet(u)">Greet</button>
    </div>
  `,
})
export class GreetingComponent {
  @Input() user?: { name: string };
  greet(u: { name: string }) { console.log('hi', u.name); }
}
```

**Sketch of the parsed R3 AST** (simplified — the real nodes carry source spans on every field):

```text
R3 AST (root nodes for the template):
[
  Template {                              // desugared from *ngIf
    templateAttrs: [BoundAttribute(ngIf, PropertyRead(user))],
    variables: [Variable(name: 'u', value: 'ngIf')],   // "as u" -> context var
    children: [
      Element {
        name: 'div',
        attributes: [TextAttribute('class', 'card')],
        children: [
          BoundText { value: Interpolation(['Hello, ', PropertyRead('u','name'), '!']) },
          Element {
            name: 'button',
            outputs: [BoundEvent('click', MethodCall('greet', [PropertyRead('u')]))],
            children: [ Text('Greet') ]
          }
        ]
      }
    ]
  }
]
```

**Sketch of the emitted instructions** (creation + update blocks, constant-pool-backed):

```javascript
// Constant pool (hoisted, deduplicated, module scope of the component file)
const _c0 = ["class", "card"];

// GreetingComponent.ɵcmp = ɵɵdefineComponent({
//   ...
//   decls: 1, vars: 1,
//   template: function GreetingComponent_Template(rf, ctx) {
    if (rf & 1) {                                   // CREATION block
      ɵɵtemplate(0, GreetingComponent_div_0_Template, 3, 2, "div", _c0);
    }
    if (rf & 2) {                                   // UPDATE block
      ɵɵproperty("ngIf", ctx.user);
    }
//   }
// });

// Embedded view generated for the *ngIf template (a nested template function):
function GreetingComponent_div_0_Template(rf, ctx) {
  const restoredCtx = ɵɵgetCurrentView();
  if (rf & 1) {
    ɵɵelementStart(0, "div", _c0);
     ɵɵtext(1);
     ɵɵelementStart(2, "button");
     ɵɵlistener("click", function GreetingComponent_div_0_Template_button_click_2_listener() {
       ɵɵrestoreView(restoredCtx);
       const u_r1 = ctx.ngIf;               // "as u" -> read the structural-directive context
       const outerCtx = ɵɵnextContext();
       return ɵɵresetView(outerCtx.greet(u_r1));
     });
     ɵɵtext(3, "Greet");
     ɵɵelementEnd();
    ɵɵelementEnd();
  }
  if (rf & 2) {
    const u_r1 = ctx.ngIf;
    ɵɵadvance(1);
    ɵɵtextInterpolate1("Hello, ", u_r1.name, "!");
  }
}
```

Note how the R3 AST's `Variable(u)` (from `as u`) becomes `const u_r1 = ctx.ngIf` reads inside the embedded template function, how the single static attribute pair `class="card"` was hoisted into the deduplicated `_c0` constant-pool entry and referenced by both the `ɵɵtemplate` call and the nested `ɵɵelementStart`, and how the single interpolation hole picks the specialized `ɵɵtextInterpolate1` instruction rather than a generic N-ary variant.

## 4. Internal Working

### 4.1 ngtsc as a custom TypeScript transformer — the hookup mechanics

`@angular/compiler-cli` does not fork or monkey-patch `tsc`; it uses TypeScript's fully public extensibility surface:

1. It calls `ts.createProgram(rootNames, options, customCompilerHost)` where `customCompilerHost` overrides `getSourceFile` to inject template/style dependency tracking and resource resolution.
2. It calls `program.emit(targetSourceFile, writeFile, cancellationToken, emitOnlyDtsFiles, customTransformers)`, where `customTransformers.before` includes Angular's transform factory. TypeScript guarantees `before` transformers run on the *checked, type-resolved* AST before printing, which is exactly the point in the pipeline where a fully resolved `ts.ClassDeclaration` (decorators intact, types resolved) is available for Angular to read and rewrite.
3. Angular's transformer visits each `ts.SourceFile`, finds classes previously identified (during the analysis phase) as Angular declarations, and returns a modified `ts.ClassDeclaration` with new static members spliced in and the original decorator stripped (this stripping is what makes it safe to ship compiled output without `reflect-metadata` at runtime — nothing at runtime ever looks at `@Component` again; the class carries `ɵcmp` directly).
4. A parallel `.d.ts` transform emits **Ivy-specific typing metadata** (`ɵɵComponentDeclaration<...>`, `ɵɵNgModuleDeclaration<...>` etc.) into the `.d.ts` files so that downstream consumers (other packages depending on a compiled library) get enough static type information to re-derive the component's selector/inputs/outputs shape without re-parsing its original template — this is the mechanism behind Ivy's "partial compilation" for published libraries (`ngcc`-free consumption).

### 4.2 Metadata extraction without reflection — the full chain

To make §2.3 concrete, the actual call chain for `@Component({selector: 'app-x', inputs: MY_INPUTS})` is:

1. `TypeScriptReflectionHost.getDecoratorsOfDeclaration(classDecl)` — uses `ts.getDecorators()`/`ts.canHaveDecorators()` (or the legacy `ts.Node.decorators` property pre-TS5) to get raw decorator nodes, then filters to ones whose call expression resolves (via `checker.getSymbolAtLocation` + `checker.getAliasedSymbol`) to `@angular/core`'s `Component`.
2. `ComponentDecoratorHandler.detect()` flags the class as a candidate.
3. `ComponentDecoratorHandler.analyze()` calls into `PartialEvaluator.evaluate(argumentExpression)`. Internally this is a recursive-descent interpreter over the TS `Expression` AST node types (`ObjectLiteralExpression`, `ArrayLiteralExpression`, `Identifier`, `PropertyAccessExpression`, `CallExpression`, `ConditionalExpression`, etc.), each handled by a dedicated case that either produces a concrete JS-like value (`ResolvedValue`: string/number/boolean/array/Map-like object/`Reference` to a declaration) or a `DynamicValue` sentinel carrying diagnostic info about *why* it couldn't be resolved (unsupported syntax, external unresolvable import, etc.).
4. For `inputs: MY_INPUTS`, the evaluator resolves `MY_INPUTS` by looking up its `const` declaration (via the checker) elsewhere in the file/module, and — if that declaration is itself a statically-analyzable literal — inlines its resolved value. If `MY_INPUTS` were instead the return value of a function call, evaluation would bottom out in `DynamicValue.fromDynamicInput(...)` and the compiler would emit a "value not statically analyzable" diagnostic pointing at that expression's span.
5. The fully resolved metadata object is packaged into a `R3ComponentMetadata` and handed to `compileComponentFromMetadata()` in `packages/compiler`, which is the same template-compiling entry point used regardless of whether metadata originally came from a decorator (AOT) or a runtime object (JIT `Component.decorators` in the rare/legacy JIT path) — this shared core is why AOT and JIT semantics for template compilation stay unified.

### 4.3 Incremental rebuild strategy in depth

Two layers of incrementality compose:

- **TypeScript-level reuse**: `ts.createProgram` accepts `oldProgram`, reusing unchanged `SourceFile` objects and their prior binder/checker results where safe. This alone avoids re-parsing/re-binding files whose text didn't change.
- **Angular-level semantic diffing** (`packages/compiler-cli/src/ngtsc/incremental`): maintains a per-file `DependencyGraph` (which files' Angular metadata a given file's compiled output depends on — e.g. Component A's compiled template depends on Component B's *public* metadata because B is used in A's template). On rebuild:
  - Files with changed *text* are re-analyzed.
  - The result is compared to the previous analysis via a **semantic-equality check on the resolved metadata** (selector, inputs, outputs, template AST shape, exported type signature) — not a byte comparison of generated code.
  - If semantically unchanged, dependents are *not* invalidated, short-circuiting a potentially large re-analysis cascade. If changed, invalidation propagates one level along the dependency graph and repeats.
  - This is what lets a developer edit a single component's method body in a 2,000-component app and get a sub-second incremental rebuild instead of a full-program re-analysis.

### 4.4 Locality and partial compilation

Because each component's instruction stream only references its *own* decorator metadata, its *own* template, and statically-resolvable imports (not "whichever NgModule declares it," as View Engine required), `ngtsc` can compile a component fully without seeing the rest of the application — this is the **locality principle**. It underpins:
- **Standalone components** (no `NgModule` needed to resolve the compilation scope at all).
- **Partial/library compilation mode** (`compilationMode: 'partial'`), used when building publishable Angular libraries: instead of fully lowering to `ɵɵdefineComponent(...)` calls tied to a specific Ivy runtime instruction set version, the compiler emits a version-tolerant, linkable IR (`ɵɵngDeclareComponent(...)`) that a downstream consumer's own build re-links against *its* installed Angular runtime version — solving the old "libraries must be compiled with the exact same Angular version as the app" problem.

## 5. Edge Cases & Gotchas

- **"Value at position X is not statically analyzable"** — the single most common ngtsc build error for library authors. Triggered whenever the `PartialEvaluator` hits a `CallExpression` it doesn't specifically recognize inside decorator metadata that must be statically known, e.g.:
  ```typescript
  @Component({ selector: buildSelector('foo'), template: '...' })  // ERROR: function call
  ```
  Fix: replace with a literal, or a `const` whose initializer is itself statically analyzable (`const SEL = 'app-foo'; @Component({selector: SEL, ...})` works fine — const-tracing across files is supported).

- **Barrel-exported re-assignment chains break tracing.** `export const X = someLib.helper()` used as a decorator argument elsewhere fails even if `X`'s *runtime* value would resolve fine — the evaluator has no runtime to fall back on.

- **`providers`/`imports` arrays built with spread of a function's return value** (`providers: [...getSharedProviders()]`) hit the same wall — spreads of a `DynamicValue` propagate the "not statically analyzable" status to the whole array, not just that element.

- **Enum-based selectors/tokens** generally *do* resolve (enums are const-foldable), but `const enum` re-exported across a project reference boundary can fail to resolve under `isolatedModules`, a subtlety that trips up monorepos.

- **Circular imports between two decorated classes** (`A` imports `B`'s type for a `providers` array, `B` imports `A`) can cause the `PartialEvaluator` to hit resolution recursion; ngtsc detects and reports this rather than looping forever, but the fix (use `forwardRef(() => B)`) is still required in the same places it always was, because `forwardRef` exists precisely to defer evaluation past the static-analysis phase to runtime.

- **Template parser vs. TS parser divide is a common confusion source**: an expression like `{{ user?.name ?? 'anon' }}` is parsed by the *Angular expression Lexer/Parser*, not the TypeScript parser — some newer TS syntax lands in Angular template expressions late (or with different semantics) because it needs to be independently implemented in `packages/compiler/src/expression_parser`. Assuming "if it's valid TS, it's valid in a template" is a frequent wrong assumption (e.g., historically `??`/optional chaining needed explicit Angular support before they worked in templates).

- **`strictTemplates` TCB diagnostics can misattribute column spans** for deeply nested generic inference failures — the mapping from synthetic TCB position back to template source span is span-tracked but not always pixel-perfect for multi-line interpolations, an acknowledged rough edge.

- **Incremental rebuild "false negative" risk**: if a change alters *runtime* behavior without altering statically-resolved metadata shape (e.g., a method body's logic changes but its signature doesn't), Angular-level incrementality correctly skips re-analyzing dependents (correct — dependents don't need new *template* compilation), but this can surprise developers who expect "I changed a file" to always mean "everything downstream got rebuilt" — TypeScript's own type-check of callers still happens via the normal `tsc` incremental path, so this is safe, just non-obvious.

- **`@Component({ template: someVariable })` where the variable is a template literal with interpolated expressions** (`` `<div>${dynamicPart}</div>` ``) is not statically analyzable as a template string with substitutions — the compiler needs a literal (or simple concatenation of literals) it can parse as HTML at compile time; a truly dynamic template string cannot be AOT-compiled by definition (this is a design constraint, not a bug — templates must be compile-time-known text).

## 6. Interview Questions & Answers

**Q1. What is `ngtsc`, and how does it differ from the old `ngc`/View Engine compiler at an architectural level?**
`ngtsc` is Angular's Ivy compiler, implemented as a set of custom TypeScript transformers and a wrapped `ts.Program`/`CompilerHost` that plug directly into a single `tsc` compilation. It reads decorator metadata via the TypeScript type checker, compiles each component's template in isolation (locality), and injects the generated `ɵɵdefineComponent` static members directly into the class's AST before TypeScript emits JavaScript. The old `ngc` ran `tsc` to produce `.metadata.json` side files, then ran a *second*, separate compilation pass that code-generated whole new `.ngfactory.ts` source files (module-scoped, not component-local), which were then compiled by `tsc` again. ngtsc is single-pass, per-component, and tree-shakable; ngc was multi-pass, per-module, and not tree-shakable.

**Q2. How does the compiler read `@Component` decorator metadata without using `reflect-metadata` at compile time?**
*Interviewer intent: checking whether the candidate understands that Angular's compile-time metadata extraction is a static-analysis problem, not a runtime reflection problem — a very common point of confusion since Angular's *older* JIT path did rely on reflection-like mechanisms.*
It uses the TypeScript `TypeChecker` to resolve which decorators on a class are `@angular/core` decorators (by resolving imports/aliases to their true declaration), then runs the decorator's argument object literal through a `PartialEvaluator` — a recursive-descent interpreter over the TypeScript AST that statically resolves literals, `const` references (tracing through import chains), object/array literals, and simple expressions into concrete values, all without ever executing the code. Values that can't be resolved this way (arbitrary function calls, values from external non-analyzable sources) become a `DynamicValue`, and if that value is required to be static (e.g. a component's `selector`), the compiler reports an error at that AST node's span.

**Q3. What is the R3 AST, and how does it relate to the HTML AST and the TypeScript AST?**
The R3 AST ("Render3 AST") is Angular's own intermediate representation for a parsed template, distinct from both the generic HTML AST (produced first, by `ml_parser`, and made of generic `Element`/`Text`/`Attribute` nodes) and the TypeScript AST (the class/module structure the component lives in). The HTML AST → R3 AST transform desugars Angular-specific syntax — structural directives (`*ngIf`) become nested `Template` nodes, bindings (`[prop]`, `(event)`, `{{ }}`) become `BoundAttribute`/`BoundEvent`/`BoundText` nodes carrying parsed expression ASTs, and control-flow blocks (`@if`/`@for`) become dedicated block nodes. The R3 AST is what the instruction-emission stage walks to produce `ɵɵ` instruction calls; it never touches the TS AST until the final synthesized static class members are spliced back in.

**Q4. Explain the binding/expression parser: why doesn't Angular just reuse the TypeScript expression parser for template bindings?**
Angular template expressions (`{{ user.name | uppercase }}`, `[disabled]="!isValid"`) use a deliberately restricted grammar with Angular-only extensions (the pipe operator `|`, safe navigation `?.` historically introduced ahead of TS, `$event`/`$implicit` as compiler-injected pseudo-identifiers) and deliberately *exclude* constructs that would be unsafe or meaningless in a declarative binding (assignment expressions in most contexts, `new`, arbitrary statements). Because this grammar is neither a strict subset nor superset of TypeScript expression grammar, Angular ships its own `Lexer`/`Parser` (`packages/compiler/src/expression_parser`) producing a distinct small expression AST (`PropertyRead`, `MethodCall`, `BindingPipe`, `Conditional`, `KeyedRead`, etc.), reused identically by both the pre-Ivy and Ivy template compilers.

**Q5. What is the constant pool, and what problem does it solve?**
*Interviewer intent: tests whether the candidate has looked at actual generated Ivy output (the `_c0`, `_c1` arrays) and understands it's a deliberate dedup mechanism, not compiler noise.*
The `ConstantPool` is a per-file/per-component deduplicating store for constant expressions the template compiler needs repeatedly — static attribute arrays, i18n message metadata, and the closures behind `ɵɵpureFunction*` for constant object/array literal bindings. When the emitter needs such a constant, it asks the pool for it; the pool checks for structural equality against already-allocated constants and either reuses the existing one or allocates a new module-scope statement (visible in output as `const _c0 = [...]`). This avoids re-literalizing identical arrays/objects at every instruction call site (bloat) and, for `pureFunction`-wrapped literals, ensures the literal is memoized rather than recreated on every change-detection pass (which would otherwise break `OnPush`/identity-sensitive bindings downstream).

**Q6. Walk through what happens, instruction-wise, for `<div [class.active]="isActive">{{ label }}</div>`.**
Creation block: `ɵɵelementStart(0, 'div')` opens the element (with any static attributes pulled from the constant pool), `ɵɵtext(1)` reserves a text node slot, `ɵɵelementEnd()` closes it. Update block, gated behind `ɵɵadvance(n)` calls that move the `LView` cursor: `ɵɵclassProp('active', ctx.isActive)` applies the conditional class binding by toggling the class via the renderer only when the boolean value changes since last check, and `ɵɵtextInterpolate1('', ctx.label, '')` (the single-hole specialized interpolation instruction) updates the text node's content. Both update instructions are preceded by change-detection's dirty check baked into their own implementation (they no-op if the bound value is `===` to the previously stored value in `LView`).

**Q7. What is a Type Check Block (TCB), and how does `strictTemplates` use it?**
A TCB is a synthetic, never-emitted TypeScript function body that ngtsc generates per component to make the *template's* data flow visible to the normal TypeScript type checker. Each template construct is translated to an equivalent TS statement using the component class's actual types — `*ngIf="user"` becomes an `if (this.user) {...}` narrowing guard, `{{ user.name }}` becomes `this.user!.name` (or without the assertion, depending on narrowing), event handler calls become real method-call expressions checked against parameter types. This synthetic source is fed to the same `ts.TypeChecker` as everything else in the program; any type errors are captured and their positions mapped back (via preserved source-span metadata) to the original template's line/column so diagnostics feel native to the template even though they're produced by ordinary TS type-checking of generated code.

**Q8. How does ngtsc achieve incremental rebuilds, and why is it faster than View Engine's approach?**
It layers Angular-specific incrementality on top of TypeScript's own incremental `Program` reuse. TypeScript-level: `ts.createProgram({oldProgram})` reuses unchanged `SourceFile`s. Angular-level: a dependency graph tracks, per file, which other files' Angular metadata it depends on (e.g., a component depends on the components/pipes/directives referenced in its template). On a text change, only the changed file is re-analyzed; the result's resolved metadata (selector, inputs/outputs, template shape) is compared semantically (not byte-for-byte) against the prior analysis, and if unchanged, dependents are not invalidated at all, stopping cascade early. This is faster than View Engine partly because View Engine's whole-module `.ngfactory.ts` generation meant a change to any declared component could force regeneration of the owning module's factory file and everything transitively depending on that module; Ivy's per-component *locality* means blast radius is inherently smaller.

**Q9. Why can Ivy compile a component without knowing which NgModule declares it, while View Engine could not?**
*Interviewer intent: probes understanding of the "locality principle" — a foundational Ivy design decision that also explains why standalone components were even possible.*
View Engine's compiled output for a component depended on its enclosing `NgModule`'s compilation scope — the module declared which other components/directives/pipes were visible inside the component's template, and the *generated* factory baked that resolved scope directly into static code, so the whole module had to be known and stable for any one component in it to compile. Ivy instead has each component declare, on itself, references to the directives/pipes it uses (originally still resolved through `NgModule` metadata attached to the class, and directly through `imports: [...]` for standalone components) as data consumed at *runtime* by the `LView`'s `TView` construction, not baked into the emitted instruction stream's shape. This means a component's own `ɵɵdefineComponent` output is fully determined by its own decorator + template + direct imports, independent of any module — enabling standalone components, better tree-shaking, and cheaper incremental analysis.

**Q10. What kinds of decorator argument values are *not* statically analyzable, and what error does the developer see?**
Anything the `PartialEvaluator` cannot resolve to a concrete literal/reference chain: arbitrary function/method call results (`selector: computeSelector()`), values obtained through non-traceable dynamic property access, spreads of dynamically-produced arrays, and values whose originating declaration crosses into code the evaluator can't see through (e.g., certain complex re-export chains, or `require()`-style dynamic imports). The developer sees a diagnostic like `Value at position N in the NgModule.imports of AppModule is not a reference` or `... is not statically analyzable` pointing at the exact expression's source span. The standard fix is to lift the value into a `const` with a directly literal initializer, or (for circular-dependency cases specifically) use `forwardRef(() => X)`, which the compiler special-cases to defer resolution.

**Q11. How does the Angular Language Service provide template autocompletion, and why is it considered "reliable" (i.e., matches the real compiler)?**
It reuses the exact same `ngtsc` program construction, `PartialEvaluator`, R3 AST template parser, and TCB generator that a full `ng build` uses — it is not a separate, approximate implementation. When the editor requests completions inside a template expression, the Language Service maps the cursor position to the corresponding position inside that component's synthetic Type Check Block, delegates to the ordinary TypeScript `LanguageService.getCompletionsAtPosition` at that mapped location (getting real, type-checker-backed completions), then maps the returned completion list back to template coordinates for the editor to render. Because it is literally the same type-checking machinery as the batch compiler, a clean editor (no red squiggles) reliably predicts a clean `ng build`, and vice versa — there's no separate "IDE type system" that could disagree with the "real" one.

**Q12. Explain "partial compilation mode" and why it exists.**
*Interviewer intent: tests awareness of a genuinely tricky historical Angular problem — cross-version library compatibility — and whether the candidate can connect the compiler's IR design to that product requirement.*
Normally ngtsc lowers a component all the way to concrete `ɵɵdefineComponent(...)` calls bound to instructions from a *specific* installed `@angular/core` version's Ivy runtime — fine for an application (compiled and run against one known Angular version) but brittle for a published library, since the library's compiled output would be tied to whatever Angular version built it, breaking if a consuming app uses a different (even patch-different, pre-stabilization) runtime instruction set. `compilationMode: 'partial'` instead emits a version-tolerant, "linkable" intermediate form (`ɵɵngDeclareComponent(...)`, `ɵɵngDeclareDirective(...)`, etc.) that encodes the resolved metadata/template compilation result in a stable, Angular-version-independent shape; when an application later builds against that library, its own build process runs a "linker" step that re-lowers the declared form into concrete instructions matching *its own* installed Angular version. This is why libraries built with `ng-packagr` can safely be consumed across a wider range of Angular versions than the exact one they were compiled with.

**Q13. Why does `strict` mode sometimes surface template errors that don't correspond to an obvious template-only mistake, and what should a developer actually look for?**
Because TCB-derived diagnostics originate from generic TypeScript type-checking of synthesized code, they can reflect issues in *component class* typings that only become visible once the compiler models the template's actual data flow — e.g. a component property typed as `T | undefined` used in a template without a guarding `*ngIf`/`?.`, which is a real potential runtime null-access the class's own type signature was hiding, not a template syntax mistake. The fix is almost always either narrowing (`*ngIf`, `?.`, `??`) in the template, tightening/loosening the underlying property's type, or (last resort, with justification) `$any()` to intentionally opt out of checking for that specific expression.

**Q14. What TS compiler API surface does ngtsc actually rely on (beyond parsing), and why not fork TypeScript instead?**
It relies on the public `ts.Program`/`ts.CompilerHost`/`ts.LanguageService` extensibility points: custom `CompilerHost` for resource resolution, `program.emit(...)`'s `customTransformers` hook for AST rewriting, `ts.TypeChecker` for symbol/type resolution feeding the `PartialEvaluator` and TCB diagnostics, and `oldProgram` reuse for incrementality. Not forking TypeScript means Angular automatically inherits every improvement to TS's own incremental algorithms, project-reference handling, and language-server protocol support, and guarantees Angular-compiled code and diagnostics stay compatible with the broader TS tooling ecosystem (editors, `tsc --build`, project references) rather than diverging into a parallel, harder-to-maintain toolchain.

## 7. Quick Revision Cheat Sheet

- **`ngtsc`** = Ivy compiler; runs as custom `ts.TransformerFactory` hooks inside a single `tsc` invocation. **`ngc`** (View Engine) = separate multi-pass compiler generating `.ngfactory.ts` files after a `.metadata.json` collection pass.
- **Decorator metadata is read via `ts.TypeChecker` + `PartialEvaluator`** — static AST interpretation, never runtime reflection. Unresolvable expressions become `DynamicValue` → "not statically analyzable" errors.
- **Pipeline**: TS `Program` → decorator detection (`DecoratorHandler`s: Component/Directive/NgModule/Pipe/Injectable) → template HTML parse → R3 AST (desugars `*ngIf`, control-flow blocks) → binding/expression parser (own grammar, own AST, own lexer) → (if `strictTemplates`) TCB generation & type-check → instruction selection + Constant Pool → output-AST → real `ts.Node` synthesis → spliced into class via `before` transformer → `tsc` emits JS/`.d.ts`.
- **R3 AST** ≠ HTML AST ≠ TS AST — three distinct tree types at three distinct stages.
- **Constant Pool**: dedupes static attribute arrays / pure-function closures / i18n metadata into hoisted `_c0`, `_c1`... module-scope constants.
- **Creation vs update instructions**: creation runs once (`ɵɵelementStart`, `ɵɵtemplate`, `ɵɵlistener`); update runs every CD pass, gated by `ɵɵadvance(n)` (`ɵɵproperty`, `ɵɵtextInterpolateN`, `ɵɵclassProp`). Arity-specialized instructions (`...1`..`...8`, `...V`) avoid runtime array allocation.
- **Locality principle**: a component compiles from its own decorator + template + direct imports only, not its owning `NgModule`'s full scope — enables standalone components, tree-shaking, small incremental blast radius.
- **Incrementality** = TS `oldProgram` reuse + Angular semantic-diffing dependency graph (compares resolved metadata shape, not file bytes) → skips re-analyzing dependents whose public shape didn't change.
- **Language Service** reuses the same `ngtsc` program/TCB/evaluator as full builds — maps cursor position into the synthetic TCB and delegates to the real TS `LanguageService` for completions/diagnostics, so editor and `ng build` never disagree.
- **Partial compilation mode** (`ɵɵngDeclareComponent`) emits a version-tolerant IR for published libraries, "linked" against the consuming app's actual Angular runtime version at its own build time.
- **Common gotcha**: decorator config expressions must bottom out in literals/traceable `const`s — no arbitrary function calls, no non-traceable spreads; use `forwardRef(() => X)` for legitimate circular-reference cases.
