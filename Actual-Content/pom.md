# Our Playwright Testing Journey — From Chaos to Structure

## 1. The Application

Before diving into the "why," let's ground everything in a real example we'll use throughout this page.

**Student List → Edit Form → Thank You** — a 3-page flow:

| Page | Purpose | Key Behavior |
|---|---|---|
| Student List | Shows up to 5 students in the class | Clicking a student navigates to Edit Form with that student's data prefilled |
| Edit Form | Shows prefilled student details | 5 mandatory fields: First Name, Last Name, Age, Class (dropdown), Class Teacher Name. Next button is **always enabled** — a missing mandatory field shows a **top-of-form error banner** instead of disabling the button |
| Thank You | Confirmation screen | No data persistence — this is a UI-flow-only confirmation |

If a user re-selects the same student after navigating back, the form **always resets** to original prefilled values — no unsaved-edit persistence.

*(Full breakdown of this application is available in the companion reference doc.)*

---

## 2. The Problem — Writing Tests Without a Plan

Here's what happens when we write Playwright tests for this application **without any structure** — just opening the browser, using codegen or writing locators inline, one spec at a time.

```typescript
test('edit student and submit', async ({ page }) => {
  await page.goto('/students');
  await page.getByText('Rahul Verma').click();
  await page.waitForTimeout(1000); // wait for form to load
  await page.getByLabel('First Name').fill('Rahul');
  await page.getByLabel('Age').fill('16');
  await page.locator('.class-dropdown').click();
  await page.getByText('10A').click();
  await page.getByRole('button', { name: 'Next' }).click();
  await page.waitForTimeout(1000);
  await expect(page.getByText('Thank you!')).toBeVisible();
});

test('shows error when mandatory field missing', async ({ page }) => {
  await page.goto('/students');
  await page.getByText('Rahul Verma').click();
  await page.waitForTimeout(1000);
  await page.getByLabel('First Name').fill(''); // clear mandatory field
  await page.getByRole('button', { name: 'Next' }).click();
  await expect(page.getByText('Please fill all mandatory fields')).toBeVisible();
});

test('edit teacher name and submit', async ({ page }) => {
  await page.goto('/students');
  await page.getByText('Priya Shah').click();
  await page.waitForTimeout(1000);
  await page.getByLabel('Class Teacher Name').fill('Mrs. Iyer');
  await page.getByRole('button', { name: 'Next' }).click();
  await page.waitForTimeout(1000);
  await expect(page.getByText('Thank you!')).toBeVisible();
});
```

**What's wrong here?**

| Problem | Where You See It |
|---|---|
| **Duplicated locators** | `page.getByText('Rahul Verma')`, the Next button, the Thank You text — all typed out fresh, in every test, by whoever happened to write it |
| **Duplicated sequences** | "Click student → wait → interact with form → click Next → wait" is repeated in all 3 tests, nearly identical |
| **Inconsistent locator strategy** | Mixing `getByText`, `getByLabel`, `getByRole`, and a raw CSS class (`.class-dropdown`) with no rule for which to use when |
| **Hard waits** | `waitForTimeout(1000)` sprinkled in, guessing how long the UI needs — flaky if the app is ever slower, wasteful if it's faster |
| **No structure for reuse** | If the "select a student and reach the edit form" sequence is needed by a 4th test tomorrow, it gets copy-pasted again |
| **Hard to debug failures** | If this test fails, the report just says "test failed" — no indication of which part (selecting student? filling form? submitting?) broke |

With 3 tests, this is annoying. With 50 pages and 500 tests, this becomes unmaintainable — nobody can safely change a locator without hunting through dozens of files.

### Folder Structure So Far

There isn't one. Everything — element lookups, waits, sequences, assertions — lives inline, directly inside a single spec file:

```
tests/
└── specs/
    └── editForm.spec.ts   ← everything lives here: locators, waits, sequences, assertions, all mixed together
```

This is the starting point we're moving away from.

---

## 3. The Destination — What We're Aiming For

Here's the **same three scenarios**, after applying the structure we'll walk through in this page:

```typescript
// specs/editForm.spec.ts
import { test } from '@playwright/test';
import { selectStudentStep } from '../steps/studentList/selectStudent.step';
import { editStudentDetailsStep } from '../steps/editForm/editStudentDetails.step';
import { submitEditFormStep } from '../steps/editForm/submitEditForm.step';
import { verifyThankYouStep } from '../steps/thankYou/verifyThankYou.step';

test('edit student and submit successfully', async ({ page }) => {
  await page.goto('/students');
  await selectStudentStep(page, 'Rahul Verma');
  await editStudentDetailsStep(page, { age: '16', class: '10A' });
  await submitEditFormStep(page);
  await verifyThankYouStep(page);
});

test('shows error when mandatory field missing', async ({ page }) => {
  await page.goto('/students');
  await selectStudentStep(page, 'Rahul Verma');
  await editStudentDetailsStep(page, { firstName: '' });
  await submitEditFormStep(page); // stays on Edit Form, shows error banner
});

test('edit teacher name and submit', async ({ page }) => {
  await page.goto('/students');
  await selectStudentStep(page, 'Priya Shah');
  await editStudentDetailsStep(page, { classTeacherName: 'Mrs. Iyer' });
  await submitEditFormStep(page);
  await verifyThankYouStep(page);
});
```

**What changed?**
- No raw locators anywhere in the spec
- No hard waits — each step handles its own waiting internally, using proper condition-based waiting
- No duplicated sequences — each step is defined once, reused across all 3 tests
- Every step name shows up individually in the Playwright report/trace, via `test.step()`
- Editing any single field is just a different argument to `editStudentDetailsStep()` — not a new test file

This is the target. The rest of this page explains **how** we get from the chaos in Section 2 to the clarity in Section 3.

### Folder Structure So Far

The spec now imports from a `steps/` folder instead of containing everything inline. We haven't unpacked what's inside `steps/` yet — that's Section 4 — but the shape is already emerging:

```
tests/
├── steps/
│   └── (contents explained in Section 4)
└── specs/
    └── editForm.spec.ts   ← now just a sequence of step calls, no raw logic
```

---

## 3A. A Half-Measure — The "One Big Helper File" Trap

It's worth calling this out explicitly, because it's an easy trap to fall into: you can get a spec that *looks* exactly like Section 3 — clean, readable, no raw locators — while still doing it the wrong way underneath, by putting everything into **one large helper file**.

```typescript
// helpers.ts — everything, for every page, in one file
export const selectStudentStep = async (page, name) => { /* ... */ };
export const editStudentDetailsStep = async (page, fields) => { /* ... */ };
export const submitEditFormStep = async (page) => { /* ... */ };
export const verifyThankYouStep = async (page) => { /* ... */ };
export const selectContactFormFields = async (page, data) => { /* ... */ };
export const mockCountryListApi = async (page) => { /* ... */ };
// ...and 40 more functions, for every page in the app, all in this one file
```

The spec importing from this file looks identical to the "good" version — which is exactly what makes this trap easy to miss in a quick review.

**Why this is still a problem, even though the spec looks clean:**

| Problem | Why It Hurts |
|---|---|
| **No separation of concerns** | Locators, page interactions, and multi-step sequences are all mixed together in one file — you can't tell what's a pure element reference vs. a composed flow step just by looking at an import |
| **Merge conflicts** | Every team editing any page's test logic is editing the *same file* — constant conflicts as the app grows |
| **No ownership boundary** | Nobody "owns" this file; it becomes a dumping ground where nobody wants to clean up someone else's addition |
| **Doesn't scale** | A 50-page application means a helper file with hundreds of exports — impossible to navigate, search, or safely refactor |
| **Locators aren't reusable on their own** | If a locator is buried inside a helper function, another test can't reuse *just* the element reference without also getting the behavior wrapped around it |

**The fix is exactly the four-layer split described in Section 4** — `locators/`, `actions/`, `steps/`, grouped by page. The clean spec in Section 3 is only genuinely "done" once the code behind it is also properly separated — not just hidden inside one large file that happens to export nicely-named functions.

---

## 4. How We Get There — Reading Top-Down

We'll explain this the same way you'd actually read the codebase: start at the spec (what you see first), then peel back one layer at a time to see how it's built.

```
Spec  →  Steps  →  Actions  →  Locators
(flow)   (sequence) (interaction) (element)
```

### 4.1 Spec — Represents the Flow

**The spec file *is* the flow.** The spec's body, read top to bottom, is the sequence of operations for that scenario:

```typescript
test('edit student and submit successfully', async ({ page }) => {
  await page.goto('/students');
  await selectStudentStep(page, 'Rahul Verma');
  await editStudentDetailsStep(page, { age: '16', class: '10A' });
  await submitEditFormStep(page);
  await verifyThankYouStep(page);
});
```

A spec's job is to decide **what order things happen in** for this scenario. It has no idea *how* `editStudentDetailsStep` fills the form — that's intentionally hidden from it.

### 4.2 Steps — Reusable, Named, Parameterized Sequences

A step composes one or more actions into a single reusable unit that a spec can call by name:

```typescript
// steps/editForm/editStudentDetails.step.ts
export const editStudentDetailsStep = async (
  page: Page,
  fieldUpdates: Partial<{ firstName: string; lastName: string; age: string; class: string; classTeacherName: string }>
) => {
  const { updateField } = editFormActions(page);

  return test.step('Edit student details', async () => {
    for (const [field, value] of Object.entries(fieldUpdates)) {
      await updateField(field, value);
    }
  });
};
```

**Two rules that matter here:**

- **`test.step()` lives inside the step function**, not re-wrapped in the spec. This means every spec that calls `editStudentDetailsStep()` automatically gets a labeled, traceable entry in the report — for free.
- **Parameterize instead of duplicating.** Notice `editStudentDetailsStep` takes *any combination* of fields as an argument. This single step covers "edit one field," "edit all fields," and "clear a mandatory field" — we don't create `editFirstName.step.ts`, `editAge.step.ts`, `clearMandatoryField.step.ts` separately. If two scenarios differ only by *data*, they should be one parameterized step, not multiple files. Separate step files are only justified when the *behavior* genuinely differs, not just the data.

### 4.3 Actions — Page-Scoped Interactions

Actions define *how* to interact with a page's elements.

```typescript
// actions/editForm/editForm.actions.ts
export const editFormActions = (page: Page) => {
  const locators = editFormLocators(page);

  return {
    updateField: async (field: string, value: string) => {
      // maps field name to the correct locator and fills/selects it
    },
    clickNext: async () => {
      await locators.nextButton.click();
    },
    getTopErrorMessage: async () => {
      return locators.topErrorBanner.textContent();
    },
  };
};
```

**Why page-scoped, not flow-scoped?** "Editing a field on the Edit Form" is a property of that page's UI — it doesn't change based on which flow brought the user there, or whether the data being entered is valid or invalid. The same `updateField()` action handles both the happy path and the missing-mandatory-field scenario — only the data passed in differs.

**Why the factory pattern?** `editFormActions(page)` computes `locators` **once** and returns a set of functions that all share it — instead of every function separately calling `editFormLocators(page)`, which would repeat that line across the file as more actions get added.

### 4.4 Locators — Pure Element References

The bottom layer. A locator file contains *only* element references — no logic, no clicking, no assertions:

```typescript
// locators/editForm/editForm.locators.ts
export const editFormLocators = (page: Page) => ({
  firstNameInput: page.getByLabel('First Name'),
  ageInput: page.getByLabel('Age'),
  classDropdown: page.getByRole('combobox', { name: 'Class' }),
  classTeacherNameInput: page.getByLabel('Class Teacher Name'),
  nextButton: page.getByRole('button', { name: 'Next' }),
  topErrorBanner: page.getByText('Please fill all mandatory fields'),
});
```

**Two rules for this layer:**

- **Pure and flow-agnostic** — a locator has zero knowledge of which flow or scenario uses it. It only answers "where is this element."
- **Shared vs. page-specific** — if an element (like a header) appears on 2+ pages, it goes in `locators/shared/`, not duplicated per page.

### Folder Structure So Far

All three layers are now visible, each grouped by page (with `locators/shared/` as the one flat exception):

```
tests/
├── locators/
│   ├── shared/
│   │   └── header.locators.ts
│   ├── studentList/
│   │   └── studentList.locators.ts
│   └── editForm/
│       └── editForm.locators.ts
│
├── actions/
│   ├── studentList/
│   │   └── studentList.actions.ts
│   └── editForm/
│       └── editForm.actions.ts
│
├── steps/
│   ├── studentList/
│   │   └── selectStudent.step.ts
│   └── editForm/
│       ├── editStudentDetails.step.ts
│       └── submitEditForm.step.ts
│
└── specs/
    ├── studentList.spec.ts
    └── editForm.spec.ts
```

One layer is still missing: how API calls get mocked. That's next.

---

## 5. API Mocking — Base Response + Manipulation

Beyond UI locators, most flows also depend on API calls — and how we mock them needs the same discipline as the rest of the structure.

### The Pattern

1. **Define one base/expected response per endpoint, per page** — the "default" shape of data that endpoint returns, stored as a plain JSON file
2. **Manipulate that base response** for whatever the specific test needs (e.g., remove a field, change a value, simulate an error)
3. **Pass the manipulated response to a method in that page's `mockResponse.ts`** that applies it via Playwright's request interception (`page.route()`), so the actual network response is swapped for our expected one

A backend call belongs to whichever page consumes it. So instead of one mock file per endpoint, each page has exactly **two** files: one plain JSON file holding raw data, one `.ts` file holding every API call that page depends on.

### Example

Edit Form depends on two endpoints — student data and a teacher list for a dropdown. Both belong to Edit Form, so both live together:

```json
// mocks/editForm/editForm.mockData.json
{
  "baseStudentResponse": {
    "firstName": "Rahul",
    "lastName": "Verma",
    "age": "15",
    "class": "10A",
    "classTeacherName": "Mrs. Iyer"
  },
  "baseTeacherListResponse": ["Mrs. Iyer", "Mr. Sharma", "Ms. Rao"]
}
```

```typescript
// mocks/editForm/editForm.mockResponse.ts
import { Page } from '@playwright/test';
import mockData from './editForm.mockData.json';

const { baseStudentResponse, baseTeacherListResponse } = mockData;

export const mockStudentApi = async (page: Page, override: Partial<typeof baseStudentResponse> = {}) => {
  await page.route('**/api/students/*', (route) =>
    route.fulfill({ json: { ...baseStudentResponse, ...override } })
  );
};

export const mockTeacherListApi = async (page: Page, override: string[] = baseTeacherListResponse) => {
  await page.route('**/api/teachers', (route) => route.fulfill({ json: override }));
};
```

> Note: importing `.json` directly requires `resolveJsonModule: true` in `tsconfig.json` (on by default in most Playwright starter configs — worth confirming in ours).

**Usage in a step or spec:**

```typescript
// default/base response — no override needed
await mockStudentApi(page);

// this test needs a student with a missing class teacher
await mockStudentApi(page, { classTeacherName: '' });

// this test needs an empty teacher list
await mockTeacherListApi(page, []);
```

### Why This Pattern

| Benefit | Explanation |
|---|---|
| **One source of truth per endpoint** | `baseStudentResponse` defines the "normal" shape once — every test starts from the same known-good data |
| **Tests stay readable** | A test only shows *what's different* about its scenario (`{ classTeacherName: '' }`), not the entire response payload every time |
| **Consistent with the "no duplication" principle** | Same idea as parameterized steps — one mock function, driven by overrides, instead of a separate mock function per scenario |
| **Interception logic lives in one place** | `route()`/`fulfill()` mechanics are written once, in `mocks/`, not repeated inline at the top of every spec |
| **Data and logic are physically separated** | The JSON file can be edited by anyone (even non-engineers) without touching the mocking mechanics |

### Handling Multiple API Calls Across Multiple Tests

Since both mocks live in the same page-scoped file, a spec imports both from one place. **Three tests — one amends the student API, another amends the teacher-list API, the third uses both as base:**

```typescript
// specs/editForm.spec.ts
import { test } from '@playwright/test';
import { mockStudentApi, mockTeacherListApi } from '../mocks/editForm/editForm.mockResponse';

test('happy path — both APIs use base response', async ({ page }) => {
  await mockStudentApi(page);        // base, no override
  await mockTeacherListApi(page);    // base, no override
  await page.goto('/students/1/edit');
  // ...proceed with steps
});

test('shows error when student has no class teacher assigned', async ({ page }) => {
  await mockStudentApi(page, { classTeacherName: '' }); // amended for this test
  await mockTeacherListApi(page);                        // base — unaffected
  await page.goto('/students/1/edit');
  // ...proceed with steps
});

test('teacher dropdown shows empty state when no teachers exist', async ({ page }) => {
  await mockStudentApi(page);                // base — unaffected
  await mockTeacherListApi(page, []);        // amended for this test
  await page.goto('/students/1/edit');
  // ...proceed with steps
});
```

**The rule this demonstrates:** every test calls **every mock the page needs**, but only passes an override to the *one* endpoint that scenario actually cares about. The other endpoint(s) are called with no arguments, which falls back to the base response by default. This keeps each test's intent obvious at a glance — the override in the test body tells you exactly what's being verified, without needing to read the mock implementation.

### Which Endpoints to Mock

Not every endpoint needs this treatment. Mock endpoints whose response **varies by test scenario** (like the student data above). Endpoints that return **stable, non-journey-dependent data** — e.g., a country list that's the same regardless of which student or scenario is being tested — generally don't need per-test mocking at all.

### Folder Structure So Far — Complete

With mocks added, this is the full, final structure:

```
tests/
├── locators/
│   ├── shared/
│   │   └── header.locators.ts
│   ├── studentList/
│   │   └── studentList.locators.ts
│   └── editForm/
│       └── editForm.locators.ts
│
├── actions/
│   ├── studentList/
│   │   └── studentList.actions.ts
│   └── editForm/
│       └── editForm.actions.ts
│
├── steps/
│   ├── studentList/
│   │   └── selectStudent.step.ts
│   └── editForm/
│       ├── editStudentDetails.step.ts
│       └── submitEditForm.step.ts
│
├── mocks/
│   ├── shared/                          ← only for an endpoint genuinely used by 2+ pages
│   ├── studentList/
│   │   ├── studentList.mockData.json
│   │   └── studentList.mockResponse.ts
│   └── editForm/
│       ├── editForm.mockData.json        ← raw base data for every endpoint this page needs
│       └── editForm.mockResponse.ts      ← every route-mocking function this page needs (mockStudentApi, mockTeacherListApi, ...)
│
└── specs/
    ├── studentList.spec.ts
    └── editForm.spec.ts
```

**One thing to watch for:** this only stays clean as long as an endpoint is exclusive to one page. If `/api/teachers` also turns out to be called by a different page later, keeping `mockTeacherListApi` inside `editForm.mockResponse.ts` forces that other page to either duplicate the function or import from Edit Form's folder — breaking the same boundary we already enforce for locators. If that happens, move just that one mock into `mocks/shared/`, same as we do for shared locators.

---

## 6. Key Rules Recap

| Rule | Why |
|---|---|
| Locators are pure, no logic | So changing one element's selector never requires touching test logic |
| Locators split shared vs. page-specific | Avoids duplicating elements that appear on multiple pages |
| Actions are page-scoped, not flow-scoped | The same action is reused by every flow that touches that page — no duplication per journey |
| Actions use a factory pattern | Locators computed once per page, no repeated instantiation |
| Steps are parameterized, not duplicated | One step handles multiple scenarios via data/options — avoids near-identical files |
| `test.step()` lives inside the step | Every consuming spec gets automatic, labeled reporting |
| No Journey/orchestration layer | The spec itself represents the flow — an extra layer added no value and risked breaking page-scoping |
| Folders grouped by page, not by flow | The same page can belong to multiple flows — grouping by flow would duplicate it |
| No hard waits | Each layer uses proper condition-based waiting internally, not `waitForTimeout` |
| No single giant helper file | Locators/actions/steps must stay physically separated by page, even if a helper file "looks" clean from the spec |
| Mock data is JSON, mock logic is TS | Data and interception mechanics are physically separated, same as locators vs. actions |
| API mocks use base response + overrides | One source of truth per endpoint; tests only show what's different about their scenario, not a full duplicated payload |