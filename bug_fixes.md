Bug 1:

Cause: The `scrollY` and `top` variables always sum to the same value, fixing the dropdown's vertical position.

Solution:

```JavaScript
// components/InputSelect/index.tsx
const getDropdownPosition: GetDropdownPositionFn = (target) => {
  if (target instanceof Element) {
    const { top, left } = target.getBoundingClientRect();
    return {
      top: top + 63,
      left,
    };
  }
```

```CSS
/* index.css */
.RampInputSelect--dropdown-container-opened {
  display: block;
  position: sticky;
}
```

Removing the `scrollY` variable produces the correct initial offset relative to the parent select element.
Setting `position: sticky` for the relevant CSS class maintains the offset even as the parent moves on
the screen due to scrolling.

Bug 2:

Cause: The `label` for the checkbox is not bound to the checkbox `input` itself.

Solution:

```JavaScript
// components/InputCheckbox/index.tsx
<div className="RampInputCheckbox--container" data-testid={inputId}>
  <label
    className={classNames("RampInputCheckbox--label", {
      "RampInputCheckbox--label-checked": checked,
      "RampInputCheckbox--label-disabled": disabled,
    })}
    htmlFor={inputId} // solved bug 2: label missing htmlFor
  />
```

Adding the `htmlFor` attribute bounds the `label` to the corresponding `input` specified by `inputId`, triggering 
the `input`'s `onChange` callback when the `label` is clicked.

Bug 3:

Cause: `loadTransactionsByEmployee` is called with `EMPTY_EMPLOYEE`, which has an undefined `id` field.

Solution:

```JavaScript
// App.tsx
onChange={async (newValue) => {
  if (newValue === null) {
    return;
  }
  if (newValue === EMPTY_EMPLOYEE) await loadAllTransactions();
  else await loadTransactionsByEmployee(newValue.id);
}}
```

Call `loadAllTransactions()` instead to retrieve paginated transactions for all employees.

Bug 4:

Cause: `setPaginatedTransactions` discards previous transactions when retrieving new ones.

Solution:

```JavaScript
// hooks/usePaginatedTransactions.ts
setPaginatedTransactions((previousResponse) => {
  if (response === null || previousResponse === null) {
    return response;
  }
  return {
    data: [...(paginatedTransactions?.data || []), ...response.data],
    nextPage: response.nextPage,
```
The `data` field in the object returned by the function now concatenates new transactions (`response.data`) 
to existing ones (`paginatedTransactions?.data`), if applicable.


Bug 5:

Cause: The `isLoading` state variable is not set to `false` until after all transactions are loaded in.

Solution:

```JavaScript
// App.tsx
const loadAllTransactions = useCallback(async () => {
  setIsLoading(true);
  transactionsByEmployeeUtils.invalidateData();

  await employeeUtils.fetchAll();
  
  setIsLoading(false);

  await paginatedTransactionsUtils.fetchAll();
```

`setIsLoading(false)` is moved before fetching all transactions.

Bug 6:

Cause: Insufficient conditional checks before the code for displaying the "View More" button result 
in the button appearing when it should not be visible.

Solution: 

```JavaScript
// App.tsx
{transactions !== null &&
  transactionsByEmployee === null &&
  paginatedTransactions?.nextPage && (
  <button
...
```

The `transactionsByEmployee === null` check handles bug 6.1 by ensuring that the "View More" button 
is not rendered when transactions are filtered by employee. The `paginatedTransactions?.nextPage` 
check handles bug 6.2 by ensuring the the "View More" button is not rendered when there are no more
transactions to display. 

Bug 7:

Cause: The transaction approval values default back to their initial states because the problem continually 
pulls the stale initial values from the cache.

Solution:

```JavaScript
// components/Transactions/index.tsx
const { fetchWithoutCache, loading, clearCache } = useCustomFetch();

const setTransactionApproval = useCallback<SetTransactionApprovalFunction>(
  async ({ transactionId, newValue }) => {
    clearCache();
    await fetchWithoutCache<void, SetTransactionApprovalParams>(
...
```

Calling `clearCache` during transaction approval writes forces cache consistency by causing a cache miss. 
This solution will result in more cache misses than simply implementing a write-through approach but suffices 
for the small scale of the current problem.

