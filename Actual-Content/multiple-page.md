# Testing Pages Independently During Parallel Development

## The Problem

In a multi-page flow, teams often build pages in parallel rather than in sequence. In our running example — Student List (Page 1) → Edit Form (Page 2) → Thank You (Page 3) — Page 1 and Page 3 can be actively developed and tested while Page 2 is still being scoped. Writing tests the naive way, by navigating through the full journey (Student List, click a student, fill the form, submit, land on Thank You), doesn't work here, because step two of that journey doesn't exist yet.

The tests for Page 1 and Page 3 need to run independently of each other and independently of Page 2, without waiting for the rest of the flow to be built.

To be clear about scope: this isn't a plan to ship without Page 2. Business requirements for Page 2 are still being discussed, but the app always ships with all three pages complete — the stub described below exists only to unblock Page 1 and Page 3 work during this window, and gets removed once Edit Form is real.

## Why a Standard Playwright Test, Not Component Testing

Playwright's experimental component testing (`@playwright/experimental-ct-react`) mounts a single component in isolation, outside a real browser navigation context. In practice this makes it equivalent to a unit test with a real DOM renderer underneath — it verifies a component in a vacuum, the same thing tools like React Testing Library already do, just running through Playwright's runner. It was considered and dropped for this use case: it doesn't solve the actual problem, which is page-to-page dependency, not component isolation. A full page still needs a real route, a real DOM, and — for Page 3 — a real (or convincingly faked) predecessor.

Instead, each page gets a standard `@playwright/test` spec that navigates straight to that page's own route with `page.goto()`. This works today for Page 1 and Page 3 regardless of whether Page 2 exists, because neither test drives through Page 2 to get there.

## The Two Kinds of Data a Page Depends On

Once a page is opened directly, it can depend on data in two different ways, and they need to be handled differently:

**API data.** Both Student List and Thank You may call an API to render themselves. This is the easy case — mock it with `page.route()` against an agreed-upon response shape, checked into the repo as a fixture.

**Inter-page context.** Thank You also depends on something that, in the real flow, only exists because the user passed through Edit Form first — for example, which student was just edited. In a real browser test, this context is normally built up as React state during client-side navigation. If you `goto()` straight to Thank You, that in-memory context was never built, because Page 2 never ran. This is the harder case, and the one this document is really about.

## Mocking the Missing Page: A Stub Page 2, Not a Playwright Injection

Before Playwright ever runs, whoever builds Page 3 already has to solve "what does context contain with no Page 2" just to see the page render locally in a browser. Whatever mechanism they use for that has to exist independent of any test — so the mock belongs in real app code, not in the test layer.

Build a minimal stub Page 2 — not the real Edit Form, just enough to produce a real context object and perform the real navigation. Concretely, the stub has two buttons, each hardcoded to a different dummy value, each calling the same real context-creation function and navigating to Thank You:

```tsx
// EditFormStub.tsx
export const EditFormStub = () => {
  const goNext = (data: EditFormSubmission) => {
    setGlobalContext(data);   // the real, shared context-creation function
    navigate('/thank-you');
  };

  return (
    <>
      <button onClick={() => goNext({ studentName: 'Asha' })}>Next (scenario X)</button>
      <button onClick={() => goNext({ studentName: 'Ben' })}>Next (scenario Y)</button>
    </>
  );
};
```

Playwright drives this like any other page: navigate to the stub, click the button for the scenario under test, land on Thank You, assert the text that scenario should produce.

```ts
await page.goto('/edit-form');
await clickNextForScenarioStep(page, 'X');
await verifyThankYouStep(page, 'text for scenario X');
```

```ts
await page.goto('/edit-form');
await clickNextForScenarioStep(page, 'Y');
await verifyThankYouStep(page, 'text for scenario Y');
```

This is a real, deliberate trade against a pure deep-link: Thank You's test now depends on the stub's route existing. In exchange, there is exactly one mock in the whole system, defined in real app code, exercised identically whether a developer opens the browser to check Thank You's UI or Playwright runs the same click. Nothing about "what context looks like" is invented separately by the test.

Two hardcoded buttons is the simplest version of this and is the right call for a small, fixed number of known scenarios — it reads clearly in the test report and needs no state or query-param plumbing. If the number of scenarios grows past a handful, or the value needs to vary continuously rather than by a small set of cases, a single input field bound to state (filled by Playwright before clicking one Next button) scales better — but that's an optimization to reach for later, not a starting requirement.

## Folder Structure

Tests are organized by page, not by flow, so that locators, actions, and steps for a given page all live together:

```
tests/
├── locators/
│   ├── shared/
│   ├── studentList/
│   ├── editForm/      (stub for now — two-button version)
│   └── thankYou/
├── actions/
│   ├── studentList/
│   ├── editForm/
│   └── thankYou/
├── steps/
│   ├── studentList/
│   ├── editForm/
│   └── thankYou/
├── fixtures/
│   └── api/
└── specs/
    ├── studentList.spec.ts
    ├── editForm.spec.ts
    └── thankYou.spec.ts
```

Locators are pure element references. Actions are a factory per page that computes locators once and returns reusable functions. Steps compose actions into named, reusable units, parameterized by data rather than duplicated per scenario. Specs consume only steps and actions — never raw locators — and are where flow-specific ordering happens, if and when a flow spec is written.

`editForm/` exists from day one even though the real form doesn't yet — it holds the stub's locators (the two buttons), actions, and steps. There's no separate `fixtures/context/` folder anymore: the dummy values the buttons pass are defined in the stub component itself, in application source, not in a test fixture.

## Key Takeaways

- Test each page independently by navigating straight to its own route, not by walking the full journey.
- API dependencies are mocked with `page.route()` against agreed fixtures.
- Inter-page context is not mocked by Playwright injecting a `window` global — that creates a second, test-owned mock that can drift from whatever the app already needs for local development.
- Instead, a minimal stub Page 2 produces the missing context through real code: two hardcoded buttons, each calling the real context-creation function with a different value and performing the real navigation.
- Playwright drives the stub like any other page — click the button for the scenario under test, land on Thank You, assert the corresponding text.
- The shared shape of the handoff lives in application source as a TypeScript interface, used by the stub today and by the real Edit Form later — one definition, not a duplicated guess.
- A page's test only asserts that page's behavior — cross-page navigation is checked by URL, not by asserting the next page's content.
- When the real Edit Form ships, it replaces the stub's buttons with real fields feeding the same context-creation function — the mechanism doesn't change, only what's fed into it does. A full-journey test should be added at that point to verify the real handoff.