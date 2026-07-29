Application Used as the Example

Every section below reuses the same three-page flow, so the app is described once here — no section repeats these details, it just applies them.

Page 1 — Student List: shows up to 5 students in a class. Clicking a student navigates to Edit Form with that student's data prefilled.
Page 2 — Edit Form: prefilled fields are firstName, lastName, age, class (dropdown), and classTeacherName — all mandatory. The Next button is always enabled; submitting with a missing mandatory field shows a top-of-form error banner instead of disabling the button. Navigating back to Student List and re-selecting the same student always resets the form to its original prefilled values — there's no unsaved-edit persistence.
Page 3 — Thank You: confirmation screen; no data is actually saved. The confirmation text shown varies depending on the data it receives.



---

Version 2
# POM Structure Example — Student List → Edit Form → Thank You

Applying the agreed structure (pure locators, factory-pattern actions, parameterized steps, no classes, no Journey layer) to a 3-page flow with conditional UI state (Next button enable/disable based on edit).

**Flow:**
Page 1 — Student List (max 5 students) → click student → Page 2 — Edit Form (prefilled, Next disabled until edited) → click Next/Submit → Page 3 — Thank You

---

## 1. Application Overview

| Page | Purpose | Key Behavior |
|---|---|---|
| Student List | Shows up to 5 students in the class | Clicking a student navigates to Edit Form with that student's data prefilled |
| Edit Form | Shows prefilled student details | All 5 fields mandatory (First Name, Last Name, Age, Class, Class Teacher). Next button stays **disabled** until at least one field is edited |
| Thank You | Confirmation screen | No data persistence required — this is a UI-flow-only confirmation |

**Fields on Edit Form:** `firstName`, `lastName`, `age`, `class`, `classTeacherName`

---

## 2. Folder Structure

```
tests/
├── locators/
│   ├── shared/
│   │   └── header.locators.ts
│   ├── studentList/
│   │   └── studentList.locators.ts
│   ├── editForm/
│   │   └── editForm.locators.ts
│   └── thankYou/
│       └── thankYou.locators.ts
│
├── actions/
│   ├── studentList/
│   │   └── studentList.actions.ts
│   ├── editForm/
│   │   └── editForm.actions.ts
│   └── thankYou/
│       └── thankYou.actions.ts
│
├── steps/
│   ├── studentList/
│   │   └── selectStudent.step.ts
│   ├── editForm/
│   │   ├── editStudentDetails.step.ts
│   │   └── submitEditForm.step.ts
│   └── thankYou/
│       └── verifyThankYou.step.ts
│
└── specs/
    ├── studentList.spec.ts
    ├── editForm.spec.ts
    └── endToEndFlow.spec.ts
```

> Same pattern as the Contact Form example: `locators/shared/` stays flat, everything else groups by page name consistently across `locators/`, `actions/`, `steps/`.

---

## 3. Locators — What Each File Holds (No Code, Just Contract)

**`locators/studentList/studentList.locators.ts`**
- Student card/row (parameterized by student name or index — max 5, so likely `nth()` or `getByRole('row', { name })`)
- Student name text within each row
- Page heading

**`locators/editForm/editForm.locators.ts`**
- `firstNameInput`
- `lastNameInput`
- `ageInput`
- `classDropdown` (dropdown/combobox — use `getByRole('combobox', ...)`)
- `classTeacherNameInput` (text field)
- `nextButton` (always enabled — no state to assert here)
- `topErrorBanner` (shown when mandatory field is missing on submit attempt)

**`locators/thankYou/thankYou.locators.ts`**
- Confirmation heading
- Confirmation message

---

## 4. Actions — What Each File Exposes (Factory Pattern)

**`actions/studentList/studentList.actions.ts`**
- `selectStudentByName(name)` — clicks the row/card matching a given student name
- `getStudentNames()` — reads all visible student names (useful for asserting max-5 constraint)

**`actions/editForm/editForm.actions.ts`**
- `getPrefilledValues()` — reads current values of all 5 fields (for asserting prefill correctness before edit)
- `updateField(fieldName, value)` — updates a single field (generic, not one function per field — keeps this action reusable across any of the 5 fields, including the `class` dropdown)
- `clearField(fieldName)` — clears a single field (used to test the missing-mandatory-field error case)
- `clickNext()`
- `getTopErrorMessage()` — reads the error banner text, if visible

**`actions/thankYou/thankYou.actions.ts`**
- `getConfirmationText()`

---

## 5. Steps — Parameterized, Composed from Actions

**`steps/studentList/selectStudent.step.ts`**
- `selectStudentStep(page, studentName)` — wraps `selectStudentByName()` in `test.step()`, named dynamically: `` `Select student: ${studentName}` ``

**`steps/editForm/editStudentDetails.step.ts`**
- `editStudentDetailsStep(page, fieldUpdates)` — accepts a partial object of field/value pairs (e.g., `{ age: '15' }` or `{ firstName: 'Rahul', class: '10A' }`)
- Loops through provided fields, calls `updateField()` for each
- This single parameterized step covers "edit one field," "edit all fields," and "edit nothing" (empty object) scenarios — no separate step files needed per field combination

**`steps/editForm/submitEditForm.step.ts`**
- `submitEditFormStep(page)` — clicks Next (always enabled, so no pre-check needed)
- Wrapped in `test.step('Submit edited student details')`
- Two possible outcomes after this step, asserted separately by the calling spec:
  - Valid data → navigates to Thank You page
  - Missing mandatory field → stays on Edit Form, `topErrorBanner` becomes visible

**`steps/thankYou/verifyThankYou.step.ts`**
- `verifyThankYouStep(page)` — asserts confirmation heading/message visible

---

## 6. Key Scenarios This Structure Needs to Support

| Scenario | How It's Composed |
|---|---|
| Happy path — edit one field, submit | `selectStudentStep` → `editStudentDetailsStep({ age: '16' })` → `submitEditFormStep` → `verifyThankYouStep` |
| Submit with no edits at all (still prefilled/valid) | `selectStudentStep` → `submitEditFormStep` → `verifyThankYouStep` (all fields still valid, so this likely succeeds — confirm this is intended, since Next is always enabled) |
| Mandatory field cleared → error banner shown | `selectStudentStep` → `editStudentDetailsStep({ firstName: '' })` → `submitEditFormStep` → assert `getTopErrorMessage()` is visible, page stays on Edit Form |
| Prefill correctness check | `selectStudentStep` → assert `getPrefilledValues()` matches the student clicked from list |
| Max 5 students constraint | On Student List spec — assert `getStudentNames().length <= 5` |
| Class dropdown edit | `editStudentDetailsStep({ class: '10A' })` — same generic `updateField()` handles dropdown selection internally |

---

## 7. Confirmed Behavior

- ✅ `class` field is a **dropdown** — locator uses `getByRole('combobox', ...)`
- ✅ Next button is **always enabled** — no disabled-state assertion needed anywhere
- ✅ Missing mandatory field on submit → **error banner shown at top of form**, Next remains enabled, page stays on Edit Form

## 8. Confirmed Behavior — Re-entry

- ✅ Navigating back to Student List and re-selecting the same student **always resets to original prefilled values** — unsaved edits are discarded. No back-navigation-prefill test scenario needed; prefill only needs verification on first entry (already covered in the scenario table above).

---

## 9. Notes for Confluence Page

- This structure directly reuses the pattern established on the **POM Architecture** and **God File + Helper Layering** pages — no new concepts introduced, just applied to a 3-page flow with conditional (disabled/enabled) UI state.
- The `editStudentDetailsStep` parameterization is a good real-world example to reference on the **Best Practices** page under "avoid duplicate steps" — same principle as the Contact Form's `giveConsent` toggle, generalized to N fields.
- No Journey layer used — `endToEndFlow.spec.ts` composes all steps directly, same as the Contact Form example's spec file.