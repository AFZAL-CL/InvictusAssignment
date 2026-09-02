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







