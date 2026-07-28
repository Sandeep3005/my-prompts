# Prompt: Generate Confluence Page — Playwright + React Best Practices

## PROMPT

I need a Confluence page written for my team covering our Playwright + React testing best practices. Write it ready to paste into Confluence: clear headings, code blocks for every example, a table where noted below, and minimal bullet use except inside "Use it when" lists and the closing takeaways. Use TypeScript with **arrow functions only — no classes, no constructors** anywhere in the examples.

Use the **Student List → Edit Form → Thank You** application as the running example throughout — every section should illustrate its point using this app, not a generic/unrelated example.

### Application Used as the Example

* **Page 1 — Student List:** shows up to 5 students in a class; clicking a student navigates to Edit Form with that student's data prefilled.
* **Page 2 — Edit Form:** prefilled fields — `firstName`, `lastName`, `age`, `class` (dropdown), `classTeacherName`. All fields mandatory. Missing a mandatory field on submit shows a top-of-form error banner (not a disabled button).
* **Page 3 — Thank You:** confirmation screen, no data is actually saved; the confirmation text shown varies depending on the data it receives.

### Points to Cover, in This Order

**1. Locator Strategy**

Include this exact priority table:

| Locator | Use case | Example |
|---|---|---|
| `getByRole` | Best for accessibility & user-facing elements | `page.getByRole('button', { name: 'Submit' })` |
| `getByText` | Match visible text | `page.getByText('Login')` |
| `getByLabel` | Form inputs with labels | `page.getByLabel('Email')` |
| `getByPlaceholder` | Inputs without labels | `page.getByPlaceholder('Enter email')` |
| `getByTestId` | Stable test selectors | `page.getByTestId('submit-btn')` |
| CSS (`locator`) | Flexible, structural targeting | `page.locator('.btn-primary')` |
| XPath | Complex DOM navigation | `page.locator('//div[@id="app"]')` |

Explain the priority order (role/text/label/placeholder before testid, testid before CSS/XPath — CSS and XPath are a last resort because they couple tests to markup structure). Show locators as pure, page-scoped factory functions (one file per page, no actions/assertions inside) using `studentListLocators(page)` and `editFormLocators(page)` as the examples.

**2. Handling Async / React State Updates**

Open with the rule: never use a hard wait (`page.waitForTimeout(...)`); always pick the wait that matches what the app is actually doing. Include one intro example using a web-first assertion after clicking a student row and navigating to Edit Form.

Then organize into three subsections, each method as its own heading, followed by a "Use it when:" bullet list, followed by its own code example grounded in the Student List/Edit Form/Thank You app (not a generic example):

* **Network-Based Waiting**
  * `page.waitForResponse()` / `page.waitForRequest()` — use cases: intercepting a specific endpoint (e.g. confirming a save call returns 200), verifying a payload's accuracy, waiting for a heavy upload to finish.
  * `page.waitForLoadState('networkidle')` — use cases: dashboards firing many calls at once, SPAs finishing initial fetch after login, third-party scripts settling before layout is measured. Note it should be used sparingly since it's the least targeted network wait.
* **Element & DOM-Based Waiting**
  * `expect(locator).toBeVisible()` (web-first assertion) — use cases: dynamic element injection (error banner after submit), page transitions (skeleton loader replaced by content), lazy-loaded content (infinite scroll).
  * `locator.waitFor({ state: 'attached' | 'detached' | 'hidden' })` — use cases: spinner disappearance, multi-step wizards where Step 1 must fully detach before Step 2 renders.
* **Navigation & Lifecycle Waiting**
  * `page.goto(url, { waitUntil: 'domcontentloaded' })` — use case: only need raw HTML to scrape text, don't care about images/CSS/scripts.
  * `page.goto(url, { waitUntil: 'load' })` — use case: standard page loads needing styles/images in place before acting; this is Playwright's default.

Close the section with a one-line rule of thumb: the thing you wait on is always a UI assertion, a real request/response, or an explicit DOM/navigation lifecycle state — never an arbitrary sleep.

**3. Selective Endpoint Mocking**

State the rule precisely as: **default to mocking every endpoint a page depends on.** The only exception is an endpoint that returns the same result every time regardless of scenario — that one is safe to leave unmocked (or mocked once with a shared fixture purely to remove the network dependency, not because it needs scenario control). Use the Edit Form's `class` dropdown options as the example of an endpoint safe to leave unmocked, and the Thank You confirmation-text endpoint (and student save endpoint) as endpoints that must always be mocked per scenario, since their return value is the thing under test. Include code for both.

Add a subsection titled **"Why the test must own the mock, not borrow the application's"** telling this specific story: while a page is being built, the real endpoint isn't ready yet, so the component ships with a hardcoded dev-time fallback (e.g. `const student = apiResponse ?? { firstName: 'Aditi', ... }`). If a Playwright test never mocks that endpoint itself and just asserts on whatever renders, it's unknowingly asserting on the fallback, and the test passes. A month later, someone unrelated changes that fallback value for their own reasons — the app is still correct, the fallback was never a contract — but the old test now fails for a reason with nothing to do with a real regression. The lesson: the test must mock the endpoint itself, explicitly, regardless of any placeholder in app code, so its result is pinned to a fixture it owns. Show the "before" (no `page.route()`, relying on the fallback) and "after" (explicit `page.route()` mock) code side by side.

Close with one line noting this only covers API data — inter-page context (state a preceding page would hand off in memory) is a separate problem not solved by `page.route()`.

**4. Using `test.step`**

Explain why to wrap logical chunks of a test in `test.step()`, covering: pinpointing failures instantly in the HTML report (failed step highlighted, passing ones collapsed), step names functioning as living documentation that can't drift out of sync with the code, support for nested steps (a macro-step containing micro-steps), automatic grouping of network calls/console logs/screenshots in Playwright's trace viewer, and that one robust end-to-end test with `test.step` granularity beats splitting a journey into many fragile single-purpose spec files.

State the placement rule explicitly: `test.step()` lives **inside the step function itself**, not wrapped again around the call site in the spec — so every spec that imports the step automatically gets a named, traceable report entry with no extra wrapping needed at the call site. Show this with `editStudentDetailsStep` (containing the `test.step()` call) used from a spec, and close by noting that composing `selectStudentStep → editStudentDetailsStep → submitEditFormStep → verifyThankYouStep` in an end-to-end spec gets four clearly named report entries for free.

### Format Notes

* Every code example uses arrow functions, never `class`/`constructor`.
* Keep the locator priority table as an actual table; do not convert the wait-strategy methods into a table — those must be headings with "Use it when:" bullet lists and a code block per method, as specified above.
* Do not include a section on contract testing, Pact, or the missing-Page-2 stub-page problem — those are separate topics, out of scope for this page.
* End with a short "Key Takeaways" bullet list recapping all four sections, including the mock-everything-except-stable rule and the test.step placement rule.