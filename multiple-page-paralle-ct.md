PROMPT
I need a Confluence page written for my team explaining how we test Page 1 and Page 3 of a flow in parallel while Page 2 is still being scoped by the business — write it ready to paste into Confluence (clear headings, code blocks where useful, minimal bullet use except for a closing takeaways list). Use TypeScript with arrow functions only — no classes, no constructors — in every code example.

Use the Student List → Edit Form → Thank You application as the running example throughout:
* Page 1 — Student List: shows up to 5 students in a class; clicking a student navigates to Edit Form with that student's data prefilled.
* Page 2 — Edit Form: still being scoped by the business, not yet built.
* Page 3 — Thank You: confirmation screen, no data is actually saved; the confirmation text shown varies depending on the data it receives.

Points to cover, in this order:

1. The problem — Page 1 and Page 3 need to be built and tested now; Page 2 is still being discussed. Testing the naive way (navigate the full journey start to finish) doesn't work because the middle of that journey doesn't exist. State clearly that this is a temporary parallel-development scaffold, not a plan to ship without Page 2 — production always ships with all three pages complete, and whatever stands in for Page 2 during development gets removed once it's real.

2. Why a standard Playwright test, not `@playwright/experimental-ct-react` — explain that experimental CT mounts a component in isolation outside real navigation, which makes it functionally equivalent to a unit test with a real DOM renderer (the same thing React Testing Library already does), and that it doesn't solve the actual problem here, which is page-to-page dependency, not component isolation. Each page instead gets a normal spec that `page.goto()`s straight to its own route.

3. The two kinds of data a page depends on — API data (mocked with `page.route()` against agreed fixtures) versus inter-page context (state a preceding page would normally hand off, which doesn't exist in memory if you deep-link past that page).

4. Why the inter-page context isn't mocked by having Playwright inject it — explain and reject a `window`-global/`addInitScript()` approach: before Playwright ever runs, whoever builds the later page already has to solve "what does context look like with no preceding page" just to render it locally, so that mock has to live in real app code anyway; a second, Playwright-owned mock on top of that is a duplicate that can drift from it.

5. The stub-page solution — a minimal stub for the missing page that does nothing but produce a real context object via the real, shared context-creation function and perform the real navigation. Cover the two-hardcoded-buttons technique specifically: each button passes a different fixed value and calls the same function, so two CT scenarios can be exercised (differing downstream Thank You text) without state or query-param plumbing. Note this is the right call for a small, fixed number of known scenarios, and that a single input field bound to state is the better choice only once the number of scenarios grows past a handful.

6. Folder structure — grouped by page, matching our POM convention (`locators/`, `actions/`, `steps/`, `specs/`, each subdivided by page name). Show the exact tree for this example, including the stub page's folder, and note there's no separate context-fixture folder since the mock values now live in the stub component itself.

Format notes:
* Every code example uses arrow functions, never `class`/`constructor`.
* Do not include a section on contract testing or Pact — that approach was considered separately and explicitly dropped for this problem.
* Do not include a "what changes once the real page ships" section as a separate heading — if worth mentioning, fold it briefly into the stub-page section instead.
* End with a short "Key Takeaways" recap list.