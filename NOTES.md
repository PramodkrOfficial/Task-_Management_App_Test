# NOTES.md

# Task Management Application — Bug Fix Notes

## Overview

This document records the functional, UI, code-quality, and accessibility issues identified during the review of the task management application.

For each issue, the following information is documented:

- **Bug ID**
- **Location**
- **Problem**
- **Root cause**
- **Fix approach**
- **Verification**

The purpose of this document is to provide a clear record of the debugging process and the reasoning behind each fix.

---

# 1. Functional Bugs

## B1 — Errors Are Never Shown

**Bug ID:** B1  
**Location:** `frontend/src/hooks/useTasks.js`, `frontend/src/components/TaskTable.jsx`

### Problem

When the backend is unavailable or an API request fails, the application keeps showing **"Loading tasks..."** forever instead of displaying an error message.

### Root Cause

`setLoading(false)` was only called when the API request succeeded. When the request failed, the loading state remained `true`.

In addition, `TaskTable` checks the loading state before checking the error state, so the error message could not be reached while loading remained `true`.

### Fix Approach

Updated the request handling so that the loading state is reset after both successful and failed requests.

When an error occurs:

- Clear the task list.
- Reset the total count.
- Store the error message.
- Set `loading` to `false`.

### Verification

Stopped the backend and refreshed the application. Verified that the loading message no longer stayed on the screen forever and that an appropriate error message was displayed.

---

## B2 — Pagination Is Not Reset After Search or Filter Changes

**Bug ID:** B2  
**Location:** `frontend/src/App.jsx`

### Problem

Changing the search query or status filter while on a later page could display **"No tasks found"** even though matching tasks existed.

### Root Cause

The current page number was not reset when the search query or status filter changed.

For example, if the user was on page 3 and then searched for a term that only had results on page 1, the application continued requesting page 3.

### Fix Approach

Reset pagination to page 1 whenever the search query or status filter changes.

Also added page clamping so that if the available number of pages becomes smaller than the current page, the current page is moved to the last valid page.

### Verification

Navigated to page 3, entered a search query with fewer results, and verified that the application returned to page 1 and displayed the matching tasks.

---

## B3 — Search Response Race Condition

**Bug ID:** B3  
**Location:** `frontend/src/hooks/useTasks.js`, `frontend/src/api.js`

### Problem

When typing a search query quickly, an older API response could arrive after a newer response and replace the results with outdated data.

### Root Cause

Previous API requests were not cancelled when a new request was started.

Because network responses can arrive in a different order from the order in which requests were sent, an older response could overwrite the results belonging to the latest search query.

### Fix Approach

Used `AbortController` to cancel the previous API request when the search query, status filter, page, or other request parameters change.

Cancelled requests are ignored by checking for `AbortError`.

### Verification

Entered search terms quickly and verified that older API responses could not overwrite the results for the latest search query.

---

## B4 — No Debounce on Search

**Bug ID:** B4  
**Location:** `frontend/src/App.jsx`, `frontend/src/hooks/useTasks.js`

### Problem

The application sent an API request for every character typed into the search box.

### Root Cause

The search query was sent to the backend immediately after every input change without any delay.

### Fix Approach

Added a debounced search value with an approximately 300 ms delay.

The API request is made only after the user stops typing for the debounce period.

### Verification

Typed a search query character by character and checked the network requests. Verified that unnecessary requests were reduced and that the final query was requested after the debounce delay.

---

## B5 — Application Crashes When Status Is Null

**Bug ID:** B5  
**Location:** `frontend/src/components/TaskTable.jsx`

### Problem

The entire page could crash when a task returned from the backend had a `null` status.

### Root Cause

The code called:

```js
task.status.toLowerCase()
```

Calling a method on `null` causes a JavaScript runtime error.

### Fix Approach

Added a safe fallback before converting the status to lowercase:

```js
(task.status ?? 'unknown').toLowerCase()
```

This prevents the application from crashing when the backend returns a missing status.

### Verification

Tested the task table with a task whose status was `null` and verified that the application remained functional instead of showing a blank page.

---

## B6 — Error State Is Not Cleared After Success

**Bug ID:** B6  
**Location:** `frontend/src/hooks/useTasks.js`

### Problem

After an API request failed, the error state could remain even after a later request succeeded.

### Root Cause

The error state was not reset when a new API request started.

### Fix Approach

Set the error state to `null` at the beginning of every new request.

This ensures that an old error message does not remain visible after a successful request.

### Verification

Forced an API failure, restored the backend, and performed another request. Verified that the previous error message was cleared after the successful request.

---

## B7 — Search Query Is Not Trimmed

**Bug ID:** B7  
**Location:** `frontend/src/App.jsx`

### Problem

Leading or trailing spaces in the search query could be sent to the backend and produce unnecessary or unexpected search results.

### Root Cause

The application used the raw input value without removing unnecessary whitespace.

### Fix Approach

Trimmed the search query before sending it to the API.

The debounced query is created from the trimmed value:

```js
const debouncedQuery = useDebouncedValue(query.trim());
```

### Verification

Searched for a task using leading and trailing spaces and verified that the application treated the query the same as the trimmed search text.

---

## B8 — Raw Status Enum Is Displayed

**Bug ID:** B8  
**Location:** `frontend/src/components/TaskTable.jsx`

### Problem

The status badge displayed the backend enum value `IN_PROGRESS` directly to the user.

### Root Cause

The UI did not convert backend status values into human-readable labels.

### Fix Approach

Created a status label mapping:

```text
OPEN        → Open
IN_PROGRESS → In Progress
DONE        → Done
```

The mapped value is used when displaying the status badge.

### Verification

Checked tasks with different statuses and verified that users saw readable labels instead of raw enum values.

---

## B9 — API Response Is Not Validated

**Bug ID:** B9  
**Location:** `frontend/src/hooks/useTasks.js`

### Problem

Invalid or incomplete API responses could cause incorrect task lists or pagination calculations.

### Root Cause

The frontend assumed that the API response always contained a valid `items` array and numeric `total` value.

### Fix Approach

Validated the API response before using it.

- Used an empty array when `items` is invalid.
- Used `0` when `total` is not a finite number.

Example:

```js
setTasks(Array.isArray(data?.items) ? data.items : []);
setTotal(Number.isFinite(data?.total) ? data.total : 0);
```

### Verification

Tested responses with missing or invalid `items` and `total` values and verified that the application remained stable and pagination did not produce `NaN`.

---

## B10 — Page Size Is Hard-Coded

**Bug ID:** B10  
**Location:** `frontend/src/App.jsx`

### Problem

The page size value `10` was repeated in multiple places.

### Root Cause

The page size was hard-coded separately in the API hook call and pagination calculation.

### Fix Approach

Created a shared `PAGE_SIZE` constant and used it wherever the page size is required.

This keeps the API request and pagination calculation consistent.

### Verification

Verified that both API requests and pagination calculations use the same page-size constant.

---

## B11 — Debug Console Log Left in Code

**Bug ID:** B11  
**Location:** `frontend/src/api.js`

### Problem

A debugging `console.log` statement was left in the application code.

### Root Cause

The temporary debugging statement was not removed after development and testing.

### Fix Approach

Removed the unnecessary `console.log` statement from `api.js`.

### Verification

Opened the browser developer console and verified that the application no longer produced the unnecessary debug log.

---

# 2. UI Layout Bugs

## L1 — Mobile Layout Overflow

**Bug ID:** L1  
**Location:** Frontend controls and task table CSS

### Problem

The controls and five-column task table could overflow horizontally on small or mobile screens.

### Root Cause

The controls did not allow wrapping and the table did not have a horizontal scroll container.

### Fix Approach

Updated the layout to:

- Allow controls to wrap.
- Make the search input flexible.
- Wrap the table inside a horizontally scrollable container.
- Give the table a minimum width so its columns remain usable.

Example:

```css
.controls {
  flex-wrap: wrap;
}

.search-input {
  min-width: 0;
  flex: 1 1 200px;
}

.table-wrapper {
  overflow-x: auto;
}

.task-table {
  min-width: 600px;
}
```

### Verification

Opened the application at a mobile-sized viewport and verified that the page no longer caused unwanted horizontal page overflow and that the table could be scrolled horizontally.

---

## L2 — Long Text Is Not Properly Handled

**Bug ID:** L2  
**Location:** Task table CSS

### Problem

Very long task titles or descriptions could expand the table and negatively affect the page layout.

### Root Cause

Long text was not constrained or wrapped properly.

### Fix Approach

Added `overflow-wrap: anywhere` to task titles and descriptions.

Descriptions were also limited to two lines using CSS line clamping.

Example:

```css
.task-title,
.task-desc {
  overflow-wrap: anywhere;
}

.task-desc {
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
}
```

### Verification

Tested tasks with very long titles and descriptions and verified that the table remained within the expected layout.

---

## L3 — Layout Jumps Between Different States

**Bug ID:** L3  
**Location:** Task table, loading, and empty-state CSS

### Problem

The page layout changed height when switching between loading, empty, and table states, causing visible layout jumps.

### Root Cause

The different UI states did not have a consistent minimum height.

### Fix Approach

Added a minimum height to the state-message container and centered its content.

A further improvement would be to keep existing rows visible while new data is loading and visually dim them instead of replacing them with a loading message.

### Verification

Changed between loading, empty, and populated states and verified that the page layout remained more stable.

---

## L4 — Table Rounded Corners Are Unreliable

**Bug ID:** L4  
**Location:** Task table CSS

### Problem

Rounded corners and overflow behavior on the table were unreliable.

### Root Cause

Using `overflow: hidden` together with `border-collapse: collapse` on a table does not reliably clip the table's rounded corners.

### Fix Approach

Moved the overflow and rounded-corner styling to a surrounding `.table-wrapper`.

Removed the unreliable overflow and border-collapse-related styling from the table itself where appropriate.

### Verification

Checked the table at different screen sizes and verified that the rounded container and horizontal scrolling worked correctly.

---

# 3. Accessibility Bugs

## A1 — Missing Accessibility Labels, Live Regions, and Focus Indicators

**Bug ID:** A1  
**Location:** Search input, status filter, loading/error states, and focus CSS

### Problem

The search input and status filter did not have proper accessible names.

Loading and error states also lacked appropriate accessibility announcements, and keyboard focus was difficult to see.

### Root Cause

The UI relied mainly on placeholders and visual styling without providing sufficient semantic and focus information for assistive technology and keyboard users.

### Fix Approach

Added:

- `aria-label="Search tasks"` to the search input.
- `aria-label="Filter by status"` to the status filter.
- `role="status"` and `aria-live="polite"` to loading and empty-state messages.
- `role="alert"` to error messages.
- Visible `:focus-visible` outlines for inputs, selects, and buttons.

Example:

```css
:focus-visible {
  outline: 2px solid #1565c0;
  outline-offset: 1px;
}
```

### Verification

Navigated through the controls using the keyboard and verified that focus was clearly visible.

Also verified that loading and error messages had appropriate accessibility attributes.

---

## A2 — Insufficient Text Contrast

**Bug ID:** A2  
**Location:** Task description and subtitle CSS

### Problem

The `#888` text color on a white background had insufficient contrast for normal-sized text.

### Root Cause

The selected gray color produced approximately a 3.5:1 contrast ratio, which is below the WCAG AA requirement of 4.5:1 for normal text.

### Fix Approach

Changed the affected text color to a darker value such as `#666` to improve readability and contrast.

### Verification

Checked the affected text against the white background and verified that the darker color provided sufficient contrast and remained readable.

---

# 4. Final Verification Checklist

Before submitting the repository, verify the following:

- [ ] Application starts with `./mvnw spring-boot:run`
- [ ] Frontend starts with `npm run dev`
- [ ] `NOTES.md` exists at the project root
- [ ] `handwritten/` folder exists
- [ ] Handwritten notes cover every bug that was fixed
- [ ] Each handwritten explanation includes location, discovery, root cause, and fix approach
- [ ] Search works correctly
- [ ] Search is debounced
- [ ] Search queries are trimmed
- [ ] Previous search requests cannot overwrite newer results
- [ ] Status filtering works correctly
- [ ] Search and status filtering work together
- [ ] Pagination resets when search/filter changes
- [ ] Pagination does not become stuck on an invalid page
- [ ] API errors are displayed
- [ ] Loading state ends after both success and failure
- [ ] Error state clears after a successful request
- [ ] Null task status does not crash the application
- [ ] API response data is safely validated
- [ ] Status values are displayed using readable labels
- [ ] Page size uses a shared constant
- [ ] Unnecessary debug logging is removed
- [ ] Application works on mobile-sized screens
- [ ] Long titles/descriptions do not break the layout
- [ ] Loading/empty/table states have stable layout
- [ ] Keyboard focus is visible
- [ ] Search and filter controls have accessible names
- [ ] Loading/error messages have appropriate ARIA attributes
- [ ] Text contrast is sufficient
- [ ] `node_modules/` is not committed
- [ ] `target/` is not committed
- [ ] Other build artifacts are not committed
- [ ] All intended changes are committed
- [ ] Changes are pushed to the repository

---

# 5. Interview Preparation

Be prepared to explain the following during the review.

### What did you fix first and why?

I prioritized the critical functional bugs first, especially error handling and pagination, because these issues could make the application unusable rather than simply affecting presentation.

### What did you choose not to fix?

I separated confirmed frontend issues from backend issues that could not be verified because the backend source was not available. I avoided making speculative changes without being able to reproduce or confirm the problem.

### What subtle issue did you find?

The search had both a race-condition problem and a request-frequency problem. Debouncing reduces unnecessary requests, while `AbortController` prevents an older response from overwriting a newer search result.

### How did you use AI/tools?

I used tools to help identify potential issues and possible solutions, but I verified the behavior by reproducing the problems and checking the application after making changes. I should be able to explain which suggestions I accepted, which I modified, and which I rejected.

### What would you do with more time?

With more time, I would:

1. Add automated frontend tests.
2. Add backend unit and integration tests.
3. Test search, filtering, and pagination together.
4. Add server-side validation for pagination parameters.
5. Add a maximum server-side page size.
6. Verify backend search and filtering behavior end-to-end.
7. Perform a more comprehensive accessibility review.

---

# 6. Submission Notes

The repository should contain the required documentation and handwritten explanations before it is submitted.

The handwritten notes should not only show the final fix. They should demonstrate the debugging process:

**Location → Discovery/Reproduction → Root Cause → Fix Approach → Verification**

This makes it easier to explain the work during the follow-up interview.
