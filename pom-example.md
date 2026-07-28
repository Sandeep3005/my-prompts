# Prompt: Generate Confluence Page — POM Architecture (Playwright)

## PROMPT

I need a Confluence page written for my team covering our POM (Page Object Model) architecture for Playwright tests. Write it in a structure ready to paste into Confluence (clear headings, tables where useful, code blocks for examples). Use TypeScript with **arrow functions only — no classes, no constructors** anywhere in the examples.

Use the **Student List → Edit Form → Thank You** application (described below) as the running example throughout the page — don't just tack it on as a separate section at the end. Each architectural point should be explained *using* this example, so a reader learns the pattern and sees it applied at the same time.

### Application Used as the Example

3-page flow:
- **Page 1 — Student List:** shows up to 5 students in a class; clicking a student navigates to the Edit Form with that student's data prefilled
- **Page 2 — Edit Form:** prefilled fields — `firstName`, `lastName`, `age`, `class` (dropdown), `classTeacherName` (text field). All fields mandatory. Next button is **always enabled**. Clicking Next with a missing mandatory field shows a **top-of-form error banner** instead of disabling the button. Navigating back to Student List and re-selecting the same student **always resets to original prefilled values** — no unsaved-edit persistence.
- **Page 3 — Thank You:** confirmation screen, no data is actually saved (UI-flow-only)

### Points to Cover (in this order), explained via the Student List example

**1. The core problem with naive POM**
- Multi-page applications have a lot of elements to capture per page
- Elements get re-captured/duplicated across multiple test cases instead of reused once
- Need an approach where you don't recapture the same element every time you write a new test

**2. Shared vs page-specific locators**
- Some elements (e.g., a header/nav) appear on multiple pages — these go in a `locators/shared/` folder
- Elements unique to one page go in `locators/<pageName>/`
- Locators must be **pure** — no logic, no actions, no assertions, just element references
- Locators must be **flow-agnostic** — a locator has no knowledge of which flow/journey uses it

**3. Pages can belong to multiple flows — so POM must never be organized by flow**
- The same page (e.g., Edit Form) can be part of more than one flow in a real app
- If POM/locators were organized by flow, the same page would end up duplicated across flow folders
- Correct approach: POM structure is organized by **page**, never by flow; flow-level composition happens elsewhere (in specs, via steps — see point 6)

**4. Actions are page-scoped, not flow-scoped**
- An action like "fill the edit form" or "select a value in the class dropdown" is a property of the page's UI, not of any particular flow's intent
- Actions stay reusable across any flow that touches that page
- Negative-path/invalid-data scenarios (e.g., submitting with a missing mandatory field) don't need separate actions — the same action is called with different data

**5. Actions — factory pattern to avoid repeated locator instantiation**
- Problem: if every action function calls `xLocators(page)` internally, that line repeats across every function in the file
- Solution: a single factory function per page computes locators **once** and returns an object of arrow functions that close over those locators
- Example using the Edit Form: `editFormActions(page)` returns `{ getPrefilledValues, updateField, clearField, clickNext, getTopErrorMessage }` — all sharing one `locators` instance, no repetition, no classes

**6. Why a "Journey" orchestration layer was considered and rejected**
- Initially considered a separate layer to sequence multi-page flows
- Rejected because: for simple flows the abstraction is unnecessary overhead, and it doesn't fit the "actions are page-scoped, not flow-scoped" principle
- The real problem it was trying to solve — repeated/duplicated sequences across spec files — is better solved by point 7 below

**7. Steps layer — parameterized, not duplicated per scenario**
- A "step" composes one or more actions into a reusable, named unit (e.g., `editStudentDetailsStep`, `submitEditFormStep`)
- Critical rule: if two scenarios differ only by **data or a boolean/options toggle**, they should be **one parameterized step**, not separate files
- Example using the Edit Form: `editStudentDetailsStep(page, fieldUpdates)` accepts a partial object of field/value pairs — this single step covers "edit one field," "edit all fields," and "edit nothing," without needing `editOneField.step.ts`, `editAllFields.step.ts`, etc.
- Only split into separate step files when the scenarios differ by **actual behavior**, not just data
- `test.step()` belongs **inside** the step function itself (not wrapped again in the spec), so any spec importing that step automatically gets a named, traceable entry in the Playwright report

**8. Folder structure — grouped by page, not flat**
- A flat `steps/` or `actions/` folder becomes unmanageable once an app has 50+ pages
- Correct structure: group `locators/`, `actions/`, and `steps/` each by page name, so once you know the page you know where all three live; only `locators/shared/` stays flat
- Show this exact structure applied to the Student List example:

```
tests/
├── locators/
│   ├── shared/
│   ├── studentList/
│   ├── editForm/
│   └── thankYou/
├── actions/
│   ├── studentList/
│   ├── editForm/
│   └── thankYou/
├── steps/
│   ├── studentList/
│   ├── editForm/
│   └── thankYou/
└── specs/
    ├── studentList.spec.ts
    ├── editForm.spec.ts
    └── endToEndFlow.spec.ts
```

**9. Specs — consume steps/actions only**
- Specs never touch raw locators directly
- A flow is simply a spec calling multiple steps in the order that specific flow requires — no orchestration layer needed
- Show the Student List example: `endToEndFlow.spec.ts` composes `selectStudentStep` → `editStudentDetailsStep` → `submitEditFormStep` → `verifyThankYouStep`, while `editForm.spec.ts` reuses the same steps independently to test the missing-mandatory-field error banner scenario

### Format Notes
- Every code example must use arrow functions, never `class`/`constructor`
- Do not include any "Migrating a legacy locator file" section
- Do not include a Journey/orchestration layer as part of the final recommended architecture — only mention it in point 6 as something considered and rejected, with the reasoning
- End with a short "Key Takeaways" recap list