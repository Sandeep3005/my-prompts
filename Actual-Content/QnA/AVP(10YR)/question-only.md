# Frontend Interview Bank — 8–10 Years Experience (Questions Only)

**Format:** 7 topics × 10 questions = 70 questions. Within each topic, questions go from simple (Q1) to tough (Q10).
**Note:** Code questions are "predict the output / spot the bug" style — no terminal needed.

*Answer key available in the companion `-WITH-ANSWERS.md` file.*

---

## 1. React

### Q1. Explain reconciliation and the role of the Fiber architecture.

### Q2. Compare state colocation, Context, and an external store (Redux/Zustand/Jotai). What's your decision tree?

### Q3. What does `useSyncExternalStore` solve? What is tearing?

### Q4. What are Server Components, and what actually changes in your mental model versus SSR?

### Q5. Explain hydration errors — root causes and how you'd systematically prevent them.

### Q6. What will this render, and what is the underlying rule?

```jsx
function Parent() {
  const [n, setN] = useState(0);
  const value = { n };
  return <Ctx.Provider value={value}><ExpensiveChild /></Ctx.Provider>;
}
const ExpensiveChild = React.memo(() => { /* no context use */ });
```

### Q7. You inherit a React app with a 4MB bundle and a 6-second LCP. Walk through your diagnosis and remediation plan.

### Q8. Design an error-handling strategy for a large React app.

### Q9. How would you migrate a large class-component + Redux codebase to hooks and a modern data layer, without freezing feature work?

### Q10. When is React the wrong choice, and what would you weigh in choosing a rendering strategy (CSR/SSR/SSG/ISR/RSC) for a new product?

---

## 2. TypeScript

### Q1. What is structural typing, and where does it surprise people?

### Q2. Explain variance: covariance, contravariance, and why `strictFunctionTypes` exists.

### Q3. What do conditional types and `infer` do? Explain distributivity.

### Q4. What does this evaluate to?

```ts
type A = string extends string | number ? 1 : 2;
type B<T> = T extends string ? 1 : 2;
type C = B<string | number>;
type D = [string | number] extends [string] ? 1 : 2;
```

### Q5. How do you model an exhaustive state machine in types so that adding a state breaks the build?

### Q6. How do you type an API boundary safely, and what's the argument against generated types alone?

### Q7. What are declaration merging, module augmentation, and when have you legitimately needed them?

### Q8. What are the real costs of heavy type-level programming, and how do you diagnose slow type-checking?

### Q9. When is `as` or a `@ts-expect-error` the right call, and how do you keep escape hatches from spreading?

### Q10. You're introducing TypeScript to a large JS codebase. What's your migration and enforcement plan?

---

## 3. JavaScript

### Q1. Explain the event loop precisely: call stack, task queue, microtask queue, and rendering.

### Q2. Explain the prototype chain and how `class` maps onto it.

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

### Q4. Explain generators and async iterators. Give a real use.

### Q5. What are `WeakMap`/`WeakRef` for, and how do memory leaks actually happen in a SPA?

### Q6. Explain how modern JS engines optimize code, and what "deoptimization" means for you in practice.

### Q7. What happens on `structuredClone` vs the spread operator here, and what breaks?

```js
const map = new Map([["k", { v: 1 }]]);
const a = { ...{ map }, d: new Date() };
const b = structuredClone({ map, d: new Date() });
```

### Q8. How do Web Workers, SharedArrayBuffer, and `Atomics` fit together — and when is a worker actually worth it?

### Q9. What are the real security concerns in frontend JS — XSS, CSRF, prototype pollution, supply chain — and what actually mitigates each?

### Q10. Where do you draw the line between a library and hand-rolling, and how do you evaluate a dependency?

---

## 4. UI Design Principles

### Q1. What is a design token system, and why is a three-tier structure (primitive → semantic → component) worth the indirection?

### Q2. How do you decide component API design — props explosion vs compound components vs render props?

### Q3. What does "accessible by default" mean for a component library, and what do you test?

### Q4. How do you approach dark mode and theming properly?

### Q5. What are Core Web Vitals, and which UI decisions most affect each?

### Q6. How do you handle internationalization and RTL properly in a UI?

### Q7. Critique this from a design-principles standpoint: a "Delete account" button styled identically to "Save changes", both in a row at the bottom of a settings page.

### Q8. How do you run a design-system governance model across multiple product teams?

### Q9. How do you evaluate whether a UI change actually worked?

### Q10. Where do you push back on a design that's beautiful but expensive to build or maintain, and how do you have that conversation?

---

## 5. Shift-Left Testing (unit, mutation, contract, component/Playwright)

### Q1. What is your test strategy philosophy, and how do you decide the shape of the suite for a given product?

### Q2. How do you make mutation testing viable in a real CI pipeline?

### Q3. Explain consumer-driven contract testing end to end, including the CI workflow and `can-i-deploy`.

### Q4. Playwright: explain the auto-waiting model, locator strategy, and the trace viewer.

### Q5. How do you handle test data and environment isolation for E2E at scale?

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

### Q7. How do you keep an E2E suite from becoming a 90-minute flaky tax on every merge?

### Q8. How do you test things that are genuinely hard — animations, drag-and-drop, canvas, real-time/websockets, third-party iframes?

### Q9. How do you shift left beyond tests — what else belongs in the developer's local loop and the PR gate?

### Q10. Your team has 85% coverage, a green pipeline, and production incidents every sprint. Diagnose.

---

## 6. Coding Standards

### Q1. How do you decide what belongs in a lint rule versus a code review versus a written guideline?

### Q2. How do you enforce architectural boundaries in a large frontend monorepo?

### Q3. What is your review philosophy on large PRs, and what do you do when you receive one?

### Q4. How do you handle a disagreement with a senior colleague over a code standard?

### Q5. What does "definition of done" include on your team, and why each item?

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

### Q7. How do you manage dependencies and upgrades over a multi-year codebase lifetime?

### Q8. How do you write and maintain ADRs, and what actually belongs in one?

### Q9. How do you introduce standards to a team that resists them?

### Q10. How do you balance shipping speed against code quality when the business is pushing hard?

---

## 7. APIs and Backend Calls

### Q1. Explain HTTP caching precisely: `Cache-Control`, `ETag`, `Last-Modified`, and the difference between stale-while-revalidate and a normal revalidation.

### Q2. How do you design authentication for a SPA + API, end to end?

### Q3. What is a BFF and when is it the right architecture?

### Q4. Explain idempotency keys, optimistic concurrency, and how you'd prevent a double-charge.

### Q5. Compare polling, long-polling, SSE, and WebSockets. How do you choose?

### Q6. How do you version a public API, and how do you deprecate an endpoint without breaking clients?

### Q7. Walk through what happens between typing a URL and the page rendering, focusing on where latency hides.

### Q8. How do you make a frontend resilient when a downstream API is slow or degraded?

### Q9. How would you architect data fetching and caching for a large app — normalization, invalidation, optimistic updates, offline?

### Q10. A user reports "the app is slow" but your API p50 is 80ms. How do you investigate?
