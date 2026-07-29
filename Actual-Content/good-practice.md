# Playwright + React Testing Best Practices

This page covers four practices we've settled on for testing our React applications with Playwright: locator strategy, handling async React state, selective endpoint mocking, and using `test.step`. All examples use TypeScript with arrow functions only — no `class`, no `constructor`.

## Application Used as the Example

Every section below reuses the same three-page flow, so the app is described once here — no section repeats these details, it just applies them.

* **Page 1 — Student List:** shows up to 5 students in a class. Clicking a student navigates to Edit Form with that student's data prefilled.
* **Page 2 — Edit Form:** prefilled fields are `firstName`, `lastName`, `age`, `class` (dropdown), and `classTeacherName` — all mandatory. The Next button is always enabled; submitting with a missing mandatory field shows a top-of-form error banner instead of disabling the button. Navigating back to Student List and re-selecting the same student always resets the form to its original prefilled values — there's no unsaved-edit persistence.
* **Page 3 — Thank You:** confirmation screen; no data is actually saved. The confirmation text shown varies depending on the data it receives.

With that context in hand, each section below can be read on its own.

---

## 1. Locator Strategy

Not all locators are equal. Some survive a redesign, some break the moment a class name changes. Use this priority order, highest first:

| Locator | Use case | Example |
|---|---|---|
| `getByRole` | Best for accessibility & user-facing elements | `page.getByRole('button', { name: 'Submit' })` |
| `getByText` | Match visible text | `page.getByText('Login')` |
| `getByLabel` | Form inputs with labels | `page.getByLabel('Email')` |
| `getByPlaceholder` | Inputs without labels | `page.getByPlaceholder('Enter email')` |
| `getByTestId` | Stable test selectors | `page.getByTestId('submit-btn')` |
| CSS (`locator`) | Flexible, structural targeting | `page.locator('.btn-primary')` |
| XPath | Complex DOM navigation | `page.locator('//div[@id="app"]')` |

Prefer the top of the table and only drop down when the element genuinely has no accessible role, text, or label. CSS and XPath are a last resort — they couple the test to markup structure rather than user-facing behavior.

Locators are grouped one file per page, and each file is a pure factory: no actions, no assertions, just element references. This keeps them reusable across every flow that touches that page.

```ts
// locators/studentList/studentList.locators.ts
export const studentListLocators = (page: Page) => ({
  studentRow: (name: string) => page.getByRole('button', { name }),
  studentList: () => page.getByRole('list', { name: 'Students' }),
});
```

```ts
// locators/editForm/editForm.locators.ts
export const editFormLocators = (page: Page) => ({
  firstName: () => page.getByRole('textbox', { name: 'First Name' }),
  lastName: () => page.getByRole('textbox', { name: 'Last Name' }),
  classDropdown: () => page.getByRole('combobox', { name: 'Class' }),
  errorBanner: () => page.getByRole('alert'),
  nextButton: () => page.getByRole('button', { name: 'Next' }),
});
```

Targeting `studentRow('Aditi Sharma')` survives a markup change; targeting `.student-row:nth-child(2)` doesn't.

---

## 2. Handling Async / React State Updates

React state changes aren't synchronous with the click or event that triggers them. The one rule that never changes: **don't perform a hard wait** (`page.waitForTimeout(...)`). Instead, pick the wait that matches what the app is actually doing — network, DOM, or navigation.

```ts
// after clicking a student row on Student List
await page.getByRole('button', { name: 'Aditi Sharma' }).click();
await expect(page).toHaveURL(/\/edit-form/);
await expect(editFormLocators(page).firstName()).toHaveValue('Aditi'); // auto-retries, no sleep needed
```

### Network-Based Waiting

**`page.waitForResponse()` / `page.waitForRequest()`**

Use it when:

* Intercepting specific endpoints — confirming `PUT /api/student` returns `200` after submitting the Edit Form.
* Verifying data payloads — capturing the exact JSON returned by an API to assert on its accuracy.
* Testing file downloads/uploads — waiting for a heavy upload endpoint to finish processing before moving on.

```ts
// confirm the Edit Form's submit call actually succeeded before checking Thank You
await Promise.all([
  page.waitForResponse((res) => res.url().includes('/api/student') && res.status() === 200),
  editFormLocators(page).nextButton().click(),
]);
```

**`page.waitForLoadState('networkidle')`**

Use it when:

* Dashboard rendering — a page fires dozens of separate API calls to populate charts and metrics.
* SPAs — ensuring the client-side framework has finished fetching initial background data after login.
* Third-party scripts — waiting for fonts, tracking pixels, and chat widgets to settle so they don't cause layout shifts. Use sparingly — it's the least targeted network wait.

```ts
// e.g. a future analytics dashboard that fires many background calls on load
await page.goto('/dashboard');
await page.waitForLoadState('networkidle');
```

### Element & DOM-Based Waiting

**`expect(locator).toBeVisible()` (web-first assertion)**

Use it when:

* Dynamic element injection — the Edit Form's error banner or a success notification only appears after a submit finishes.
* Page transitions — a skeleton loader disappears and the real content is fully visible before you interact with it.
* Lazy-loaded content — scrolling an infinite feed and waiting for the next batch of cards to load.

```ts
// wait for the error banner to appear rather than assuming it's already there
await expect(editFormLocators(page).errorBanner()).toBeVisible();
```

**`locator.waitFor({ state: 'attached' | 'detached' | 'hidden' })`**

Use it when:

* Spinner disappearance — waiting for a loading spinner to become `detached` or `hidden` so it no longer blocks the UI.
* Multi-step wizards — waiting for Step 1's elements to be fully `detached` before interacting with Step 2's inputs.

```ts
// wait for a loading spinner to leave the DOM before asserting on Thank You text
await page.getByTestId('loading-spinner').waitFor({ state: 'detached' });
```

### Navigation & Lifecycle Waiting

**`page.goto(url, { waitUntil: 'domcontentloaded' })`**

Use it when:

* Fast HTML parsing — you only need the raw HTML structure to immediately scrape text, ignoring images, CSS, or scripts.

```ts
// scrape the rendered student names without waiting on images/CSS
await page.goto('/student-list', { waitUntil: 'domcontentloaded' });
```

**`page.goto(url, { waitUntil: 'load' })`**

Use it when:

* Standard page loading — traditional multi-page apps where you need the page, styles, and images fully loaded before acting. This is Playwright's default.

```ts
// standard navigation to Student List before interacting with any row
await page.goto('/student-list', { waitUntil: 'load' });
```

The thing you wait on should always be one of: a UI assertion (state you expect to be true), a real request/response, or an explicit DOM/navigation lifecycle state — never an arbitrary sleep.

---

## 3. Selective Endpoint Mocking

Default to mocking every endpoint a page depends on. The only exception is an endpoint that's going to return the same result every time, regardless of scenario — that one is safe to leave unmocked, since there's no scenario-specific variance it could get wrong.

**Endpoints that can be left unmocked** — the `class` dropdown options on the Edit Form, a country list, anything that doesn't vary per scenario or per test. If you'd still rather remove the network dependency entirely, a single shared fixture is enough since the result never changes:

```ts
// mocks/reference-data.mock.ts
export const mockReferenceData = (page: Page) =>
  page.route('**/api/classes', (route) =>
    route.fulfill({ json: ['Grade 1A', 'Grade 1B', 'Grade 2A'] }),
  );
```

**Everything else must be mocked.** This includes every journey-critical endpoint — whatever produces the Thank You confirmation text, or returns a student's saved data — and it needs a fixture per scenario, because the returned value is the thing under test:

```ts
// mocks/thankYou.mock.ts
export const mockThankYouContext = (page: Page, variant: 'updated' | 'unchanged') =>
  page.route('**/api/confirmation', (route) =>
    route.fulfill({
      json: { message: variant === 'updated' ? 'Details updated' : 'No changes made' },
    }),
  );
```

Under-mocking a journey-critical endpoint reintroduces flakiness, because the test now depends on real backend state instead of a controlled fixture. The rule: mock everything by default; the only opt-out is an endpoint proven to always return the same result.

### Why the test must own the mock, not borrow the application's

A specific way under-mocking creeps in: while the Edit Form is being built, the real `/api/student` endpoint isn't ready yet, so the component ships with a temporary fallback so developers have something to look at:

```ts
// EditForm.tsx — a dev-time placeholder, never intended as a contract
const student = apiResponse ?? { firstName: 'Aditi', lastName: 'Sharma', age: 10, class: 'Grade 1A' };
```

If the Playwright test never mocks `/api/student` itself and just asserts on whatever renders, it's really asserting on this fallback:

```ts
// no page.route() here — the test is unknowingly relying on the app's own placeholder
await page.goto('/edit-form/1');
await expect(editFormLocators(page).firstName()).toHaveValue('Aditi'); // passes, but why?
```

It passes today because the fallback happens to say `Aditi`. The test gets merged, everyone moves on.

A month later, someone else is working in that same part of the app, has no idea a test is silently depending on that fallback value, and changes it to `Rohan` for their own reasons. The application is still correct — that value was never a contract. But the old test now fails, and it fails for a reason completely disconnected from any real regression: the app's internal scaffolding moved, and the test happened to be standing on it.

That's the test not being independent — its result is coupled to an implementation detail nobody agreed to preserve. The fix is the same rule already stated: the test mocks the endpoint itself, explicitly, regardless of whatever fallback or placeholder exists in app code:

```ts
await mockStudentResponse(page, { firstName: 'Aditi', lastName: 'Sharma', age: 10, class: 'Grade 1A' });
await page.goto('/edit-form/1');
await expect(editFormLocators(page).firstName()).toHaveValue('Aditi'); // passes because of a fixture we own
```

Now the assertion is pinned to a fixture the test controls. The in-app placeholder can change or disappear the day the real endpoint ships, and this test won't notice or care.

This only covers API data. It doesn't cover inter-page context — state a preceding page would normally hand off in memory (for example, whatever the Edit Form would pass to Thank You). That's a different kind of dependency and isn't solved by `page.route()`; it needs its own handling at the page/navigation level.

---

## 4. Using `test.step`

Wrap logical chunks of a test in `test.step()` so a run's report reads as a named, traceable story instead of a wall of raw actions. It costs nothing beyond a wrapper and pays for itself the first time a test fails in CI.

Why it's worth using:

* **Pinpoint failures instantly** — the HTML report highlights the exact step that failed and collapses the passing ones, so you're not scanning dozens of raw lines to find where things broke.
* **Readable code as documentation** — step names describe intent ("Edit student details", "Submit Edit Form") in a way a PM or manual QA tester can follow, and unlike a separate doc, they can't drift out of sync with the code.
* **Nested steps** — a macro-step like "Submit Edit Form" can contain micro-steps like "fill each mandatory field," without cluttering the top-level test.
* **Automatic tracing** — Playwright's trace viewer and UI mode group network calls, console logs, and screenshots under the step they belong to, cutting debugging time.
* **One robust end-to-end test beats five fragile single-purpose specs** written just to get granular reporting — `test.step` gets you that granularity inside a single spec.

Where it belongs, given our steps layer: `test.step()` lives inside the step function itself, not wrapped again around the call site in the spec. That way every spec that imports the step automatically gets a named, traceable entry in the report — nobody has to remember to add the wrapper each time it's used.

```ts
// steps/editForm/editStudentDetails.step.ts
export const editStudentDetailsStep = (page: Page, fieldUpdates: Partial<StudentFields>) =>
  test.step('Edit student details on Edit Form', async () => {
    const actions = editFormActions(page);
    for (const [field, value] of Object.entries(fieldUpdates)) {
      await actions.updateField(field, value);
    }
  });
```

```ts
// specs/editForm.spec.ts
test('shows error banner when a mandatory field is missing', async ({ page }) => {
  await page.goto('/edit-form/1');
  await editStudentDetailsStep(page, { firstName: '' }); // reported as its own named step, no extra wrapping needed
  await submitEditFormStep(page);
  await expect(editFormLocators(page).errorBanner()).toBeVisible();
});
```

Because the `test.step()` wrapper lives in the step, not the spec, `endToEndFlow.spec.ts` composing `selectStudentStep → editStudentDetailsStep → submitEditFormStep → verifyThankYouStep` gets four clearly named entries in the trace and report automatically — with zero extra wrapping at the call site.

---

## Key Takeaways

* Prefer `getByRole`, `getByText`, `getByLabel`, and `getByPlaceholder` over `getByTestId`, and reserve CSS/XPath selectors for cases nothing else covers.
* Locators are pure, page-scoped factories — no actions or assertions baked in.
* Never wait on a fixed timer; wait on a UI assertion or a real request/response.
* Default to mocking every endpoint; only leave unmocked the ones that always return the same result regardless of scenario.
* Mock journey-critical endpoints per scenario, since their return value is what the test verifies.
* Selective mocking covers API data only — inter-page context is a separate problem.
* Wrap logical chunks of a test in `test.step()`, defined inside the step function itself, so reports stay readable and every spec gets traceable steps for free.