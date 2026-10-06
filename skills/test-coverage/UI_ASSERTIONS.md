# UI & Playwright Test Assertions

Guidance for asserting UI state in browser tests. Examples use Playwright notation
because UI assertions need a concrete locator API — the patterns transfer directly to
Cypress, Testing Library, and other runners.

> **These are defaults.** The user's instructions, `CLAUDE.md` / `AGENTS.md`, and the
> conventions of the test file you're editing all take precedence — see **Precedence** in
> [SKILL.md](./SKILL.md). In particular: if the project uses a different runner or a
> different locator strategy, follow theirs.

Two things to balance: **data verification** (the correct content is displayed) and
**flow verification** (the UI responds correctly to interaction). Most weak UI tests
cover neither — they assert that an element exists.

The core failure mode in UI tests: `toBeVisible()` as a stand-in for a real assertion.
An element being present says nothing about whether it shows the right thing.

## Locators

Locate what the user interacts with the way a user — or a screen reader — finds it: by
role and accessible name (`getByRole`, `getByLabel`). The test then fails when a button
loses its name or a field loses its label, which is a real bug for keyboard and
screen-reader users, and it survives markup refactors that would break a CSS selector.

Fall back to `data-testid` for content with no role or name of its own — a total, a date
in a table cell, a status badge. That's why the display assertions below still use it.
Avoid CSS ids and classes as locators altogether.

```typescript
// BAD — passes even if the button has no accessible name
await page.click('[data-testid="delete-invoice-btn"]');
await page.fill('#email', 'test@customer.com');

// GOOD — fails if the name or label goes missing
await page.getByRole('button', { name: 'Delete invoice' }).click();
await page.getByLabel('Email').fill('test@customer.com');
```

## Data Display Assertions

### Table / List Content

```typescript
// BAD — only verifies count
await expect(page.getByRole('row')).toHaveCount(4);  // header + 3

// GOOD — count AND content
const rows = page.getByRole('table', { name: 'Invoices' }).locator('tbody').getByRole('row');
await expect(rows).toHaveCount(3);

await expect(rows.nth(0).getByRole('cell').nth(0)).toHaveText('INV-001');
await expect(rows.nth(0).getByRole('cell').nth(1)).toHaveText('Acme Corp');
await expect(rows.nth(0).getByRole('cell').nth(2)).toHaveText('$1,500.00');

// Or, when order doesn't matter, verify the set
const invoiceNumbers = await page.locator('[data-testid="invoice-number"]').allTextContents();
expect(invoiceNumbers).toContain('INV-001');
expect(invoiceNumbers).toContain('INV-002');
expect(invoiceNumbers).toContain('INV-003');
```

### Displayed Values Match the Data

```typescript
// BAD — element exists
await expect(page.locator('.total-amount')).toBeVisible();

// GOOD — the value is right, including formatting
await expect(page.locator('[data-testid="total-amount"]')).toHaveText('$1,500.00');

// Better still: derive the expectation from the test fixture
const invoice = await createTestInvoice({ totalAmount: 1500 });
await page.goto(`/invoices/${invoice.id}`);
await expect(page.locator('[data-testid="total-amount"]')).toHaveText('$1,500.00');
```

### Sorting

Clicking a sort control and asserting the table still renders proves nothing.

```typescript
// BAD
await page.getByRole('button', { name: 'Sort by date' }).click();
await expect(page.getByRole('table')).toBeVisible();

// GOOD — verify the actual order, and that the sort state is exposed
await page.getByRole('button', { name: 'Sort by date' }).click();

await expect(page.getByRole('columnheader', { name: /Date/ })).toHaveAttribute('aria-sort', 'descending');

const dates = await page.locator('[data-testid="invoice-date"]').allTextContents();
const parsed = dates.map(d => new Date(d).getTime());

for (let i = 1; i < parsed.length; i++) {
  expect(parsed[i - 1]).toBeGreaterThanOrEqual(parsed[i]);  // descending
}
```

### Empty States

```typescript
// BAD
await expect(page.locator('.empty-state')).toBeVisible();

// GOOD — exact message, and prove the list is actually empty
await expect(page.locator('[data-testid="empty-state"]')).toHaveText(
  'No invoices found. Create your first invoice to get started.'
);
await expect(page.locator('[data-testid="invoice-row"]')).toHaveCount(0);
```

### Totals and Summaries

Assert the values, then assert the invariant that ties them together — that catches
rendering bugs where each figure is individually plausible.

```typescript
await expect(page.locator('[data-testid="subtotal"]')).toHaveText('$1,200.00');
await expect(page.locator('[data-testid="tax"]')).toHaveText('$96.00');
await expect(page.locator('[data-testid="total"]')).toHaveText('$1,296.00');

// Consistency: subtotal + tax === total
const money = async (id: string) =>
  parseFloat((await page.locator(`[data-testid="${id}"]`).textContent())!.replace(/[$,]/g, ''));

expect(await money('subtotal') + await money('tax')).toBeCloseTo(await money('total'), 2);
```

## Visual State Assertions

### Text Content Over Visibility

```typescript
// BAD
await expect(page.locator('.status-badge')).toBeVisible();

// GOOD
await expect(page.locator('[data-testid="status-badge"]')).toHaveText('Paid');
```

### Disabled / Enabled Transitions

```typescript
const submit = page.getByRole('button', { name: 'Create invoice' });

// BAD — assumes the button is clickable
await submit.click();

// GOOD — assert the gate before and after
await expect(submit).toBeDisabled();

await page.getByLabel('Customer name').fill('Acme Corp');
await page.getByLabel('Amount').fill('100.00');

await expect(submit).toBeEnabled();
```

### Classes and Data Attributes

```typescript
await expect(page.locator('[data-testid="invoice-row"]')).toHaveAttribute('data-status', 'paid');

// Error state — assert what assistive tech sees, not only the styling
await expect(page.getByLabel('Email')).toHaveAttribute('aria-invalid', 'true');
await expect(page.getByLabel('Email')).toHaveClass(/error/);
```

### Selected / Active States

Assert the selected item **and** that the previously-selected one deselected.

```typescript
// BAD — clicks the tab, verifies nothing
await page.getByRole('tab', { name: 'Settings' }).click();

// GOOD
await page.getByRole('tab', { name: 'Settings' }).click();
await expect(page.getByRole('tab', { name: 'Settings' })).toHaveAttribute('aria-selected', 'true');
await expect(page.getByRole('tab', { name: 'Overview' })).toHaveAttribute('aria-selected', 'false');
```

## User Flow Assertions

### Form Submission Success

```typescript
// BAD
await page.getByLabel('Name').fill('Test Customer');
await page.getByRole('button', { name: 'Save' }).click();
await expect(page.locator('.success')).toBeVisible();

// GOOD — message, navigation, and the resulting data
await page.getByLabel('Name').fill('Test Customer');
await page.getByLabel('Email').fill('test@customer.com');
await page.getByRole('button', { name: 'Save' }).click();

await expect(page.getByRole('status')).toHaveText('Customer created successfully');
await expect(page).toHaveURL(/\/customers\/[a-f0-9-]+$/);
await expect(page.locator('[data-testid="customer-name"]').first()).toHaveText('Test Customer');
```

### Form State After Submit

```typescript
await page.getByLabel('Customer').fill('Acme Corp');
await page.getByLabel('Amount').fill('500.00');
await page.getByRole('button', { name: 'Create invoice' }).click();

await expect(page.getByRole('status')).toHaveText('Invoice created successfully');

// If the form should clear:
await expect(page.getByLabel('Customer')).toHaveValue('');
await expect(page.getByLabel('Amount')).toHaveValue('');

// If it should show the created item:
await expect(page.locator('[data-testid="invoice-number"]')).toContainText('INV-');
```

### Loading State Transitions

```typescript
await page.getByRole('button', { name: 'Load data' }).click();

await expect(page.locator('[data-testid="loading-spinner"]')).toBeVisible();
await expect(page.locator('[data-testid="loading-spinner"]')).toBeHidden();

// Final state, with content
await expect(page.locator('[data-testid="data-table"] tbody tr')).toHaveCount(5);
```

### UI Reflects the Change

Assert the before state, act, then assert both the direct change and the secondary UI
consequences.

```typescript
await expect(page.locator('[data-testid="invoice-status"]')).toHaveText('Draft');

await page.getByRole('button', { name: 'Send invoice' }).click();

await expect(page.locator('[data-testid="invoice-status"]')).toHaveText('Sent');

// Related affordances updated
await expect(page.getByRole('button', { name: 'Send invoice' })).toBeHidden();
await expect(page.getByRole('button', { name: 'Record payment' })).toBeVisible();
```

## Form & Input Assertions

### Validation Messages

```typescript
// BAD
await page.getByRole('button', { name: 'Save' }).click();
await expect(page.locator('.error')).toBeVisible();

// GOOD — per-field, exact text, and tied to its field. The accessible description is
// what a screen reader reads with the field, so this proves the text *and* the wiring.
await page.getByRole('button', { name: 'Save' }).click();

await expect(page.getByLabel('Name')).toHaveAccessibleDescription('Name is required');
await expect(page.getByLabel('Email')).toHaveAccessibleDescription('Please enter a valid email address');
await expect(page.getByLabel('Amount')).toHaveAccessibleDescription('Amount must be greater than 0');
```

### Errors Clear After Correction

```typescript
await page.getByLabel('Email').fill('invalid');
await page.getByRole('button', { name: 'Save' }).click();
await expect(page.getByLabel('Email')).toHaveAccessibleDescription('Please enter a valid email address');

await page.getByLabel('Email').fill('valid@email.com');

await expect(page.getByLabel('Email')).toHaveAccessibleDescription('');
await expect(page.getByLabel('Email')).not.toHaveAttribute('aria-invalid', 'true');
```

### Prefill Verification

```typescript
await page.goto(`/customers/${customerId}/edit`);

await expect(page.getByLabel('Name')).toHaveValue('Acme Corp');
await expect(page.getByLabel('Email')).toHaveValue('billing@acme.com');
await expect(page.getByLabel('Credit limit')).toHaveValue('10000.00');
await expect(page.getByLabel('Status')).toHaveValue('active');
```

## Navigation & URL Assertions

```typescript
// BAD — clicks, never verifies navigation
await page.getByRole('link', { name: 'View details' }).click();

// GOOD
await page.getByRole('link', { name: 'View details' }).click();
await expect(page).toHaveURL(`/invoices/${invoiceId}`);

// For dynamic IDs, constrain the shape
await expect(page).toHaveURL(/\/invoices\/[a-f0-9-]{36}$/);
```

Verify the page actually rendered, not only that the URL changed — a client-side route
can change the URL and then fail to load.

```typescript
await page.getByRole('link', { name: 'Settings' }).click();

await expect(page).toHaveURL('/settings');
await expect(page).toHaveTitle('Settings | MyApp');
await expect(page.getByRole('heading', { level: 1 })).toHaveText('Settings');
```

Redirects after an action:

```typescript
await page.getByLabel('Username').fill('testuser');
await page.getByLabel('Password').fill('testpass');
await page.getByRole('button', { name: 'Log in' }).click();

await expect(page).toHaveURL('/dashboard');
await expect(page.locator('[data-testid="welcome-message"]')).toContainText('Welcome, testuser');
```

## Accessibility Assertions

These are the tests behind a UI story's **Accessibility** criteria. Role and label
locators already prove names and roles as a side effect; the patterns below cover what
they don't — focus, keyboard, and announcements.

### Focus Management

Assert where focus lands, not just that a dialog opened.

```typescript
// BAD — the dialog is visible, but keyboard users may still be on the page behind it
await page.getByRole('button', { name: 'Delete invoice' }).click();
await expect(page.getByRole('dialog')).toBeVisible();

// GOOD — focus moves in, Escape closes, focus returns to the trigger
const trigger = page.getByRole('button', { name: 'Delete invoice' });
await trigger.click();

const dialog = page.getByRole('dialog', { name: 'Delete INV-001?' });
await expect(dialog).toBeVisible();
await expect(dialog.getByRole('button', { name: 'Cancel' })).toBeFocused();

await page.keyboard.press('Escape');
await expect(dialog).toBeHidden();
await expect(trigger).toBeFocused();
```

Same idea after an SPA route change: assert focus moved to the new page's heading.

```typescript
await page.getByRole('link', { name: 'Settings' }).click();
await expect(page.getByRole('heading', { level: 1, name: 'Settings' })).toBeFocused();
```

### Keyboard-Only Flow

When the story says a flow must work without a mouse, drive it without one.

```typescript
await page.getByLabel('Customer name').fill('Acme Corp');
await page.keyboard.press('Tab');
await expect(page.getByLabel('Amount')).toBeFocused();

await page.keyboard.type('100.00');
await page.keyboard.press('Enter');

await expect(page.getByRole('status')).toHaveText('Invoice created successfully');
```

### Announcements

A toast that's visible but not in a live region is silent to screen-reader users.
`getByRole('status')` matches the `status` role — a polite live region — so it proves the
message is announced (`'alert'` for errors). If the project's announcer is a bare
`aria-live` element with no role, assert that attribute instead.

```typescript
// BAD — visible, but is it announced?
await expect(page.locator('.toast')).toBeVisible();

// GOOD — exact text, inside a live region
await expect(page.getByRole('status')).toHaveText('Invoice INV-001 sent');
```

### Automated Scan

Only if `@axe-core/playwright` is already a project dependency — don't add it to write a
test. Scope the scan to what the change touched, and assert the empty list so a failure
prints the violations.

```typescript
import AxeBuilder from '@axe-core/playwright';

const results = await new AxeBuilder({ page })
  .include('[data-testid="invoice-form"]')
  .withTags(['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa', 'wcag22aa'])
  .analyze();

expect(results.violations).toEqual([]);
```

A clean scan catches only a fraction of real barriers. It complements the focus,
keyboard, and announcement assertions above; it never replaces them.

## Anti-Patterns and Fixes

### 1. Count-Only

```typescript
// BAD
await expect(page.locator('[data-testid="item"]')).toHaveCount(3);

// GOOD
const items = page.locator('[data-testid="item"]');
await expect(items).toHaveCount(3);
await expect(items.nth(0)).toContainText('Expected Item 1');
await expect(items.nth(1)).toContainText('Expected Item 2');
await expect(items.nth(2)).toContainText('Expected Item 3');
```

### 2. Visibility-Only for Messages

```typescript
// BAD — an error toast satisfies this too
await expect(page.locator('.success-message')).toBeVisible();

// GOOD
await expect(page.getByRole('status')).toHaveText('Invoice INV-001 created successfully');
```

### 3. Missing State Verification After an Action

```typescript
// BAD — deletes but never verifies removal
await page.getByRole('button', { name: 'Delete invoice' }).click();
await page.getByRole('button', { name: 'Confirm delete' }).click();

// GOOD — prove it was there, then prove it's gone
const invoiceRow = page.locator(`[data-testid="invoice-row-${invoiceId}"]`);
await expect(invoiceRow).toBeVisible();

await page.getByRole('button', { name: 'Delete invoice' }).click();
await page.getByRole('button', { name: 'Confirm delete' }).click();

await expect(invoiceRow).toBeHidden();
await expect(page.locator('[data-testid="invoice-row"]')).toHaveCount(2);  // was 3
```

### 4. Filters Tested Only Positively

```typescript
// BAD — count alone; doesn't prove the filter excluded anything
await page.getByLabel('Status').selectOption('paid');
await expect(page.locator('[data-testid="invoice-row"]')).toHaveCount(2);

// GOOD — inclusions AND exclusions
await page.getByLabel('Status').selectOption('paid');

await expect(page.locator('[data-testid="invoice-INV-001"]')).toBeVisible();
await expect(page.locator('[data-testid="invoice-INV-003"]')).toBeVisible();
await expect(page.locator('[data-testid="invoice-INV-002"]')).toBeHidden();  // unpaid
await expect(page.locator('[data-testid="invoice-row"]')).toHaveCount(2);
```

### 5. Arbitrary Waits

A fixed sleep is both slow and unreliable — it hides real races and passes when nothing
happened at all.

```typescript
// BAD
await page.getByRole('button', { name: 'Save' }).click();
await page.waitForTimeout(2000);
// assumes success...

// GOOD — wait on the condition, then assert it
await page.getByRole('button', { name: 'Save' }).click();
await expect(page.getByRole('status')).toHaveText('Saved successfully', { timeout: 5000 });
```

## Complete Examples

### Create and Verify in List

```typescript
test('create invoice appears in invoice list', async ({ page }) => {
  await page.goto('/invoices/new');

  await page.getByLabel('Customer').selectOption('Acme Corp');
  await page.getByLabel('Line item 1 description').fill('Consulting Services');
  await page.getByLabel('Line item 1 amount').fill('1500.00');

  await page.getByRole('button', { name: 'Create invoice' }).click();

  // Success feedback, exact text, announced
  await expect(page.getByRole('status')).toHaveText('Invoice created successfully');

  // Navigation
  await expect(page).toHaveURL(/\/invoices\/[a-f0-9-]+$/);

  // Detail view shows the right data
  await expect(page.locator('[data-testid="invoice-customer"]')).toHaveText('Acme Corp');
  await expect(page.locator('[data-testid="invoice-total"]')).toHaveText('$1,500.00');
  await expect(page.locator('[data-testid="invoice-status"]')).toHaveText('Draft');

  // And it shows up in the list
  await page.goto('/invoices');
  const row = page.getByRole('row').filter({ hasText: 'Acme Corp' });
  await expect(row).toBeVisible();
  await expect(row).toContainText('$1,500.00');
});
```

### Filter and Verify Results

```typescript
test('filter invoices by date range shows correct results', async ({ page }) => {
  // Setup: January → INV-001, INV-002.  February → INV-003.
  await page.goto('/invoices');

  await page.getByLabel('From').fill('2024-01-01');
  await page.getByLabel('To').fill('2024-01-31');
  await page.getByRole('button', { name: 'Apply filters' }).click();

  await expect(page.locator('[data-testid="invoice-row"]')).toHaveCount(2);

  const invoiceNumbers = await page.locator('[data-testid="invoice-number"]').allTextContents();
  expect(invoiceNumbers).toContain('INV-001');
  expect(invoiceNumbers).toContain('INV-002');
  expect(invoiceNumbers).not.toContain('INV-003');

  // Every displayed row genuinely falls in range
  const dates = await page.locator('[data-testid="invoice-date"]').allTextContents();
  for (const dateStr of dates) {
    const date = new Date(dateStr);
    expect(date.getMonth()).toBe(0);        // January
    expect(date.getFullYear()).toBe(2024);
  }
});
```
