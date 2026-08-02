A 3-page application flow:
Page 1 — Student List → click a student → Page 2 — Edit Form (prefilled) → click Next/Submit → Page 3 — Thank You

Page 1: Student List
Displays a list of students in the class — maximum 5 students
Each student is shown as a row/card with their name
Clicking on a student navigates to the Edit Form, pre-loaded with that student's existing details
Page 2: Edit Form
Shows the selected student's details, pre-filled with their existing data
5 fields, all mandatory:
First Name (text)
Last Name (text)
Age (text)
Class (dropdown)
Class Teacher Name (text)
The Next button is always enabled — it's never disabled based on form state
If the user clicks Next while a mandatory field is empty, a top-of-form error banner appears (instead of the button being disabled)
If the user navigates back to the Student List and re-selects the same student, the form always resets to the original prefilled values — any unsaved edits are discarded
No data is actually persisted anywhere — this is purely a UI flow demonstration
Page 3: Thank You
A simple confirmation screen shown after successful submission
No further action required, no data saved