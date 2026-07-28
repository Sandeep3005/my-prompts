# Prompt: Extract Shared App Example to Top of Page

## PROMPT

I have a Confluence page (Playwright + React testing best practices) that uses one running example — the **Student List → Edit Form → Thank You** application — across multiple sections (locator strategy, async/React state handling, selective endpoint mocking, `test.step`). Right now the application's details are only partially introduced up front, and individual sections lean on bits of app context (field names, page behavior) that a reader would need to piece together from scattered mentions.

Restructure the page so the application example is fully described once, at the very top, before any of the technical sections — and every section below can be read independently after that, without needing to jump elsewhere in the page to understand what a code snippet is referring to.

Specifically:

1. Expand the top-of-page "Application Used as the Example" section so it's a complete, standalone description of all three pages: Student List (up to 5 students, click navigates to Edit Form prefilled), Edit Form (list every field: `firstName`, `lastName`, `age`, `class` dropdown, `classTeacherName`; note all fields are mandatory; note the Next button is always enabled and a missing mandatory field shows a top-of-form error banner instead of disabling the button; note re-selecting a student from Student List always resets to original prefilled values), and Thank You (confirmation-only, no data saved, confirmation text varies based on the data it receives).
2. Go through each existing section (locator strategy, async/state handling, selective mocking, `test.step`) and remove any re-explanation of app behavior that's now redundant with the top-level description — keep only what's specific to that section's point (e.g. keep the code samples and the technique being taught, drop restated app context).
3. Do not change the substance of any section's technical content, code examples, or the order of sections — this is a restructuring pass for redundancy and readability, not a rewrite of the guidance itself.
4. Keep the same formatting conventions as the rest of the page: TypeScript arrow functions only (no classes/constructors), code blocks for examples, the locator priority table unchanged, and the closing "Key Takeaways" list unchanged.

Output the full page with this restructuring applied, ready to paste into Confluence.