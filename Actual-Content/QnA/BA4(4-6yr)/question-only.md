# Frontend Interview Bank — 4–6 Years Experience (Questions Only)

**Format:** 7 topics × 10 questions = 70 questions. Within each topic, questions go from simple (Q1) to tough (Q10).
**Note:** Code questions are "predict the output / spot the bug" style — no terminal needed.

*Answer key available in the companion `-WITH-ANSWERS.md` file.*

---

## 1. React

### Q1. What is the difference between state and props? When would you lift state up?

### Q2. What does the dependency array of `useEffect` control? What happens with `[]` vs no array vs `[dep]`?

### Q3. Why does React need a `key` on list items? What's wrong with using the array index?

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

### Q5. Explain `useMemo` vs `useCallback` vs `React.memo`. When do they actually help?

### Q6. What is the difference between controlled and uncontrolled components? Which do you prefer for a large form and why?

### Q7. What does this render? Explain the effect timing.

```jsx
useEffect(() => {
  console.log("A");
  return () => console.log("B");
}, [id]);
```
`id` changes from 1 → 2.

### Q8. What problem does `useRef` solve that `useState` doesn't? Give two real uses.

### Q9. A list re-renders every keystroke even though the items didn't change. Walk me through how you'd diagnose and fix it.

### Q10. Explain what happens on state update in React 18 — batching, and what "concurrent rendering" changes for you as a developer.

---

## 2. TypeScript

### Q1. What's the difference between `any`, `unknown`, and `never`?

### Q2. `interface` vs `type` — what are the practical differences?

### Q3. What's the difference between `unknown[]`, `any[]`, and a tuple `[string, number]`?

### Q4. What does this produce?

```ts
type A = { a: string };
type B = { b: number };
type C = A & B;
type D = A | B;
const x: C = { a: "1", b: 2 };
const y: D = { a: "1" };
```

### Q5. What are generics for? Write the signature of a function that returns the first element of an array, preserving the type.

### Q6. Explain the utility types `Partial`, `Pick`, `Omit`, `Record`, and `ReturnType`. Give one use for each.

### Q7. What is a discriminated union and why is it better than optional fields? Show a small example.

### Q8. What is type narrowing? Name four ways to narrow, and explain what a type guard function is.

### Q9. Why is this unsafe, and what does `strict` mode buy you?

```ts
const data = JSON.parse(res) as User;
```

### Q10. Explain `keyof`, `typeof`, and mapped types. What does this do?

```ts
type Getters<T> = { [K in keyof T & string as `get${Capitalize<K>}`]: () => T[K] };
```

---

## 3. JavaScript

### Q1. `==` vs `===`, and what is `null == undefined`?

### Q2. Explain `var` vs `let` vs `const` — scope, hoisting, and reassignment.

### Q3. What does this log?

```js
for (var i = 0; i < 3; i++) setTimeout(() => console.log(i), 0);
for (let j = 0; j < 3; j++) setTimeout(() => console.log(j), 0);
```

### Q4. What is a closure? Give a practical use.

### Q5. Order the output.

```js
console.log("A");
setTimeout(() => console.log("B"), 0);
Promise.resolve().then(() => console.log("C"));
queueMicrotask(() => console.log("D"));
console.log("E");
```

### Q6. What is `this` in each case?

```js
const obj = {
  name: "x",
  regular() { return this.name; },
  arrow: () => this?.name,
};
const fn = obj.regular;
```

### Q7. Explain shallow vs deep copy. What do `{...obj}`, `Object.assign`, and `structuredClone` do?

### Q8. What's the difference between `Promise.all`, `allSettled`, `race`, and `any`?

### Q9. What does this output, and why is it a classic bug?

```js
const arr = [1, 2, 3];
const result = arr.map(async (n) => n * 2);
console.log(result);
```

### Q10. Explain event delegation, bubbling vs capturing, and why `e.stopPropagation()` can be dangerous.

---

## 4. UI Design Principles

### Q1. What is visual hierarchy and which tools create it?

### Q2. What do proximity, alignment, and consistency (Gestalt/CRAP principles) mean in practice?

### Q3. Why does a design system use a spacing scale (4/8px) instead of arbitrary values?

### Q4. What are the WCAG contrast minimums, and why is colour alone not enough to convey state?

### Q5. Design the loading, empty, and error states for a data table. What does each need?

### Q6. What is affordance and feedback? Give an example of each being violated.

### Q7. How do you make a form accessible and forgiving? Walk through the specifics.

### Q8. Mobile-first vs desktop-first — what changes in how you build, and what are common responsive mistakes?

### Q9. A page feels "slow" even though the API responds in 200ms. What UI-level causes and fixes would you consider?

### Q10. How would you introduce a design system into an existing product with inconsistent UI? What do you do about migration?

---

## 5. Shift-Left Testing (unit, mutation, contract, component/Playwright)

### Q1. What does "shift left" mean, and why does catching a bug earlier matter?

### Q2. Explain the testing pyramid (or trophy). Where should the bulk of your tests live for a frontend app?

### Q3. What makes a good unit test? What's wrong with testing implementation details?

### Q4. What is code coverage, and why is 100% coverage not the goal?

### Q5. What is mutation testing? What does a "surviving mutant" tell you?

### Q6. What is contract testing and what problem does it solve that E2E doesn't?

### Q7. What is component testing in Playwright, and how does it differ from an E2E test and from a Testing Library unit test?

### Q8. How do you deal with flaky tests? Name concrete causes and fixes in Playwright.

### Q9. How do you decide what to mock? What's the risk of over-mocking?

### Q10. Design a testing strategy for a checkout flow. What runs on pre-commit, on PR, and nightly?

---

## 6. Coding Standards

### Q1. What is linting vs formatting? Why do you need both, and why automate them?

### Q2. What makes a good function or variable name? Rewrite `const d = getD(x)` for a date-difference calculation.

### Q3. What is DRY, and when is duplication actually the better choice?

### Q4. What makes a good pull request and a good code review comment?

### Q5. What makes a comment useful vs noise? When should code be self-documenting?

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

### Q7. How do you handle errors consistently across a codebase?

### Q8. How do you keep a shared codebase consistent as the team grows? What do you automate vs document?

### Q9. What is technical debt? How do you make the case to pay it down?

### Q10. What would you put in a frontend codebase's CONTRIBUTING/standards doc for a new team?

---

## 7. APIs and Backend Calls

### Q1. What do these status codes mean: 200, 201, 204, 400, 401, 403, 404, 409, 422, 429, 500, 503?

### Q2. GET vs POST vs PUT vs PATCH vs DELETE. What do idempotent and safe mean?

### Q3. What is CORS and what actually causes a preflight request?

### Q4. `fetch` doesn't throw on a 404. What does that mean for your error handling?

### Q5. How do you cancel an in-flight request, and why does it matter in React?

### Q6. Where do you store an auth token, and what are the tradeoffs?

### Q7. What is a debounce vs throttle, and how would you use each for API calls?

### Q8. Explain retries, exponential backoff, and when NOT to retry.

### Q9. How would you design the data-fetching layer for a dashboard with 6 widgets, each needing different data?

### Q10. REST vs GraphQL — what tradeoffs matter on the frontend? And what is over-fetching vs under-fetching?
