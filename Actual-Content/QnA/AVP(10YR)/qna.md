# Frontend Interview Bank — 8–10 Years Experience (With Answers)

**Format:** 7 topics × 10 questions = 70 questions. Within each topic, questions go from simple (Q1) to tough (Q10).
**Note:** Code questions are "predict the output / spot the bug" style — no terminal needed.

**What you're screening for at this level:** not recall, but judgement — tradeoffs, failure modes, blast radius, migration strategy, and the ability to say "it depends, and here's on what". A candidate who only gives the textbook answer without the caveat is a mid-level candidate.

---

## 1. React

### Q1. Explain reconciliation and the role of the Fiber architecture.

**Answer:**
Reconciliation is diffing the new element tree against the previous one to compute the minimum set of DOM mutations. React assumes different element types produce different trees, and uses keys to match siblings. Fiber re-implemented this as an interruptible linked-list walk with a work loop, split into a render phase (pure, interruptible, can be discarded or restarted) and a commit phase (synchronous, side-effectful). That split is what makes concurrent features, priority lanes, and time-slicing possible.
*Strong signal:* explains why render must be pure — it may run twice or be thrown away.

### Q2. Compare state colocation, Context, and an external store (Redux/Zustand/Jotai). What's your decision tree?

**Answer:**
Default to the most local option. Lift only when genuinely shared. Context is a dependency-injection mechanism, not a state manager — every consumer re-renders when the value's identity changes, so it suits low-frequency values (theme, locale, auth user) and is a poor fit for high-frequency state. External stores add selector-based subscriptions, so components re-render only on the slice they read; use them for cross-cutting, high-churn, or middleware-needing state. Server data belongs in a server-cache library, not in a global client store at all — that distinction removes most "we need Redux" cases.

### Q3. What does `useSyncExternalStore` solve? What is tearing?

**Answer:**
Tearing is when a concurrent render is interrupted, the external store changes mid-render, and different components in the same commit read different values — producing an internally inconsistent UI. `useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot)` lets React subscribe to external mutable sources in a way that forces a consistent read, opting those reads out of concurrent tearing. It's why store libraries had to be rewritten for React 18, and `getServerSnapshot` covers SSR/hydration mismatch.

### Q4. What are Server Components, and what actually changes in your mental model versus SSR?

**Answer:**
Classic SSR renders client components to HTML on the server, then ships and hydrates the same JS. RSC components run *only* on the server, never ship their code to the client, can await data directly, and are serialized as a component payload that streams and merges into the client tree without hydration. Consequences: no hooks/state/effects/event handlers in server components, a hard serialization boundary at `"use client"`, secrets can live server-side safely, bundle size drops for non-interactive subtrees, and you now design around where the boundary sits rather than where the data-fetching hook lives.
*Strong signal:* mentions that the client bundle is determined by the boundary placement, and that RSC is not a performance silver bullet for highly interactive UIs.

### Q5. Explain hydration errors — root causes and how you'd systematically prevent them.

**Answer:**
Hydration mismatch means server HTML ≠ first client render. Causes: `Date.now()`/`Math.random()`, locale/timezone formatting, reading `window`/`localStorage` during render, browser extensions mutating DOM, invalid HTML nesting the browser silently corrects (`<div>` inside `<p>`), and user-agent-conditional rendering.
Prevention: make render deterministic; move browser-only reads into `useEffect` or a mounted flag; pass a server-generated timestamp/ID down as a prop; use `useId` for IDs; format dates with an explicit timezone; render the neutral state on both sides and enhance after mount; `suppressHydrationWarning` only for genuinely unavoidable single values. Selective hydration in 18 makes mismatches recoverable but still costly.

### Q6. What will this render, and what is the underlying rule?

```jsx
function Parent() {
  const [n, setN] = useState(0);
  const value = { n };
  return <Ctx.Provider value={value}><ExpensiveChild /></Ctx.Provider>;
}
const ExpensiveChild = React.memo(() => { /* no context use */ });
```

**Answer:**
`ExpensiveChild` does *not* re-render on `setN` — `React.memo` holds, because it's passed as `children`/an element whose props didn't change and it doesn't consume the context. But if it *did* consume `Ctx`, `memo` would not save it: context consumers bypass memo entirely and re-render whenever the provider value's identity changes — and `value` is a new object every render. Fixes: `useMemo` the value, split contexts by update frequency, or move to selector-based subscriptions.
*Strong signal:* knows memo does not block context propagation, and knows the "pass children through" trick to avoid re-rendering subtrees.

### Q7. You inherit a React app with a 4MB bundle and a 6-second LCP. Walk through your diagnosis and remediation plan.

**Answer:**
Measure first: RUM field data (Core Web Vitals from CrUX/RUM, not just lab Lighthouse), then a bundle analyzer for composition, and a performance trace for main-thread work.
Common findings and fixes: moment/lodash/icon libraries imported wholesale → tree-shakeable alternatives and per-module imports; no route-level code splitting → `React.lazy` per route plus prefetch on intent; render-blocking CSS/fonts → critical CSS, `font-display: swap`, preload the LCP image; client-side-only rendering → SSR/SSG or streaming for above-the-fold content; duplicate transitive dependencies → dedupe/resolutions; heavy third-party scripts → defer, lazy-load, or remove.
Then: enforce a CI bundle-size budget and a performance budget so it doesn't regress. Sequence by impact/effort, ship incrementally, and verify against field data — not a local Lighthouse score.

### Q8. Design an error-handling strategy for a large React app.

**Answer:**
Layered. Error boundaries at route level (keep the shell and nav alive) plus around risky widgets, with a reset key so recovery is possible without a full reload. Boundaries don't catch async/event-handler/SSR errors — those need explicit handling, a query-library `onError`, and `window.onerror`/`unhandledrejection` for the tail. Distinguish expected domain errors (render inline, actionable, keep user input) from unexpected bugs (generic message + retry + report). Centralise API error mapping in one client/interceptor; handle 401 refresh once globally with request queuing. Report to Sentry with release + source maps, user/session/correlation ID, and breadcrumbs; scrub PII. Add alerting thresholds on error rate per route, and feature flags/kill switches so a bad path can be turned off without a deploy.

### Q9. How would you migrate a large class-component + Redux codebase to hooks and a modern data layer, without freezing feature work?

**Answer:**
Strangler-fig, not big bang. Establish the target architecture and a written ADR first. Introduce the new patterns at the boundary — new features and new routes only — with lint rules preventing new code in the old pattern. Split "server state" out of Redux into TanStack Query/RTK Query first: that usually deletes the majority of Redux (thunks, loading flags, normalized caches) for the largest win. Migrate leaf components before containers. Codemods for mechanical transforms, manual review for lifecycle→effect semantics (that's where bugs hide: `componentDidUpdate` conditionals, `getDerivedStateFromProps`). Keep both systems interoperable during the transition (a bridge/adapter). Gate behind flags, migrate by route with metrics and error-rate comparison, and track percentage-migrated publicly so it doesn't stall at 60%. Regression safety comes from characterization tests written *before* touching a module.

### Q10. When is React the wrong choice, and what would you weigh in choosing a rendering strategy (CSR/SSR/SSG/ISR/RSC) for a new product?

**Answer:**
React is a poor fit for mostly-static content sites (a static generator or islands framework ships far less JS), for hard real-time/canvas/graphics-heavy work where the reconciler is overhead, for tightly constrained low-end-device or low-bandwidth contexts, and for teams without the depth to maintain the ecosystem churn.
Rendering choice drivers: SEO and social crawlers; time-to-first-byte vs time-to-interactive; content freshness (build-time vs request-time vs revalidate); personalization (per-user content can't be CDN-cached the same way); infrastructure and cost (static CDN vs always-on server); team familiarity and operational maturity. Practical answer: SSG/ISR for marketing and catalogue, SSR/RSC + streaming for personalized shells, CSR for authenticated app-like surfaces behind a login — and hybrid per-route rather than one global choice.
*Strong signal:* frames it as per-route, and mentions the operational cost of SSR (cache strategy, cold starts, server capacity) that teams underestimate.

---

## 2. TypeScript

### Q1. What is structural typing, and where does it surprise people?

**Answer:**
TS compares shapes, not declared names — anything with the right members is assignable. Surprises: two unrelated domain types (`UserId` and `OrderId`, both `string`) are interchangeable, so bugs pass the compiler; excess property checks only fire on *fresh object literals*, so an intermediate variable slips extra props through; empty interfaces accept almost anything. Remedy: branded/nominal types (`type UserId = string & { __brand: "UserId" }`) for identifiers that must not be mixed.

### Q2. Explain variance: covariance, contravariance, and why `strictFunctionTypes` exists.

**Answer:**
Return types are covariant (a function returning `Dog` is assignable where one returning `Animal` is expected). Parameters are *contravariant* in a sound system — a handler accepting `Animal` is usable where one accepting `Dog` is expected, not the reverse. TS historically made parameters bivariant for ergonomics; `strictFunctionTypes` enforces contravariance for function-type positions but deliberately leaves *method* declarations bivariant, which is why array/`Promise` method signatures stay permissive. This is the root of the classic unsound `Array<Dog>` → `Array<Animal>` assignment.

### Q3. What do conditional types and `infer` do? Explain distributivity.

**Answer:**
`T extends U ? X : Y` branches at the type level; `infer` captures a type variable in the extends clause (`type Unwrap<T> = T extends Promise<infer U> ? U : T`). When the checked type is a naked type parameter and the input is a union, the conditional *distributes* over each member — `Unwrap<Promise<A> | B>` becomes `A | B`. Wrapping in a tuple (`[T] extends [U]`) disables distribution, which is the standard trick for `IsNever`/`IsAny`-style checks.

### Q4. What does this evaluate to?

```ts
type A = string extends string | number ? 1 : 2;
type B<T> = T extends string ? 1 : 2;
type C = B<string | number>;
type D = [string | number] extends [string] ? 1 : 2;
```

**Answer:**
`A = 1`. `C = 1 | 2` (distributes over the union: `B<string>`=1, `B<number>`=2). `D = 2` (tuple wrapping blocks distribution, and `string|number` is not assignable to `string`). The point: distribution is the single most common source of "why is my conditional type a union?".

### Q5. How do you model an exhaustive state machine in types so that adding a state breaks the build?

**Answer:**
Discriminated union for states, plus an exhaustiveness guard:
```ts
const assertNever = (x: never): never => { throw new Error(`Unhandled: ${JSON.stringify(x)}`); };
switch (state.status) {
  case "idle": ...
  default: return assertNever(state);
}
```
Add a new variant and the `never` assignment fails at compile time. Extend it by typing the transition table as `Record<Status, Partial<Record<Event, Status>>>` so illegal transitions are unrepresentable, or adopt XState for anything with guards, history, and side-effects.

### Q6. How do you type an API boundary safely, and what's the argument against generated types alone?

**Answer:**
Types are erased at runtime, so a generated `openapi-typescript` type is still just a compile-time promise — it doesn't stop a server from returning something else after a deploy skew, a partial cache, or a proxy error page. The robust pattern is a schema (Zod/Valibot) as the *single source of truth*, `z.infer` for the static type, and `parse` at the boundary so bad data fails loudly at the edge with a useful message instead of surfacing as `undefined is not a function` three components deep. Generated types are excellent for *authoring* speed and drift detection in CI; combine both — generate from the spec, validate at runtime on responses you don't control, and skip validation on hot paths you do control if profiling demands it.

### Q7. What are declaration merging, module augmentation, and when have you legitimately needed them?

**Answer:**
Interfaces and namespaces with the same name merge. Module augmentation (`declare module "express" { interface Request { user?: User } }`) extends third-party types without forking them. Legitimate uses: adding a property a middleware attaches, extending a theme type for styled-components/MUI, typing `import.meta.env`, adding custom matchers to `expect`, or declaring non-code imports (`*.svg`). Risk: it's global and invisible — an augmentation in one package silently changes types everywhere, so keep them in a dedicated `types/` file and document them.

### Q8. What are the real costs of heavy type-level programming, and how do you diagnose slow type-checking?

**Answer:**
Deeply recursive conditional/mapped types, huge unions (cartesian blowups), and inference-heavy generic chains cause exponential checker work — slow editors, slow CI, and eventually "type instantiation is excessively deep". Diagnose with `tsc --extendedDiagnostics`, `--generateTrace` viewed in Perfetto, and `--noEmit` timing per project.
Mitigations: annotate return types explicitly to stop inference cascades, prefer interfaces over large intersections, split with project references/incremental builds, cap recursion, replace clever generics with plain overloads, and set `skipLibCheck`. The judgement call: type-level cleverness is a cost paid by everyone who touches the file — reserve it for library boundaries, not application code.

### Q9. When is `as` or a `@ts-expect-error` the right call, and how do you keep escape hatches from spreading?

**Answer:**
Legitimate when you genuinely know more than the compiler and can't express it: a validated boundary right after a runtime check, third-party types that are wrong, test doubles, or a narrowing the checker can't follow. `@ts-expect-error` is strictly better than `@ts-ignore` because it errors if the underlying problem is fixed — it self-cleans.
Containment: require a comment with a reason and ticket; lint rules banning bare `any` and `as any`; keep assertions inside a thin adapter layer so the unsafe surface is auditable; measure with `type-coverage` in CI; prefer a validated guard over an assertion whenever the cost is acceptable. Never use `as` to silence a real modelling problem — that's how a codebase becomes typed-in-name-only.

### Q10. You're introducing TypeScript to a large JS codebase. What's your migration and enforcement plan?

**Answer:**
Enable TS with `allowJs`, `checkJs: false`, and non-strict settings so the build is green on day one — adoption dies if the first PR is 800 errors. Then ratchet: turn on one strict flag at a time, starting with `noImplicitAny` and `strictNullChecks` (the highest value, hardest one — do it per-directory). Convert leaves and shared utilities first, then types for the API boundary (generated from the spec), then containers. Type third-party gaps with local `.d.ts` rather than `any`. Enforce with `type-coverage` thresholds and a lint rule that new files must be `.ts`, plus CODEOWNERS on the config so nobody quietly disables strict. Track progress as a visible metric. Expect to find real bugs during `strictNullChecks` conversion — budget for them.
*Strong signal:* mentions that the migration's value is concentrated in null-safety and the API boundary, not in annotating every internal function.

---

## 3. JavaScript

### Q1. Explain the event loop precisely: call stack, task queue, microtask queue, and rendering.

**Answer:**
Per turn: run the current macrotask to completion → drain the *entire* microtask queue (promises, `queueMicrotask`, MutationObserver), including microtasks queued by microtasks → then, if it's time, run `requestAnimationFrame` callbacks → style/layout/paint → possibly `requestIdleCallback`. Then the next macrotask (timers, I/O, DOM events). Consequences: an infinite microtask chain starves rendering entirely; a long sync task blocks input (this is what INP measures); `setTimeout(0)` is clamped (~4ms after nesting) and never runs before pending microtasks.

### Q2. Explain the prototype chain and how `class` maps onto it.

**Answer:**
Every object has an internal `[[Prototype]]` link; property lookup walks the chain until `Object.prototype` then `null`. `class` is syntax over constructor functions: methods live on `Ctor.prototype` (shared), fields declared in the class body are per-instance, `extends` sets both the prototype chain and the static chain, `super()` must run before `this`, and class bodies are always strict mode and not hoisted-callable. Practical implication: methods are shared and cheap; arrow-function class fields create one closure per instance but auto-bind `this`.

### Q3. What does this log?

```js
const o = {
  a: 1,
  get b() { return this.a + 1; },
};
const { b } = o;
const p = Object.create(o);
p.a = 10;
console.log(b, p.b, o.hasOwnProperty("b"), p.hasOwnProperty("b"));
```

**Answer:**
`2 11 true false`. Destructuring *invokes* the getter once with `this = o` → 2. `p.b` finds the getter up the prototype chain but runs it with `this = p`, so `10 + 1 = 11`. The getter is an own property of `o`, not of `p`. This is the mechanic behind prototype-based defaults — and behind the "why did my value freeze at destructuring time" bug.

### Q4. Explain generators and async iterators. Give a real use.

**Answer:**
A generator (`function*`) returns a lazy iterator you drive with `next()`, and `yield` can both emit and receive values, so it's a coroutine — the basis of redux-saga and of how async/await is modelled. `for await...of` consumes async iterators. Real uses: paginating an API transparently (`async function* fetchAll()` yielding pages until the cursor is null), streaming a response body chunk by chunk, backpressure-friendly processing of large datasets, infinite sequences, and cancellable long-running loops.

### Q5. What are `WeakMap`/`WeakRef` for, and how do memory leaks actually happen in a SPA?

**Answer:**
`WeakMap`/`WeakSet` hold keys weakly, so an entry disappears when the key is otherwise unreachable — ideal for attaching metadata/private state to objects or DOM nodes without pinning them. Common SPA leaks: listeners on `window`/`document` never removed, intervals/timeouts not cleared, subscriptions/observers left open, detached DOM nodes retained by a closure or a cache, growing module-level caches with no eviction, and closures capturing large objects. Diagnose with heap snapshots and the "three snapshot" technique, plus the detached-nodes filter and the allocation timeline.

### Q6. Explain how modern JS engines optimize code, and what "deoptimization" means for you in practice.

**Answer:**
The engine interprets first (Ignition), profiles, then JIT-compiles hot functions (TurboFan) with speculative assumptions based on observed shapes ("hidden classes"/maps) and types — inline caches make repeated property access on the same shape cheap. If an assumption breaks (a new property added later, mixed types in an array, changed object shape order), the engine bails back to the interpreter and re-optimizes: that's deopt.
Practical guidance: initialize all fields in one place and keep object shapes stable and monomorphic, avoid mixing element types in arrays, don't `delete` properties (use `null` or a Map), and avoid megamorphic call sites. But: measure. Micro-tuning is almost never the bottleneck compared to network, layout, and framework overhead.

### Q7. What happens on `structuredClone` vs the spread operator here, and what breaks?

```js
const map = new Map([["k", { v: 1 }]]);
const a = { ...{ map }, d: new Date() };
const b = structuredClone({ map, d: new Date() });
```

**Answer:**
Spread copies references — `a.map` is the *same* Map instance, so mutating it mutates the original; `a.d` is the same Date object. `structuredClone` performs a deep structured clone: `b.map` is a new Map with cloned entries, `b.d` a new Date, and cycles are handled. It throws on functions, symbols, DOM nodes, and Error subclass identity/class prototypes are lost (you get plain objects for class instances). That last point is the usual production surprise.

### Q8. How do Web Workers, SharedArrayBuffer, and `Atomics` fit together — and when is a worker actually worth it?

**Answer:**
Workers run on a separate thread with no DOM access, communicating via `postMessage` structured-clone (a copy) or transferables (zero-copy ownership handoff for ArrayBuffers). `SharedArrayBuffer` gives genuinely shared memory, with `Atomics` for safe coordination — but it requires cross-origin isolation headers (COOP/COEP) post-Spectre, which is often the real blocker.
Worth it when there is sustained CPU work that jank the main thread: parsing large JSON/CSV, image/video processing, crypto, diffing large datasets, search indexing. Not worth it for I/O-bound work (fetch is already async), or when the postMessage serialization cost exceeds the computation — measure the round trip first. Also consider `scheduler.yield`/time-slicing as a cheaper alternative.

### Q9. What are the real security concerns in frontend JS — XSS, CSRF, prototype pollution, supply chain — and what actually mitigates each?

**Answer:**
- **XSS:** untrusted data reaching an HTML/JS sink. Mitigate with framework escaping by default, never `dangerouslySetInnerHTML`/`innerHTML` without DOMPurify, a strict CSP with nonces (no `unsafe-inline`), and Trusted Types.
- **CSRF:** the browser auto-sending cookies. Mitigate with `SameSite=Lax/Strict`, anti-CSRF tokens, and checking `Origin`. Token-in-header auth is largely immune.
- **Prototype pollution:** merging attacker-controlled JSON that sets `__proto__`. Mitigate with `Object.create(null)` for maps, guarded deep-merge, `Object.freeze(Object.prototype)`, and schema validation.
- **Supply chain:** lockfiles + integrity hashes, `npm audit`/Dependabot/Socket, pinned versions, minimal dependencies, SRI on CDN scripts, restricting postinstall scripts, and a CSP that limits where scripts may load from.
*Strong signal:* notes that CSP is the single highest-leverage control and that "sanitize on output, validate on input" is the ordering rule.

### Q10. Where do you draw the line between a library and hand-rolling, and how do you evaluate a dependency?

**Answer:**
Evaluate: does it solve a problem that is genuinely hard or high-risk to get right (dates/timezones, i18n, crypto, virtualization, accessibility-complete primitives, rich text)? Then check bundle cost (with tree-shaking actually verified), maintenance signal (release cadence, open issue triage, bus factor, funding), transitive dependency count, license, TypeScript quality, escape hatches, and how expensive removal would be if it's abandoned.
Hand-roll trivia (a debounce, a classnames helper, a shallow-equal) rather than adding a dependency and its supply-chain surface. Wrap third-party libraries behind your own thin interface at the boundary so swapping them later is a contained change. And weigh the org cost: every dependency is a thing someone must patch at 2am when a CVE lands.

---

## 4. UI Design Principles

### Q1. What is a design token system, and why is a three-tier structure (primitive → semantic → component) worth the indirection?

**Answer:**
Primitives are raw values (`blue-600`, `space-4`). Semantic tokens express intent (`color-action-primary`, `color-text-danger`). Component tokens scope to a component (`button-bg-hover`). The indirection means a rebrand, a dark theme, or a density mode changes one layer without touching component code, and designers/engineers share one vocabulary. Without the semantic layer you get `blue-600` hardcoded in 400 places and no way to theme. Tooling: Style Dictionary / W3C DTCG format to emit CSS vars, native platform values, and Figma variables from one source.

### Q2. How do you decide component API design — props explosion vs compound components vs render props?

**Answer:**
A flat prop list is fine until it grows boolean flags that combine invalidly (`isPrimary` + `isGhost`). Compound components (`<Select><Select.Trigger/><Select.Options/></Select>`) trade a little verbosity for composability and let consumers reorder and inject freely without you predicting every case. Headless/hook-based APIs (`useSelect`) go further: behaviour and accessibility from the library, markup and styling entirely from the consumer — best for design-system primitives that must fit an opinionated visual language. Rule of thumb: configuration for the 90% case, composition for the tail. Watch for the API becoming a leaky reimplementation of children.

### Q3. What does "accessible by default" mean for a component library, and what do you test?

**Answer:**
Semantics and behaviour are correct without the consumer doing anything: proper roles and native elements, full keyboard operation including arrow-key composite widgets per the ARIA Authoring Practices, focus management (trap in modals, restore on close, visible focus ring), `aria-live` for async status, respecting `prefers-reduced-motion` and `prefers-contrast`, contrast-passing default tokens, and no keyboard trap.
Testing: axe in unit/component tests catches maybe 30–40% of issues; add keyboard-only traversal tests in Playwright, screen-reader spot checks (NVDA/VoiceOver) on complex widgets, zoom to 200% and 400% reflow, and a manual audit checklist per release. The honest answer includes "automated tooling is necessary but nowhere near sufficient".

### Q4. How do you approach dark mode and theming properly?

**Answer:**
Not an inverted palette — semantic tokens re-mapped, with elevation expressed through surface lightness rather than shadows (shadows barely read on dark), desaturated accent colours (saturated hues vibrate on dark), re-checked contrast for every pair, and adjusted image/illustration treatment. Implementation: CSS custom properties swapped by a `data-theme` attribute or `light-dark()`/`color-scheme`, default to `prefers-color-scheme` with a user override persisted, set the theme before first paint via a blocking inline script to avoid a flash, and set `color-scheme` so native controls and scrollbars follow. Test both themes in visual regression, not just one.

### Q5. What are Core Web Vitals, and which UI decisions most affect each?

**Answer:**
- **LCP (loading, ≤2.5s):** the largest above-fold element. Affected by hero image size/format/priority, render-blocking CSS/fonts, client-only rendering, and slow TTFB.
- **INP (interactivity, ≤200ms):** replaced FID; measures worst-case interaction latency. Affected by long tasks, heavy re-renders on input, unbatched state updates, expensive synchronous handlers, and third-party scripts.
- **CLS (stability, ≤0.1):** affected by images/ads/embeds without reserved dimensions, late-injected banners, font swap without size-adjust, and content inserted above existing content.
*Strong signal:* insists on field data over lab scores, and knows INP is usually the hardest for React apps.

### Q6. How do you handle internationalization and RTL properly in a UI?

**Answer:**
Design for text expansion (German/Finnish +30–40%, so no fixed-width buttons or truncation-dependent layouts), never concatenate translated fragments (use ICU MessageFormat with placeholders, plurals, and gender), format dates/numbers/currency with `Intl` and the user's locale and timezone rather than hand-rolling. For RTL: use CSS logical properties (`margin-inline-start`, `padding-block`, `inset-inline`) instead of left/right so mirroring is automatic, mirror directional icons but not logos/media controls, and remember numbers and code stay LTR inside RTL text. Test with pseudo-localization early. Also: locale affects sort order, name/address forms, calendars, and first day of week.

### Q7. Critique this from a design-principles standpoint: a "Delete account" button styled identically to "Save changes", both in a row at the bottom of a settings page.

**Answer:**
Failures: no visual hierarchy or destructive affordance (destructive actions should be visually distinct and *not* adjacent to the primary action — proximity invites misclicks); irreversible action with no confirmation, no type-to-confirm, and presumably no undo; a low-frequency destructive action given equal prominence to the routine one; likely no keyboard/focus consideration so tab-then-enter lands somewhere dangerous.
Better: move account deletion to a separate "Danger zone" section, secondary/destructive styling, explicit consequences stated, confirmation requiring intent (typing the account name), and a soft-delete grace period with an email notification. Undo beats confirm where technically possible — confirmation dialogs get click-through-blindness.

### Q8. How do you run a design-system governance model across multiple product teams?

**Answer:**
Decide the operating model: centralized (one team owns everything — consistent, becomes a bottleneck), federated (contributors from product teams with core review — scales, needs strong standards), or hybrid, which is what usually works. Concretely: a public contribution process with a component RFC/proposal template, a defined promotion path (product-local → candidate → core), semantic versioning with a documented deprecation policy and codemods for breaking changes, a shared Figma↔code token pipeline so design and code can't drift, adoption metrics per team, office hours and a support channel, and a release cadence teams can plan around.
Failure modes to name: the system that ships components nobody asked for, and the system that says no to everything so teams fork it.

### Q9. How do you evaluate whether a UI change actually worked?

**Answer:**
Define the metric before shipping and distinguish leading from lagging indicators. Quantitative: task success and completion time, funnel conversion, error/validation-failure rate, rage clicks and dead clicks, support ticket volume on that flow, retention. Method: A/B test with a pre-registered primary metric, adequate power and runtime (full weekly cycles), guardrail metrics so you don't win conversion while wrecking latency or refunds, and awareness of novelty effects and peeking. Qualitative: moderated usability testing (5 users surfaces most severe issues), session replay, and support/CS themes.
*Strong signal:* says that not everything should be A/B tested — some changes are table stakes (accessibility, obvious bugs) and testing them wastes time — and that a flat result is a real result.

### Q10. Where do you push back on a design that's beautiful but expensive to build or maintain, and how do you have that conversation?

**Answer:**
Push back where cost is structural rather than cosmetic: bespoke one-off components that duplicate system primitives, layouts that can't survive text expansion or 200% zoom, animation-heavy interactions that will break `prefers-reduced-motion` and jank on mid-tier Android, custom form controls reimplementing native accessibility, or anything requiring a heavy dependency for a marginal visual effect.
How: bring data not opinion (bundle/latency numbers, device distribution from analytics, accessibility obligations), offer a cheaper alternative that preserves the *intent* rather than just saying no, quantify the tradeoff in time ("this version is 2 days, that one is 2 sprints plus ongoing maintenance — here's what we'd cut"), and escalate to the shared goal rather than the artifact. Involve engineering during design, not at handoff — most of these conversations are cheap in week one and expensive in week six.

---

## 5. Shift-Left Testing (unit, mutation, contract, component/Playwright)

### Q1. What is your test strategy philosophy, and how do you decide the shape of the suite for a given product?

**Answer:**
Shape follows risk and architecture, not dogma. Drivers: how many services are involved (more integration points → more contract testing), how much logic is pure vs glue (pure domain logic rewards unit tests, glue rewards integration), regulatory/financial exposure, deploy frequency, and how expensive an incident is. For a typical React product I'd weight heavily toward component/integration tests with MSW, keep unit tests for pure logic and edge cases, contract-test every service boundary, and keep E2E to a small, ruthlessly maintained set of revenue-critical journeys. The metric that matters is defect escape rate and mean time to detection, not test count.

### Q2. How do you make mutation testing viable in a real CI pipeline?

**Answer:**
Full-repo mutation runs are too slow for PRs. Practical setup: Stryker with incremental mode and `--since` so only changed files are mutated on PR; full runs nightly or weekly; scope by directory with a higher threshold on high-risk modules (pricing, auth, permissions, tax/discount logic) and a lower or zero threshold on UI glue. Parallelize across workers, exclude generated code and config, and use the mutant report to *find missing assertions*, not as a vanity number. Tune out equivalent mutants (semantically identical changes that can never be killed) so the score stays credible — an untrusted metric gets ignored.
*Strong signal:* names the equivalent-mutant problem and the cost/benefit boundary.

### Q3. Explain consumer-driven contract testing end to end, including the CI workflow and `can-i-deploy`.

**Answer:**
The consumer writes tests against a mock provider; the framework records interactions into a pact file and publishes it to a broker tagged with the consumer version and branch. The provider's pipeline pulls the relevant pacts and replays them against the real provider (with provider states to set up data), then publishes verification results. Before deploying, each side runs `can-i-deploy` against the broker, which answers "is the version I'm about to deploy compatible with what's currently in that environment?" — that's the gate. Broker features that make it work at scale: pending pacts (a new consumer expectation doesn't break the provider's build immediately), WIP pacts, and branch/environment tagging.
Limits: contract tests verify shape and interaction, not business correctness or performance — they're not a substitute for the provider's own tests.

### Q4. Playwright: explain the auto-waiting model, locator strategy, and the trace viewer.

**Answer:**
Locators are lazy and re-resolve on each action; before acting Playwright waits for actionability — attached, visible, stable (not animating), enabled, and receiving events (not covered). Web-first assertions (`await expect(locator).toHaveText(...)`) retry until timeout, which removes almost all need for explicit sleeps.
Locator priority: user-facing and accessibility-first — `getByRole`, `getByLabel`, `getByPlaceholder`, `getByText`, then `data-testid` as an explicit contract, and CSS/XPath last because they couple to markup. `getByRole` doubles as a light accessibility check.
Trace viewer records a DOM snapshot per action plus network, console, and source — so a CI failure is debuggable post-hoc without reproduction. Configure `trace: "on-first-retry"` to keep artifacts small.

### Q5. How do you handle test data and environment isolation for E2E at scale?

**Answer:**
Never share mutable fixtures across tests — that's the main source of order-dependence and flake. Options in rough order of preference: create data per test via API (fast, explicit) and clean up after; seeded ephemeral database per worker or per branch (containerized/preview environments); tenant-scoped data with a unique run ID so tests can run in parallel against a shared environment; snapshot-restore for expensive setup.
Auth: log in once via API and reuse `storageState` rather than driving the login UI in every test. Third parties: stub at the network layer or use provider sandboxes with deterministic responses. Time: inject a clock. Also: make cleanup idempotent and run a scheduled sweeper, because tests will crash mid-run.

### Q6. What does this Playwright test do wrong?

```ts
test("checkout", async ({ page }) => {
  await page.goto("/cart");
  await page.waitForTimeout(2000);
  await page.click(".btn-primary");
  expect(await page.locator(".total").textContent()).toBe("$99.00");
  await page.click("text=Pay");
});
```

**Answer:**
- `waitForTimeout` — arbitrary sleep: slow when unnecessary, flaky when insufficient.
- `.btn-primary` — CSS class coupling; breaks on restyle and is ambiguous if there are two.
- `expect(await ...textContent())` — a one-shot assertion with no retry; should be `await expect(page.locator(...)).toHaveText("$99.00")`.
- Hardcoded `"$99.00"` — depends on shared, mutable, locale-formatted data.
- No test isolation/setup: assumes a cart already exists.
- `text=Pay` may match multiple elements; `getByRole("button", { name: "Pay" })` is unambiguous.
- Clicking Pay with no assertion afterward — the test ends without verifying the outcome, and may hit a real payment provider.

### Q7. How do you keep an E2E suite from becoming a 90-minute flaky tax on every merge?

**Answer:**
Budget it explicitly (e.g. "PR feedback under 10 minutes") and treat that as a constraint on suite size. Techniques: shard across many parallel workers/containers; run only smoke E2E on PR and the full matrix on merge/nightly; test-impact analysis to select affected specs; API-based setup so UI steps only cover what's actually under test; delete tests that duplicate cheaper coverage; push assertions down the pyramid.
Flake discipline: track flake rate per spec as a first-class metric, auto-quarantine above a threshold with an owning team and a deadline, and never allow blanket retries as a permanent fix — retries hide races that will become production incidents. A suite people bypass has negative value.

### Q8. How do you test things that are genuinely hard — animations, drag-and-drop, canvas, real-time/websockets, third-party iframes?

**Answer:**
- **Animations:** disable them (`prefers-reduced-motion`, CSS override) and assert end state; test the *logic* of the transition separately.
- **Drag and drop:** native HTML5 DnD is poorly supported by automation — use `mouse.down/move/up` sequences, or test the underlying reorder function as a unit and cover the wiring with one smoke test.
- **Canvas/WebGL:** no DOM to assert; test the model/state layer directly and use pixel/visual regression with tolerance for the rendering itself.
- **Websockets/real-time:** stub the transport and drive events deterministically; test reconnection and out-of-order/duplicate messages explicitly, since those are the real bugs.
- **Third-party iframes (payments, auth):** use provider sandbox/test cards, use `frameLocator`, and keep it to one or two tests — don't build a suite whose failures you can't fix.
The general principle: push the assertion to the lowest layer that still gives real confidence, and accept lower coverage on the parts where automation cost exceeds its value.

### Q9. How do you shift left beyond tests — what else belongs in the developer's local loop and the PR gate?

**Answer:**
Types and lint, obviously, but also: schema/contract validation, accessibility checks (axe + keyboard smoke), security scanning (SAST, dependency CVEs, secret scanning pre-commit), bundle-size and performance budgets, i18n key-extraction checks, database migration compatibility checks, feature-flag hygiene, and preview environments so PMs and designers can review the real thing before merge. Also observability shifted left: structured logging, tracing, and error reporting written *with* the feature, and dashboards/alerts defined in the same PR — so a rollout is observable on day one rather than after the first incident. Plus a rollback/kill-switch plan as part of the definition of done.
*Strong signal:* frames it as "reduce the time between making a mistake and finding out", not "add more gates".

### Q10. Your team has 85% coverage, a green pipeline, and production incidents every sprint. Diagnose.

**Answer:**
Coverage measures execution, not assertion quality — start by running mutation testing on the modules involved in recent incidents; a low mutation score confirms the tests aren't asserting. Then do an incident post-mortem review across the sprint's escapes and classify them: were they *unit-testable logic bugs* (test quality problem), *integration/contract mismatches* (missing contract tests, over-mocking making tests lie), *environment/config/data differences* (test environment doesn't resemble production; needs preview envs, prod-like data, migration testing), *concurrency/race/timing bugs* (usually invisible to the suite entirely), or *requirements misunderstandings* (no amount of testing fixes that — it's a spec/review problem).
The classification tells you where to invest. Complementary moves: canary/progressive rollout with automated rollback on error-rate regression, feature flags, synthetic monitoring on critical paths, and better production observability so MTTR drops even when prevention fails. And check whether the suite is *trusted* — teams with high flake rates rubber-stamp reruns and merge red.
*Strong signal:* refuses to answer "write more tests" and instead demands the escape-classification data first.

---

## 6. Coding Standards

### Q1. How do you decide what belongs in a lint rule versus a code review versus a written guideline?

**Answer:**
Lint anything mechanical, objective, and frequently violated — it's cheap, consistent, and removes ego from the conversation. Review is for what tools can't judge: is this the right abstraction, does it fit the domain model, is the failure mode acceptable, is the naming honest, should this exist at all. Written guidelines cover recurring *judgement* calls where you want a default without re-litigating (state management choice, error handling shape, when to add a dependency). The test: if you find yourself leaving the same review comment three times, it should become a lint rule or a doc — probably a lint rule.

### Q2. How do you enforce architectural boundaries in a large frontend monorepo?

**Answer:**
Make illegal imports fail the build, not the review. Tools: `eslint-plugin-boundaries` or `import/no-restricted-paths` for layer rules (features may not import from other features' internals; shared may not import from features), Nx/Turborepo project tags with dependency constraints, package-level `exports` fields so only the public entry point is importable, TS project references, and CODEOWNERS on boundary-defining files. Add a dependency-graph check in CI and fail on cycles. Document the intent in an ADR so people understand *why* the rule exists — otherwise they route around it with a `// eslint-disable`.

### Q3. What is your review philosophy on large PRs, and what do you do when you receive one?

**Answer:**
Review quality drops sharply past a few hundred lines — beyond that, reviewers approve rather than review. Prevention is the real fix: stacked PRs, feature flags so incomplete work can merge, separating mechanical changes (renames, formatting, moves) into their own commits/PRs, and agreeing a size norm.
When one arrives: don't reflexively reject if it's already written — that wastes the work and creates resentment. Instead, ask for a split *if* it's separable, otherwise do a structured review — pass 1 for architecture and interfaces, pass 2 for correctness in risky areas, skip style entirely — and pair with the author for a walkthrough on genuinely complex changes. Then fix the process so it doesn't recur.

### Q4. How do you handle a disagreement with a senior colleague over a code standard?

**Answer:**
Separate preference from consequence. Ask what problem their position solves — often the disagreement dissolves once the constraint is explicit. Bring evidence where evidence exists (incident history, benchmark, bundle impact, the actual codebase's pain points). Timebox the debate: style-level disagreements aren't worth more than one round — pick one, encode it in tooling, move on. For consequential decisions (architecture, dependency, data flow), write a short ADR with options and tradeoffs so the discussion is about the document rather than the people, and get a decision owner. Then disagree-and-commit visibly, and revisit with data if it goes wrong — don't relitigate in review comments for six months.

### Q5. What does "definition of done" include on your team, and why each item?

**Answer:**
Code merged behind a flag where appropriate; tests at the right layer passing; types strict, no new escape hatches; accessibility checked (keyboard + axe); errors handled with user-facing messaging; telemetry/logging and a dashboard or alert if it's a critical path; docs/ADR updated if a decision was made; feature flag and rollback plan; performance budget not regressed; designer/PM review on a preview environment; and no new lint/security warnings. The purpose is to make the invisible work visible — observability, rollback, and accessibility are the items teams skip, and they're exactly the ones that cost most later.

### Q6. What's wrong with this module?

```ts
// utils.ts
export const API = "https://api.prod.example.com";
export let cache: any = {};
export async function getUser(id) {
  if (cache[id]) return cache[id];
  const r = await fetch(`${API}/users/${id}`);
  const u = await r.json();
  cache[id] = u;
  return u;
}
export function fmt(d) { return new Date(d).toLocaleDateString(); }
```

**Answer:**
Hardcoded production URL (no env config, no way to point at staging); a `utils` grab-bag with no cohesion mixing networking and formatting; exported mutable module-level state — a global unbounded cache with no TTL, no invalidation, no size cap, and a memory leak; `any` plus implicit `any` parameters; no `res.ok` check so an error page gets cached as a user; no error handling, timeout, or cancellation; cache stampede — concurrent calls for the same id fire duplicate requests (should cache the *promise*); `toLocaleDateString` with no locale or timezone argument, so output varies by machine and is untestable; `let` export creates a live binding others can reassign; no tests, no types on the response boundary.

### Q7. How do you manage dependencies and upgrades over a multi-year codebase lifetime?

**Answer:**
Policy first: a documented bar for adding a dependency (see the JS section), a lockfile committed, and renovate/dependabot grouped and scheduled so upgrades are routine rather than a yearly crisis. Separate patch/minor auto-merge (with a green suite) from majors, which get a ticket, a changelog read, and a migration branch. Keep the framework and build tooling close to current — the cost of a 4-version jump is superlinear. Track EOL and CVEs, maintain an inventory/SBOM, and budget maintenance capacity explicitly rather than pretending it's free. Wrap volatile third-party APIs behind an adapter so major upgrades touch one file. And be willing to *remove* dependencies — the cheapest upgrade is the one you deleted.

### Q8. How do you write and maintain ADRs, and what actually belongs in one?

**Answer:**
Short, immutable, numbered, in-repo next to the code. Contents: context and constraints at the time, the decision, the options considered with their tradeoffs, and the consequences (including what becomes harder). Status transitions (proposed → accepted → superseded by ADR-0042) rather than editing history — the value is knowing *why* past-you chose something under different constraints, which is exactly what gets lost.
Write one when a decision is expensive to reverse, affects multiple teams, or will be questioned by someone who wasn't there. Don't write them for reversible or local choices — an ADR process nobody reads is worse than none.

### Q9. How do you introduce standards to a team that resists them?

**Answer:**
Diagnose the resistance first: it's usually rational — past standards were imposed without context, or the standard adds friction without visible benefit, or it's a rebrand of one person's taste. So: start with the pain the team already complains about (review nitpicks, merge conflicts on formatting, recurring bug class), make the first standard something that *removes* work, and automate it so compliance is free. Get adoption via a respected engineer rather than a mandate. Apply to new code first and never demand a big-bang cleanup. Measure and share the result. Leave room to change the rule — "this is a default, propose a change with a reason" invites ownership, while "this is the standard" invites malicious compliance. And drop the rules that don't earn their keep; a bloated standards doc discredits the good rules in it.

### Q10. How do you balance shipping speed against code quality when the business is pushing hard?

**Answer:**
Reject the framing that they're opposed at every timescale — most quality practices are speed practices past about two weeks. But some genuinely trade: a spike, a prototype, a deadline-driven launch. So make the tradeoff *explicit and time-bounded* rather than silent: agree what's being deferred, write the ticket, put a flag or a boundary around the shortcut so it can't metastasize, and set a date. Distinguish the things that are never negotiable (security, data integrity, accessibility obligations, anything irreversible like a schema or public API) from things that are (test depth on a low-risk path, refactoring, internal tooling polish). Communicate in business terms — cycle time, incident rate, the cost of the last outage — and give the business a real choice rather than either a blank "no" or silent debt. Then actually pay it back when the pressure lifts, or the next negotiation has no credibility.

---

## 7. APIs and Backend Calls

### Q1. Explain HTTP caching precisely: `Cache-Control`, `ETag`, `Last-Modified`, and the difference between stale-while-revalidate and a normal revalidation.

**Answer:**
`Cache-Control` directives: `max-age` (freshness lifetime), `s-maxage` (shared/CDN only), `no-cache` (must revalidate before use — not "don't cache"), `no-store` (never persist), `private` vs `public`, `immutable` (skip revalidation entirely — pair with content-hashed filenames), `must-revalidate`. After expiry the client revalidates conditionally with `If-None-Match` (ETag) or `If-Modified-Since`; a `304` means reuse the cached body — cheap. `stale-while-revalidate=N` allows serving the stale response *immediately* while refreshing in the background, so the user never waits; `stale-if-error` serves stale content when the origin is down. Standard pattern: hashed static assets `max-age=31536000, immutable`; HTML `no-cache`; API responses tuned per endpoint with ETags and short `s-maxage` + SWR at the CDN.

### Q2. How do you design authentication for a SPA + API, end to end?

**Answer:**
OAuth2/OIDC Authorization Code flow with PKCE — the implicit flow is deprecated. Access token short-lived (5–15 min) held in memory (not localStorage, not a JS-readable cookie); refresh token in an `HttpOnly; Secure; SameSite=Strict` cookie scoped to the refresh endpoint, with rotation and reuse detection so a stolen refresh token invalidates the family. Silent refresh via a single-flight interceptor that queues concurrent 401s so you don't fire N refreshes. Logout must revoke server-side, not just clear client state. Validate JWTs on the server (signature, `iss`, `aud`, `exp`, `nbf`) and keep them small; use opaque tokens + introspection if you need instant revocation. Cross-tab sync via BroadcastChannel. Defence in depth: strict CSP, because XSS defeats every token storage choice.

### Q3. What is a BFF and when is it the right architecture?

**Answer:**
A Backend-For-Frontend is a thin server owned by the frontend team that aggregates and reshapes downstream services for one client type. Right when: the client needs many round trips to compose a view (waterfall latency), payloads are far larger than the UI needs, secrets/tokens must not reach the browser (the BFF holds them and uses a session cookie — the "token-handler" pattern), different clients (web/mobile) need different shapes, or you need to insulate the client from a churning microservice landscape.
Costs: another deployable to own, operate, and monitor; risk of becoming a monolith of business logic that belongs downstream; an extra network hop; versioning coupling. Keep it aggregation/transformation-only and owned by the team that consumes it.

### Q4. Explain idempotency keys, optimistic concurrency, and how you'd prevent a double-charge.

**Answer:**
The client generates a UUID per logical payment attempt and sends it as `Idempotency-Key`; the server stores the key with the result and returns the *same* response for a replay rather than charging twice — with a lock or unique constraint so concurrent duplicates serialize, and a TTL. Critically, the key must be generated once per intent and reused across retries — regenerating it per retry defeats the purpose.
Optimistic concurrency handles the other problem: send the ETag/version you read (`If-Match`), and the server rejects with `409/412` if it changed underneath you — so you show the user a conflict instead of silently clobbering.
Frontend obligations: disable the button on submit, treat network timeouts as *unknown* rather than failed (retry with the same key, then reconcile), and make the success path idempotent client-side too.

### Q5. Compare polling, long-polling, SSE, and WebSockets. How do you choose?

**Answer:**
- **Polling:** trivial, cacheable, stateless, wasteful; fine for low-frequency, tolerant-of-staleness data.
- **Long-polling:** lower latency, works everywhere, holds a connection per client; a legacy fallback.
- **SSE:** server→client only, over plain HTTP/2, auto-reconnect and `Last-Event-ID` built in, works with standard infra and proxies, no special protocol. Ideal for notifications, feeds, progress, LLM token streaming.
- **WebSockets:** full duplex, lowest latency, but a stateful protocol needing sticky sessions or a pub/sub backplane, its own auth story (no headers on the handshake in browsers), heartbeats, and reconnection/backoff logic you write yourself.
Choose SSE for one-way streaming (most cases), WebSockets for genuinely bidirectional/high-frequency (collab editing, trading, multiplayer). In all cases design for reconnection, out-of-order and duplicate delivery, and resumption from a cursor — that's where real systems break.

### Q6. How do you version a public API, and how do you deprecate an endpoint without breaking clients?

**Answer:**
Prefer additive, backward-compatible evolution — new optional fields, never repurposing an existing one, never tightening validation silently. When a break is unavoidable: URL versioning (`/v2/`) is bluntest and most legible; media-type/header versioning is cleaner but easier to get wrong; GraphQL uses field-level `@deprecated` instead of whole-API versions.
Deprecation process: announce with a timeline, emit `Deprecation` and `Sunset` headers, log and *measure* per-consumer usage so you know who is left, contact remaining consumers directly, dual-run both versions, then brownout (short scheduled outages) before final removal. Never remove on the announced date without confirming usage is zero. Consumer-driven contract tests tell you concretely who breaks.

### Q7. Walk through what happens between typing a URL and the page rendering, focusing on where latency hides.

**Answer:**
DNS (cacheable; use short TTLs carefully, consider prefetch) → TCP handshake (1 RTT) → TLS (1 RTT with TLS 1.3, 0-RTT for resumption) → HTTP request → server processing and origin latency → TTFB → HTML parse, which discovers subresources (preload scanner) → render-blocking CSS and synchronous JS → font fetch → layout → paint → hydration if applicable.
Latency hides in: uncached DNS, connection setup to many origins (hence `preconnect` and reducing third-party origins), slow TTFB from an uncached origin or a serverless cold start, render-blocking CSS/JS, late-discovered LCP images (fix with `fetchpriority="high"` and `preload`), font swap, and a long hydration/main-thread block at the end. HTTP/2 multiplexing removes head-of-line blocking at the app layer but not TCP-level; HTTP/3 (QUIC) fixes that and improves on lossy mobile networks. CDN edge caching addresses the largest chunk for most sites.

### Q8. How do you make a frontend resilient when a downstream API is slow or degraded?

**Answer:**
Set explicit timeouts (fetch has none by default — use `AbortSignal.timeout`), retry only idempotent calls with exponential backoff + jitter and a capped budget, and implement a client-side circuit breaker so a failing endpoint stops being hammered. Degrade gracefully: render the shell and the parts that work, serve stale cache (SWR), fall back to a reduced feature set, and make non-critical widgets fail silently inside their own error boundaries rather than taking the page down. Give the user honest feedback and a retry, preserve their input, and queue writes for background sync where offline-capable. On the org side: agree SLOs with the backend team, monitor from the client (RUM) not just the server, and have a kill switch for the feature. Also guard against the retry storm your own client can cause during an incident.

### Q9. How would you architect data fetching and caching for a large app — normalization, invalidation, optimistic updates, offline?

**Answer:**
Treat server state as a cache, not application state. Use TanStack Query/RTK Query/Apollo with structured query keys mirroring the resource hierarchy, so invalidation is targeted (`["orders", orgId]`) rather than nuke-everything. Choose document caching (simple, some duplication, per-query staleness) vs normalized caching (single source per entity, consistent updates across views, but more complexity and cache-shape bugs) based on how much entity overlap the UI has — normalization pays off in graph-like data, and is overhead in list/detail apps.
Invalidation strategy: invalidate on mutation, plus time-based staleness, plus server-pushed invalidation (SSE/websocket) for multi-user freshness. Optimistic updates need a snapshot for rollback, a version/ETag to detect conflicts, and a defined conflict-resolution policy — silent last-write-wins is a data-loss bug. Offline: persist the cache (IndexedDB), a durable mutation queue with idempotency keys, replay on reconnect, and a merge strategy (CRDT if genuinely concurrent editing).
*Strong signal:* names cache invalidation as the hard part and describes the conflict policy explicitly.

### Q10. A user reports "the app is slow" but your API p50 is 80ms. How do you investigate?

**Answer:**
p50 is the wrong statistic — start at p95/p99 and segment by route, region, device class, network, and cohort; an average hides the tail where users actually live. Then determine whether the problem is even server-side: correlate RUM (LCP/INP/TTFB from real users) with backend traces via a shared trace/correlation ID, so you can see the whole path rather than each team looking at their own green dashboard.
Candidate causes to work through: waterfalls (fast individual calls, serial dependency chain), N+1 requests, payload size vs latency, cold starts, a CDN miss or a geography with no edge presence, client-side main-thread blocking (INP) making a fast API feel slow, a slow third-party script, memory pressure on low-end devices, or a specific tenant with pathological data volume.
Then: reproduce with throttling on a mid-tier device, use a distributed trace on a real slow request, and check whether "slow" means *loading* or *unresponsive* — those have completely different fixes. Finally, define an SLO so "slow" becomes measurable and the next report is a data question, not an anecdote.
*Strong signal:* immediately challenges the metric, asks *which* users and *which* interaction, and insists on end-to-end tracing rather than debating dashboards.
