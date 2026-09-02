# Bugs found

Keep this file in the repo and **commit it** with your fixes.

---

## Bug 1

**How to reproduce:** Open the app. The expense list says “Newest first”. The first row is Wine (7 Mar). Board game (15 Mar) is further down.

**What is wrong:** The list is showing oldest expenses first. Newest should be at the top.

**What I changed:** In `src/components/ExpenseList.jsx`, I updated the sort comparator from `dateValue(a.date) - dateValue(b.date)` to `dateValue(b.date) - dateValue(a.date)`. Additionally, because `localStorage` serializes Date objects to strings, the subtraction was resulting in `NaN` and failing to sort. I fixed this by updating `dateValue` in `src/lib/format.js` to return `new Date(date).getTime()` so dates are properly compared as timestamps.

---

## Bug 2

**How to reproduce:** Delete or edit an amount for any expense other than the very first one in the list.

**What is wrong:** The `ExpenseList` was passing the array `index` from the sorted and filtered list to the `App` dispatch methods. The `store` reducer then mistakenly used this index to splice/update the unsorted `state.expenses` array, leading to the wrong expense being modified.

**What I changed:** Updated `ExpenseList.jsx` to pass `expense.id` to the `onDelete` and `onUpdate` callbacks. Updated `App.jsx` to use `id` in the dispatch payloads, and modified the `store.js` reducers to filter and map over expenses using their `id` instead of the array index.

---

## Bug 3

**How to reproduce:** Create an expense where the person who paid for it is not included in the split (e.g. they paid for a cab they didn't ride).

**What is wrong:** The `computeBalances` function in `src/lib/balances.js` had a faulty logic block that subtracted a portion of the bill from the payer's balance if they were not in the split, essentially charging them for an expense they shouldn't owe anything for.

**What I changed:** I removed the conditional block `if (!(exp.paidBy in shares) ...)` that was incorrectly deducting the amount from the payer's balance in `src/lib/balances.js`.

---

## Bug 4

**How to reproduce:** Create an expense that doesn't divide perfectly (e.g., $10 split equally among 3 people). Check the total balances or settle up amounts.

**What is wrong:** Due to `.toFixed(2)` rounding in `splitEqual` and `splitByPercent` (`src/lib/money.js`), shares often total slightly less or more than the original expense amount, causing the group to "lose" or "invent" pennies.

**What I changed:** I updated `splitEqual` and `splitByPercent` to accumulate the exact shares assigned so far and allocate the remaining exact difference to the last person in the split, guaranteeing the sum of shares always perfectly matches the original expense amount. I also relaxed the strict equality check in `percentsSumTo100` to tolerate floating point inaccuracies.

---

## Bug 5

**How to reproduce:** Setup a scenario where one person owes exactly what another person is owed (e.g. Person A has a balance of -$10 and Person B has +$10). Go to the "Settle up" list.

**What is wrong:** The algorithm in `suggestSettlements` (`src/lib/settle.js`) handles cases where the debtor owes more or less than the creditor, but if the amounts match exactly, it increments the pointers without ever generating the transfer record.

**What I changed:** I added a `transfers.push(...)` block inside the `else` block (when `d.amount === c.amount`) to ensure the transfer is properly recorded before advancing the pointers.

---

## Bug 6

**How to reproduce:** In the filter section, select a specific person under "Paid by".

**What is wrong:** All expenses disappear from the list, even if that person paid for them. This happens because the filter dropdown returns a String ID, while the expenses store the `paidBy` field as a Number. The strict inequality check in `App.jsx` (`e.paidBy !== paidBy`) then filters out everything.

**What I changed:** In `App.jsx`, I updated the filter condition to compare matching types (for example, `String(e.paidBy) !== String(paidBy)`).

---

## Bug 7

**How to reproduce:** In the expense list, try to edit an amount by typing a new number and pressing Enter, or by typing letters/blank and clicking away.

**What is wrong:** Pressing Enter does nothing (you are forced to click outside to save). Also, if you type an invalid amount (like text or leave it blank) and click away, the input box stays with the invalid text and gets out of sync with the actual amount.

**What I changed:** In `ExpenseList.jsx`, I added an `onKeyDown` handler to blur the input on Enter, added an `else` branch in `onBlur` to reset the input box back to the actual valid amount if the user enters something invalid, and also added a `useEffect` to ensure the draft input stays synced if the amount changes externally.

---

## Bug 8

**How to reproduce:** Look at the "Balances" panel on the right side of the screen. Notice the text next to each person. For example, if someone paid for an expense they weren't a part of, they should be owed money.

**What is wrong:** The labels "owes" and "is owed" are completely reversed! Positive balances (people who paid more than their share) are incorrectly labeled as "owes", and negative balances are labeled as "is owed". This makes it look like the payer owes money to the group.

**What I changed:** In `src/components/BalancesPanel.jsx`, I swapped the logic so that `bal > 0` displays "is owed" (and uses the green text class) and `bal < 0` displays "owes" (using the red text class).

---

## Bug 9

**How to reproduce:** Go to the Summary panel, and add a new member by typing a name and clicking Add. Notice that the member does not appear in the "Paid so far" list below until you happen to add or edit an expense.

**What is wrong:** The `perPerson` calculation in `SummaryCards.jsx` uses a `useMemo` hook to avoid recalculating unnecessarily, but it misses `members` in its dependency array. Thus, when a new member is added, the list isn't recalculated because it only re-runs when `expenses` change.

**What I changed:** I updated the dependency array of the `useMemo` in `src/components/SummaryCards.jsx` to include `members` so the list immediately updates when someone is added.

---

## Bug 10

**How to reproduce:** Add a new expense (e.g. "Ice cream") and notice its date is nicely formatted as "16 Mar 2026". Now, refresh the page. Look at the same expense again—the date is now ugly formatted as "2026-03-16".

**What is wrong:** The `loadState` function in `src/state/store.js` pulled the data from `localStorage` using `JSON.parse` but completely skipped hydrating it. Because of this, all `Date` objects were loaded back into the app as plain ISO strings, causing the `formatDate` helper to fall back to an ugly string slice format instead of a localized date string. 

**What I changed:** I updated `loadState` to return `hydrate(JSON.parse(raw))` instead of just `JSON.parse(raw)`. This ensures that strings are correctly parsed back into `Date` objects upon loading the app from memory, maintaining formatting consistency across reloads.

---

## Bug 11

**How to reproduce:** Fill out the "Add expense" form (e.g., Description: "Lunch", Amount: "45") and click "Save expense". Notice that the expense appears in the list, but the form remains fully populated with "Lunch" and "45". 

**What is wrong:** The form state does not reset upon successful submission. This is a classic UX bug that makes it extremely easy for a user to accidentally double-click "Save expense" and create duplicate entries because the form gives no visual reset feedback.

**What I changed:** I added `setDescription("")` and `setAmount("")` to the end of the `submit` function in `src/components/AddExpenseForm.jsx` so that the form properly clears itself for the next entry.

---

## Bug 12

**How to reproduce:** In the "Add expense" form, select "Custom %". Try to type a percentage with a decimal point (like "33.33"). Notice that as soon as you type the decimal point ("33."), the decimal point immediately vanishes, making it practically impossible to type decimal percentages!

**What is wrong:** The React `<input>` handler for the custom percentages was casting `e.target.value` immediately into a `Number(...)` before saving it to state. When you type `"33."`, `Number("33.")` instantly resolves back to `33`, meaning the decimal point is completely stripped before you can even type the next digit.

**What I changed:** I updated the `onChange` handler in `src/components/AddExpenseForm.jsx` to store the raw string (`e.target.value`) in state instead of aggressively parsing it to a `Number`. The math helper functions already cleanly convert these strings to numbers behind the scenes, so now the user can type decimal points smoothly.

---

## Bug 13

**How to reproduce:** Go to the Summary panel and try to add a new member whose name starts with a symbol (like "@John") or contains numbers (like "John123").

**What is wrong:** The input accepts these values without throwing any warning or error, meaning a user can accidentally create invalid or typo-filled member names.

**What I changed:** I added an `error` state in `src/components/SummaryCards.jsx`. When the form is submitted, it validates that the name starts with a letter (`/^[a-zA-Z]/`) and does not contain any numbers (`/[0-9]/`). If validation fails, it displays a descriptive error message underneath the input.

---

## Bug 14

**How to reproduce:** In the "Add expense" form, clear the Date field (using the 'X' icon or backspacing) and submit the form. 

**What is wrong:** The form lacks validation for the Date field. It successfully creates the expense but initializes it with an "Invalid Date" object. This breaks the sorting algorithm (putting the expense in an unpredictable spot) and displays "Invalid Date" directly in the UI list.

**What I changed:** I added a validation check `if (!date)` to the `submit` function in `src/components/AddExpenseForm.jsx` to ensure a valid date string must be present before the form is allowed to save, throwing an error otherwise.

---

## Bug 15

**How to reproduce:** Create an expense with an astronomically large amount (e.g., "$99,999,999").

**What is wrong:** The "Group total" and "Avg / person" numbers physically overflow outside of their containing boxes in the Summary panel, breaking the layout. 

**What I changed:** I updated `src/index.css` by adding `overflow-wrap: anywhere;` and `word-break: break-all;` to the `.stat b` class. This perfectly forces massive text strings to wrap to the next line rather than bleeding out of the component.

---

## Bug 16

**How to reproduce:** Add a new expense and type a massively long string of characters without spaces into the description (e.g. "SuperLongDescriptionWithNoSpacesAtAllThatIsWayTooLong"). 

**What is wrong:** In the main Expense List, the grid column containing the description expands infinitely to fit the unbroken word. This blows out the UI grid horizontally and pushes the amount and delete button completely off-screen.

**What I changed:** I targeted the middle column of `.expense` in `src/index.css` (`.expense > div:nth-child(2)`) and added `min-width: 0;` alongside `word-break: break-word;`. Setting `min-width: 0` stops the CSS grid container from expanding past its track width, forcing the text to correctly wrap to the next line.

---

## Bug 17

**How to reproduce:** Go to the Summary panel and type in the name of a member who already exists (e.g., "Aisha Khan") and click "Add".

**What is wrong:** The app allows you to add a member with the exact same name as someone who is already in the group. While they receive different internal IDs and the math still functions, this is incredibly confusing for the UI since there's no way to visually tell the two users apart when selecting who paid.

**What I changed:** I added a validation check to the form submit handler in `src/components/SummaryCards.jsx`. It now checks `members.some((m) => m.name.toLowerCase() === trimmed.toLowerCase())`. If a duplicate is found (case-insensitive), it halts the submission and shows the error: "A member with this name already exists."

---

