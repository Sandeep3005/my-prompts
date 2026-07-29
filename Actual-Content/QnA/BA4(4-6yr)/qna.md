# Frontend Interview Bank — 4–6 Years Experience (With Answers)

**Format:** 7 topics × 10 questions = 70 questions. Within each topic, questions go from simple (Q1) to tough (Q10).
**Note:** Code questions are "predict the output / spot the bug" style — no terminal needed.

**Suggested use:** Pick 3–4 topics per round. Q1–Q4 = warm-up, Q5–Q8 = core signal, Q9–Q10 = differentiators.

---

## 1. React

### Q1. What is the difference between state and props? When would you lift state up?

**Answer:**
Props are read-only inputs passed from a parent; state is data owned and mutated by the component itself. Lift state up when two sibling components need to read or write the same value — move it to their closest common ancestor and pass down value + setter.
*Strong signal:* mentions that lifting too far causes unnecessary re-renders and that context or a store is the alternative.

### Q2. What does the dependency array of `useEffect` control? What happens with `[]` vs no array vs `[dep]`?

**Answer:**
- No array → effect runs after every render.
- `[]` → runs once after mount (and cleanup on unmount).
- `[dep]` → runs on mount and whenever `dep` changes by `Object.is` comparison.
*Strong signal:* notes that objects/arrays/functions in deps break equality every render unless memoized.

### Q3. Why does React need a `key` on list items? What's wrong with using the array index?

**Answer:**
Keys let React match elements across renders to decide reuse vs remount. Index keys break when the list is reordered, filtered, or items are inserted/removed — React reuses the wrong DOM node, so component state (input values, focus, animation) leaks to the wrong row.
*Strong signal:* index keys are fine for static, append-only, never-reordered lists.

### Q4. What will this log, and why?

```jsx
function Counter() {
  const [count, setCount] = useState(0);
  const handle = () => {
    setCount(count + 1);
    setCount(count + 1);
    console.log(count);
  };
  return <button onClick={handle}>{count}</button>;
}
```

**Answer:**
Logs `0`, and count becomes `1` (not 2). `count` is captured from the current render's closure, so both calls compute `0 + 1`. State updates are batched and the variable is not reassigned mid-handler. Fix: `setCount(c => c + 1)` twice → 2.

### Q5. Explain `useMemo` vs `useCallback` vs `React.memo`. When do they actually help?

**Answer:**
- `useMemo` caches a computed *value*.
- `useCallback` caches a *function identity*.
- `React.memo` skips re-render of a child when its props are shallow-equal.
They help when (a) a computation is genuinely expensive, or (b) a stable reference is required to keep a memoized child or an effect dependency from re-triggering. Otherwise they add memory + comparison cost for nothing.
*Strong signal:* `useCallback` is pointless if the child isn't memoized.

### Q6. What is the difference between controlled and uncontrolled components? Which do you prefer for a large form and why?

**Answer:**
Controlled = value lives in React state, every keystroke re-renders. Uncontrolled = DOM holds the value, read via ref/FormData. For large forms, uncontrolled (or a library like React Hook Form using refs + subscriptions) avoids re-rendering the whole form on each keystroke.
*Strong signal:* mentions validation timing and that controlled is easier for cross-field logic.

### Q7. What does this render? Explain the effect timing.

```jsx
useEffect(() => {
  console.log("A");
  return () => console.log("B");
}, [id]);
```
`id` changes from 1 → 2.

**Answer:**
Order: `A` (mount with id=1) … then on change: `B` (cleanup for id=1) then `A` (effect for id=2). Cleanup always runs before the next effect with the *previous* closure's values — this is why stale-request cancellation belongs in the cleanup.

### Q8. What problem does `useRef` solve that `useState` doesn't? Give two real uses.

**Answer:**
`useRef` gives a mutable box that persists across renders *without* triggering re-render. Uses: (1) holding a DOM node for focus/scroll/measure, (2) storing mutable instance data — interval IDs, previous values, an `isMounted` flag, a latest-callback ref to avoid stale closures.

### Q9. A list re-renders every keystroke even though the items didn't change. Walk me through how you'd diagnose and fix it.

**Answer:**
Diagnose with React DevTools Profiler / "Highlight updates" to confirm which components re-render and why. Common causes: parent state change cascading down, new object/array/function literals passed as props each render, context value recreated inline, list not memoized.
Fixes: `React.memo` on the row + `useCallback`/`useMemo` for props, split state so the input owns its own state, memoize context value, or virtualize the list.
*Strong signal:* measures before optimizing.

### Q10. Explain what happens on state update in React 18 — batching, and what "concurrent rendering" changes for you as a developer.

**Answer:**
React 18 batches *all* updates by default (including in promises, timeouts, native handlers), not just React event handlers. Concurrent rendering means a render can be interrupted, paused, and restarted — so render functions must be pure and side-effect free, and effects may run more than once in StrictMode dev. `useTransition`/`startTransition` marks updates as non-urgent so typing stays responsive; `useDeferredValue` defers an expensive derived value.
*Strong signal:* mentions "render can be thrown away, so never mutate during render".

---

## 2. TypeScript

### Q1. What's the difference between `any`, `unknown`, and `never`?

**Answer:**
`any` disables type checking entirely (contagious, dangerous). `unknown` is the type-safe top type — you can assign anything to it but must narrow before use. `never` is the bottom type — no value inhabits it; it's the return type of a function that always throws or loops forever, and the result of an impossible union.

### Q2. `interface` vs `type` — what are the practical differences?

**Answer:**
`interface` supports declaration merging and is idiomatic for object/class shapes and public APIs. `type` can express unions, intersections, tuples, mapped/conditional types, and primitives. Both support extension. Practical rule: `interface` for object contracts you may want extended/merged, `type` for anything unions or computed.

### Q3. What's the difference between `unknown[]`, `any[]`, and a tuple `[string, number]`?

**Answer:**
`any[]` — elements unchecked. `unknown[]` — array of unknown, each element must be narrowed. Tuple — fixed length and per-position types; `[string, number]` allows exactly two elements in that order (though `push` can still bypass length unless it's `readonly`).

### Q4. What does this produce?

```ts
type A = { a: string };
type B = { b: number };
type C = A & B;
type D = A | B;
const x: C = { a: "1", b: 2 };
const y: D = { a: "1" };
```

**Answer:**
Both are valid. `&` (intersection) requires *all* members; `|` (union) requires at least one branch to match. Common gotcha: intersection of conflicting primitive types (`string & number`) collapses to `never`.

### Q5. What are generics for? Write the signature of a function that returns the first element of an array, preserving the type.

**Answer:**
Generics let a type flow through a function rather than being fixed. `const first = <T,>(arr: T[]): T | undefined => arr[0];`
*Strong signal:* returns `T | undefined` rather than `T`, and mentions `noUncheckedIndexedAccess`.

### Q6. Explain the utility types `Partial`, `Pick`, `Omit`, `Record`, and `ReturnType`. Give one use for each.

**Answer:**
- `Partial<T>` — all props optional; PATCH payloads, form drafts.
- `Pick<T, K>` — subset; narrowing a DTO for a component prop.
- `Omit<T, K>` — remove keys; strip `id` from a create payload.
- `Record<K, V>` — map keys to values; lookup dictionaries, enum→config maps.
- `ReturnType<typeof fn>` — derive a type from an existing function instead of duplicating it.

### Q7. What is a discriminated union and why is it better than optional fields? Show a small example.

**Answer:**
A union of object types sharing a literal "tag" field, letting TS narrow exhaustively:
```ts
type State =
  | { status: "loading" }
  | { status: "success"; data: User }
  | { status: "error"; error: Error };
```
Better than `{ loading?: boolean; data?: User; error?: Error }` because impossible combinations (loading *and* error) become unrepresentable, and a `switch` gets exhaustiveness checking via a `never` default.

### Q8. What is type narrowing? Name four ways to narrow, and explain what a type guard function is.

**Answer:**
Narrowing = TS refining a broad type within a branch. Ways: `typeof`, `instanceof`, `in`, literal/discriminant equality, truthiness, `Array.isArray`. A user-defined type guard is `const isUser = (v: unknown): v is User => ...` — the `v is User` predicate tells the compiler the branch is safe.
*Strong signal:* notes guards are unchecked assertions — the runtime logic must actually be correct.

### Q9. Why is this unsafe, and what does `strict` mode buy you?

```ts
const data = JSON.parse(res) as User;
```

**Answer:**
`as` is an assertion, not a validation — `JSON.parse` returns `any`, and the cast silently lies if the server changes. Nothing is checked at runtime. Fix: validate at the boundary with a schema library (Zod/Valibot/io-ts) and infer the type from the schema.
`strict` enables `strictNullChecks`, `noImplicitAny`, `strictFunctionTypes`, etc. — most real bug-catching lives there, especially null checks.

### Q10. Explain `keyof`, `typeof`, and mapped types. What does this do?

```ts
type Getters<T> = { [K in keyof T & string as `get${Capitalize<K>}`]: () => T[K] };
```

**Answer:**
`keyof T` = union of T's keys; `typeof x` lifts a value into its type; a mapped type iterates keys to build a new type. This one remaps each string key to a `getXxx` method returning that property's type — e.g. `{ name: string }` → `{ getName: () => string }`. Uses key remapping (`as`) and a template literal type.

---

## 3. JavaScript

### Q1. `==` vs `===`, and what is `null == undefined`?

**Answer:**
`===` compares type and value with no coercion; `==` coerces. `null == undefined` is `true`, but `null === undefined` is `false`, and `null == 0` is `false`. Rule of thumb: always `===`, except the idiomatic `x == null` to catch both null and undefined.

### Q2. Explain `var` vs `let` vs `const` — scope, hoisting, and reassignment.

**Answer:**
`var` is function-scoped, hoisted and initialized to `undefined`. `let`/`const` are block-scoped and hoisted but in the temporal dead zone until initialized (accessing early throws `ReferenceError`). `const` blocks reassignment of the binding, not mutation of the object it points to.

### Q3. What does this log?

```js
for (var i = 0; i < 3; i++) setTimeout(() => console.log(i), 0);
for (let j = 0; j < 3; j++) setTimeout(() => console.log(j), 0);
```

**Answer:**
`3 3 3` then `0 1 2`. `var` has one function-scoped binding shared by all callbacks (already 3 when timers fire); `let` creates a fresh binding per iteration.

### Q4. What is a closure? Give a practical use.

**Answer:**
A function retaining access to its lexical scope after that scope has returned. Practical uses: private state (counters, module pattern), memoization caches, debounce/throttle holding a timer ID, partial application, and React hooks capturing values per render.

### Q5. Order the output.

```js
console.log("A");
setTimeout(() => console.log("B"), 0);
Promise.resolve().then(() => console.log("C"));
queueMicrotask(() => console.log("D"));
console.log("E");
```

**Answer:**
`A E C D B`. Sync code first (A, E), then the microtask queue drains in FIFO order (C, D), then macrotasks/timers (B). Key point: microtasks always run to completion before the next macrotask.

### Q6. What is `this` in each case?

```js
const obj = {
  name: "x",
  regular() { return this.name; },
  arrow: () => this?.name,
};
const fn = obj.regular;
```

**Answer:**
`obj.regular()` → `"x"` (call-site receiver). `obj.arrow()` → `undefined` (arrow has no own `this`; it takes the enclosing scope's, typically module/global). `fn()` → `undefined` in strict mode / throws, because the receiver is lost on detachment. Fix with `.bind(obj)` or a wrapper arrow.

### Q7. Explain shallow vs deep copy. What do `{...obj}`, `Object.assign`, and `structuredClone` do?

**Answer:**
Spread and `Object.assign` copy one level — nested objects stay shared by reference. `structuredClone` deep-clones and handles Dates, Maps, Sets, and cycles (but not functions, DOM nodes, or class prototypes). `JSON.parse(JSON.stringify(x))` deep-copies but loses `undefined`, functions, `Date` → string, and throws on cycles.

### Q8. What's the difference between `Promise.all`, `allSettled`, `race`, and `any`?

**Answer:**
- `all` — resolves with all values, rejects fast on the first rejection.
- `allSettled` — never rejects; returns `{status, value|reason}` per promise.
- `race` — settles with the first to settle, fulfilled *or* rejected.
- `any` — resolves with the first fulfilment, rejects with `AggregateError` only if all reject.
*Strong signal:* notes none of them cancel the other in-flight requests.

### Q9. What does this output, and why is it a classic bug?

```js
const arr = [1, 2, 3];
const result = arr.map(async (n) => n * 2);
console.log(result);
```

**Answer:**
`[Promise, Promise, Promise]` — `map` doesn't await. `async` callbacks always return promises. Fix: `await Promise.all(arr.map(async n => n * 2))`. Same trap with `forEach` + `await`, which silently doesn't wait at all — use `for...of` for sequential work.

### Q10. Explain event delegation, bubbling vs capturing, and why `e.stopPropagation()` can be dangerous.

**Answer:**
Events capture down from the root, hit the target, then bubble up. Delegation attaches one listener on a container and inspects `e.target` — fewer listeners, works for dynamically added children. `stopPropagation` prevents ancestors from ever seeing the event, which silently breaks outside-click handlers, analytics, modal/dropdown dismissal, and third-party widgets. Prefer checking `e.target`/`e.currentTarget` or `preventDefault` where that's what you actually mean.

---

## 4. UI Design Principles

### Q1. What is visual hierarchy and which tools create it?

**Answer:**
Hierarchy is the order in which the eye consumes the page. Tools: size, weight, colour/contrast, spacing, position, and grouping. The rule is one clear primary action per view; everything else steps down. *Strong signal:* mentions that if everything is emphasized, nothing is.

### Q2. What do proximity, alignment, and consistency (Gestalt/CRAP principles) mean in practice?

**Answer:**
Proximity — related items sit closer than unrelated ones; whitespace groups better than borders. Alignment — shared edges create an invisible grid and reduce visual noise. Consistency — the same component/spacing/wording means the same thing everywhere, so users transfer learning.

### Q3. Why does a design system use a spacing scale (4/8px) instead of arbitrary values?

**Answer:**
A constrained scale removes per-decision guesswork, makes rhythm consistent across teams, keeps designs and code in sync via tokens, and prevents the 13px/15px/17px drift that makes a UI feel sloppy. It also reduces CSS surface area.

### Q4. What are the WCAG contrast minimums, and why is colour alone not enough to convey state?

**Answer:**
4.5:1 for normal text, 3:1 for large text (≥18.66px bold / 24px) and for UI component/graphic boundaries (AA). Colour alone fails colour-blind users (~8% of men) and low-contrast environments — pair colour with an icon, text label, or pattern. Error states need text, not just a red border.

### Q5. Design the loading, empty, and error states for a data table. What does each need?

**Answer:**
- **Loading:** skeleton matching the final layout (avoids layout shift), not a centred spinner that hides context.
- **Empty:** distinguish "no data yet" (onboarding, primary CTA) from "no results for this filter" (offer to clear the filter). Explain + give an action.
- **Error:** plain-language cause, a retry affordance, preserve the user's input/filters, and a support/error ID if relevant.
*Strong signal:* mentions partial/stale data and optimistic states too.

### Q6. What is affordance and feedback? Give an example of each being violated.

**Answer:**
Affordance = the UI signals what it can do (a button looks pressable). Feedback = the system acknowledges the action within ~100ms. Violations: a clickable div with no hover/cursor/focus style; a Save button that shows nothing for 3 seconds so users click it three times.

### Q7. How do you make a form accessible and forgiving? Walk through the specifics.

**Answer:**
Real `<label for>` (not placeholder-as-label), logical tab order, visible focus ring, group related fields with `fieldset/legend`, inputs typed correctly (`type=email`, `inputmode`), errors tied via `aria-describedby` and announced via a live region, validation on blur/submit rather than on every keystroke, don't clear the form on error, and 44×44px minimum touch targets.

### Q8. Mobile-first vs desktop-first — what changes in how you build, and what are common responsive mistakes?

**Answer:**
Mobile-first writes base styles for the smallest viewport and layers `min-width` media queries, which forces content prioritisation and produces less override CSS. Common mistakes: hiding content instead of restructuring it, hover-only interactions with no touch equivalent, fixed heights, ignoring the on-screen keyboard and safe-area insets, breakpoints chosen from device names rather than where the content breaks.

### Q9. A page feels "slow" even though the API responds in 200ms. What UI-level causes and fixes would you consider?

**Answer:**
Causes: layout shift (CLS) from late-loading images/fonts, blocking spinners replacing the whole view, no optimistic update, long main-thread tasks blocking input (INP), unbatched sequential requests, animations over 300ms, waiting for all data before showing anything.
Fixes: skeletons, optimistic UI with rollback, stale-while-revalidate, streaming/progressive rendering, reserved space for media, `font-display: swap`, debounced input with immediate visual acknowledgment.

### Q10. How would you introduce a design system into an existing product with inconsistent UI? What do you do about migration?

**Answer:**
Audit and inventory existing components (count the button variants — that's the pitch). Start with tokens (colour, spacing, type, radius, motion) since they give broad wins cheaply. Build the highest-traffic primitives first with accessibility built in. Adopt incrementally: new work uses the system, migrate old screens opportunistically or per-route, with a codemod/lint rule blocking raw hex values and one-off spacing. Document usage + do/don't, version it, and measure adoption. Avoid a big-bang rewrite.

---

## 5. Shift-Left Testing (unit, mutation, contract, component/Playwright)

### Q1. What does "shift left" mean, and why does catching a bug earlier matter?

**Answer:**
Moving quality activities — tests, types, linting, contract checks, accessibility checks, review — as early as possible in the lifecycle, ideally into the developer's local loop and PR pipeline instead of a QA phase after code-complete. It matters because the cost and blast radius of a defect grows by orders of magnitude the later it's found, and the feedback loop for the person who wrote the code is fastest while the context is fresh.

### Q2. Explain the testing pyramid (or trophy). Where should the bulk of your tests live for a frontend app?

**Answer:**
Pyramid: many fast unit tests, fewer integration, fewest E2E. The "testing trophy" (Kent C. Dodds) argues for frontend the widest layer should be *integration/component* tests, because they exercise real user behaviour with good speed. Static analysis (TS + ESLint) is the base. E2E covers a handful of critical revenue paths only.

### Q3. What makes a good unit test? What's wrong with testing implementation details?

**Answer:**
Good: deterministic, isolated, fast, one behaviour per test, arranged/act/assert, fails with a message that tells you what broke. Testing implementation (internal state, private methods, CSS class names, hook call counts) makes tests fail on refactors that didn't change behaviour — high maintenance, low confidence. Test observable behaviour and public contracts.

### Q4. What is code coverage, and why is 100% coverage not the goal?

**Answer:**
Coverage measures which lines/branches executed during tests — not whether anything was asserted. A test with no assertions still covers lines. It's a *find-the-gaps* tool, not a quality score; chasing 100% incentivises trivial tests on getters and generated code. Better: high coverage on critical paths, plus mutation score for real assertion quality.

### Q5. What is mutation testing? What does a "surviving mutant" tell you?

**Answer:**
The tool (Stryker for JS/TS) makes small deliberate changes to your source — flip `>` to `>=`, remove a statement, negate a condition — then reruns the tests. If tests still pass, the mutant *survived*, meaning no test actually asserted that behaviour. Mutation score = killed / total viable mutants. It's the answer to "my coverage is 90% but bugs still ship". Cost: slow, so run it on critical modules or nightly, not every PR.

### Q6. What is contract testing and what problem does it solve that E2E doesn't?

**Answer:**
Consumer-driven contract testing (Pact, or schema-based checks) has the consumer record its expectations of a provider's API; the provider verifies those expectations in *its own* pipeline against a shared broker. It catches breaking API changes without spinning up every service together. E2E does catch it, but only later, more slowly, more flakily, and it needs all services deployed at once. Contract tests are fast, run independently, and pinpoint who broke what.

### Q7. What is component testing in Playwright, and how does it differ from an E2E test and from a Testing Library unit test?

**Answer:**
Playwright component testing mounts a single component in a real browser and drives it with real input, so you get real layout, CSS, focus, and events without a full app or server. E2E boots the whole app and hits real routes/backends — highest fidelity, slowest, flakiest. Testing Library + jsdom is fastest but simulates the DOM, so layout, real focus behaviour, and visual state are approximations. Use component tests for interaction-heavy or visually-dependent widgets.

### Q8. How do you deal with flaky tests? Name concrete causes and fixes in Playwright.

**Answer:**
Causes: hard `waitForTimeout`, racing on network, shared mutable state between tests, animations, time/locale/timezone dependence, test order dependence, real third-party services.
Fixes: use web-first auto-retrying assertions (`await expect(locator).toBeVisible()`) instead of sleeps; role/label-based locators instead of CSS/XPath; `route.fulfill` to stub network; seed and isolate data per test; freeze the clock; disable animations; run tests fully parallel and independent; quarantine and *fix* flakes rather than blanket-retrying. Blanket retries hide real race conditions.

### Q9. How do you decide what to mock? What's the risk of over-mocking?

**Answer:**
Mock at architectural boundaries you don't own — network, time, randomness, third-party SDKs, filesystem. Don't mock your own internal modules just to make a test easier; that's a design smell. Over-mocking produces tests that pass while production is broken because the mock drifted from reality. Mitigate with MSW (intercepting at the network layer so app code is untouched), contract tests to pin the real shape, and a thin layer of real integration tests.

### Q10. Design a testing strategy for a checkout flow. What runs on pre-commit, on PR, and nightly?

**Answer:**
- **Pre-commit (seconds):** TypeScript, ESLint, Prettier, changed-file unit tests.
- **PR (minutes):** full unit + component tests (Playwright CT / RTL), MSW-backed integration tests, consumer contract tests published to the broker, a11y checks (axe), bundle-size budget, and a small smoke E2E of the happy path against a preview deploy.
- **Nightly / scheduled:** full E2E matrix across browsers, provider contract verification, mutation testing on payment/pricing logic, visual regression, performance budgets (Lighthouse), and load tests.
Plus production guardrails: feature flags, canary release, synthetic monitoring on the checkout path, and error/RUM tracking.
*Strong signal:* ties test placement to feedback speed and risk, and mentions that payment logic deserves mutation testing specifically.

---

## 6. Coding Standards

### Q1. What is linting vs formatting? Why do you need both, and why automate them?

**Answer:**
Formatting (Prettier) is deterministic whitespace/style — no bug-finding. Linting (ESLint) is static analysis for correctness and consistency (unused vars, exhaustive deps, forbidden APIs). Automating both ends style debate in review, keeps diffs clean, and makes the standard the default rather than a document nobody reads. Enforce in pre-commit hooks *and* CI, since hooks can be bypassed.

### Q2. What makes a good function or variable name? Rewrite `const d = getD(x)` for a date-difference calculation.

**Answer:**
Names should say intent, not type or implementation — searchable, pronounceable, no unexplained abbreviations, booleans read as predicates (`isActive`, `hasAccess`), functions are verbs. `const daysUntilDue = getDaysBetween(today, dueDate);`

### Q3. What is DRY, and when is duplication actually the better choice?

**Answer:**
DRY = one authoritative source for each piece of *knowledge*. Duplication is better when two pieces of code only look alike coincidentally — abstracting them couples unrelated concerns and the abstraction ends up with boolean flags and branches. "Prefer duplication over the wrong abstraction"; wait for the third occurrence and for the shared reason to be clear.

### Q4. What makes a good pull request and a good code review comment?

**Answer:**
PR: small and single-purpose, descriptive title, a "why" in the body, screenshots for UI, self-reviewed first, tests included, green CI.
Review: prioritise correctness/security/design over style (let tools handle style), ask questions rather than issue verdicts, distinguish blocking from nit ("nit:" prefix), explain reasoning, and approve rather than nitpick indefinitely. Review promptly — a PR sitting for two days is a bigger cost than most of the comments on it.

### Q5. What makes a comment useful vs noise? When should code be self-documenting?

**Answer:**
Useful comments explain *why* — the non-obvious constraint, the workaround with a ticket link, the business rule, the perf tradeoff. Noise restates the code (`// increment i`) and rots. If you need a comment to explain *what* code does, prefer extracting a well-named function. JSDoc on exported/public APIs is worth it.

### Q6. What's wrong with this code? List everything.

```js
function calc(a, b, c) {
  if (a == null) return;
  var res = a.price * b;
  if (c == true) { res = res - res * 0.1 }
  console.log(res)
  return res
}
```

**Answer:**
Meaningless names (`calc`, `a/b/c`); `var` instead of `const`; magic number `0.1` with no named constant; `c == true` instead of a truthy check on a boolean named `hasDiscount`; a silent `return undefined` for the null case (inconsistent return type — throw or return a typed result); leftover `console.log`; float arithmetic on money; no types; missing semicolons relying on ASI; the function does two things (calculate and log); no unit test.

### Q7. How do you handle errors consistently across a codebase?

**Answer:**
Decide and document: throw typed error classes (or return a Result type) rather than returning `null` for failures; never swallow with an empty `catch`; catch only where you can add context or recover; preserve the cause (`new Error(msg, { cause })`); centralise user-facing mapping (error boundary / interceptor); log with correlation IDs and structured context, never log secrets/PII; distinguish expected domain errors from unexpected bugs so alerting stays meaningful.

### Q8. How do you keep a shared codebase consistent as the team grows? What do you automate vs document?

**Answer:**
Automate everything mechanical: Prettier, ESLint with shared config, TS strict, commit-lint + conventional commits, CODEOWNERS, PR templates, dependency and bundle-size checks, import boundary rules (`eslint-plugin-boundaries`), pre-commit hooks + CI as the gate. Document only judgement calls: folder/architecture conventions, state-management choices, error-handling and testing philosophy, ADRs for significant decisions. A rule that can be linted should never be a doc.

### Q9. What is technical debt? How do you make the case to pay it down?

**Answer:**
Debt is a deliberate or accidental shortcut whose interest is paid as slower future change. Frame it in business terms: cycle time, incident rate, onboarding time, escaped defects, hotspot files that keep appearing in bug fixes. Tactics: allocate a standing percentage of each sprint, attach cleanup to features touching the same area (boy-scout rule), track it as visible tickets with impact, and prioritise debt that sits on the roadmap's critical path rather than debt that merely offends you.

### Q10. What would you put in a frontend codebase's CONTRIBUTING/standards doc for a new team?

**Answer:**
Branch and commit conventions; PR size/review expectations and SLAs; folder structure and module boundaries; component conventions (naming, props, composition over config); state-management decision tree (local → lifted → context → server-cache → global store); styling approach and design tokens; testing expectations per layer with a definition of done; accessibility baseline; error handling and logging; dependency-addition policy; performance budgets; a short ADR log for "why we do it this way". Keep it short enough to be read, and link to enforced tooling rather than restating rules.

---

## 7. APIs and Backend Calls

### Q1. What do these status codes mean: 200, 201, 204, 400, 401, 403, 404, 409, 422, 429, 500, 503?

**Answer:**
200 OK; 201 Created (with `Location`); 204 No Content; 400 malformed request; 401 unauthenticated (who are you?); 403 authenticated but not permitted; 404 not found; 409 conflict (version/duplicate); 422 semantically invalid payload; 429 rate-limited (respect `Retry-After`); 500 server error; 503 unavailable (usually retryable).
*Key distinction:* 401 vs 403, and 4xx = don't blindly retry, 5xx/429 = retry with backoff.

### Q2. GET vs POST vs PUT vs PATCH vs DELETE. What do idempotent and safe mean?

**Answer:**
GET (safe, idempotent, cacheable), POST (neither — creates), PUT (idempotent full replace), PATCH (partial update, not necessarily idempotent), DELETE (idempotent — deleting twice leaves the same state). Safe = no state change. Idempotent = repeating the call yields the same resulting state, which is what makes retries safe. For POST, use an idempotency key.

### Q3. What is CORS and what actually causes a preflight request?

**Answer:**
CORS is a *browser* policy: cross-origin requests need the server to opt in via `Access-Control-Allow-Origin` and friends. A preflight `OPTIONS` fires when the request isn't "simple" — non-simple method (PUT/PATCH/DELETE), custom headers (`Authorization`, `X-*`), or a `Content-Type` other than form/text/plain (so `application/json` triggers it). Fixed on the server, never in the frontend; with credentials you need an explicit origin (not `*`) plus `Allow-Credentials: true`.

### Q4. `fetch` doesn't throw on a 404. What does that mean for your error handling?

**Answer:**
`fetch` only rejects on network failure/CORS/abort — an HTTP error status still resolves. You must check `res.ok` (or `res.status`) and throw yourself. Axios does this for you. Also: `res.json()` throws on an empty body (e.g. 204), so guard it. Wrap this once in an API client rather than repeating it at every call site.

### Q5. How do you cancel an in-flight request, and why does it matter in React?

**Answer:**
`AbortController` — pass `signal` to fetch and call `abort()` in the effect cleanup (or use a library's built-in cancellation / TanStack Query). It matters for race conditions: fast typing in a search box means response #1 can land after response #3, so without cancellation or a request-sequence guard you render stale results. Also prevents state updates for unmounted/superseded views.

### Q6. Where do you store an auth token, and what are the tradeoffs?

**Answer:**
`localStorage` is readable by any JS on the page → XSS steals it, but it's simple and works cross-domain. An `HttpOnly; Secure; SameSite` cookie is unreadable by JS (XSS-resistant) but needs CSRF protection and same-site/CORS care. The common pattern: short-lived access token in memory, refresh token in an HttpOnly cookie. Never put tokens in URLs (leaks via logs/referrer), and treat XSS as the root risk either way.

### Q7. What is a debounce vs throttle, and how would you use each for API calls?

**Answer:**
Debounce waits for a pause in activity then fires once — right for search-as-you-type or autosave (300–500ms). Throttle fires at most once per interval regardless of activity — right for scroll/resize/mousemove or a rate-limited endpoint. Debounce alone still needs cancellation of superseded requests.

### Q8. Explain retries, exponential backoff, and when NOT to retry.

**Answer:**
Retry only idempotent requests and only on transient failures — network errors, 502/503/504, and 429 respecting `Retry-After`. Use exponential backoff with jitter to avoid a thundering herd, cap attempts, and surface failure to the user rather than retrying forever. Never auto-retry 400/401/403/404/422 — the request is wrong and retrying just amplifies load. A circuit breaker prevents hammering a service that's already down.

### Q9. How would you design the data-fetching layer for a dashboard with 6 widgets, each needing different data?

**Answer:**
Prefer parallel independent requests over a waterfall, each widget owning its own loading/error boundary so one slow endpoint doesn't block the page. Use a server-cache library (TanStack Query/SWR) for dedup, caching, stale-while-revalidate, and background refetch; key the cache per widget+params. Consider a BFF or a batched endpoint if the number of round trips is the bottleneck, and streaming/Suspense for progressive render. Add request cancellation on unmount, retry policy, and a shared error boundary for auth failures.
*Strong signal:* mentions measuring whether the bottleneck is round trips, payload size, or backend latency before choosing.

### Q10. REST vs GraphQL — what tradeoffs matter on the frontend? And what is over-fetching vs under-fetching?

**Answer:**
Over-fetching = the endpoint returns more fields than the screen needs (payload/bandwidth). Under-fetching = the screen needs several round trips to assemble one view (waterfall latency).
GraphQL: one round trip, client-specified fields, strong schema/typegen — at the cost of harder HTTP caching, N+1 risk on the server, query-cost/depth-limit concerns, and a heavier client. REST: simple, cacheable at CDN/HTTP level, easy to reason about and debug — at the cost of endpoint proliferation or a BFF layer to shape responses. Choose by team, caching needs, and how varied the client's data requirements are.
