# UI & Playwright Test Assertions

Guidance for asserting UI state in browser tests. Examples use Playwright notation
because UI assertions need a concrete locator API — the patterns transfer directly to
Cypress, Testing Library, and other runners.

> **These are defaults.** The user's instructions, `CLAUDE.md` / `AGENTS.md`, and the
> conventions of the test file you're editing all take precedence — see **Precedence** in
> [SKILL.md](./SKILL.md). In particular: if the project uses a different runner or a
> different locator strategy (roles over `data-testid`, say), follow theirs. The `data-testid`
> selectors below are illustrative, not prescriptive.

Two things to balance: **data verification** (the correct content is displayed) and
**flow verification** (the UI responds correctly to interaction). Most weak UI tests
cover neither — they assert that an element exists.

The core failure mode in UI tests: `toBeVisible()` as a stand-in for a real assertion.
An element being present says nothing about whether it shows the right thing.

## Data Display Assertions

### Table / List Content

```typescript
// BAD — only verifies count
await expect(page.locator('table tbody tr')).toHaveCount(3);

// GOOD — count AND content
await expect(page.locator('table tbody tr')).toHaveCount(3);

const rows = page.locator('table tbody tr');
await expect(rows.nth(0).locator('td').nth(0)).toHaveText('INV-001');
await expect(rows.nth(0).locator('td').nth(1)).toHaveText('Acme Corp');
await expect(rows.nth(0).locator('td').nth(2)).toHaveText('$1,500.00');

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
await expect(page.locator('.total-amount')).toHaveText('$1,500.00');

// Better still: derive the expectation from the test fixture
const invoice = await createTestInvoice({ totalAmount: 1500 });
await page.goto(`/invoices/${invoice.id}`);
await expect(page.locator('[data-testid="total-amount"]')).toHaveText('$1,500.00');
```

### Sorting

Clicking a sort control and asserting the table still renders proves nothing.

```typescript
// BAD
await page.click('[data-testid="sort-by-date"]');
await expect(page.locator('table')).toBeVisible();

// GOOD — verify the actual order
await page.click('[data-testid="sort-by-date"]');

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
await expect(page.locator('table tbody tr')).toHaveCount(0);
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
// BAD — assumes the button is clickable
await page.click('#submit-button');

// GOOD — assert the gate before and after
await expect(page.locator('#submit-button')).toBeDisabled();

await page.fill('#customer-name', 'Acme Corp');
await page.fill('#amount', '100.00');

await expect(page.locator('#submit-button')).toBeEnabled();
```

### Classes and Data Attributes

```typescript
await expect(page.locator('[data-testid="invoice-row"]')).toHaveAttribute('data-status', 'paid');
await expect(page.locator('.invoice-row')).toHaveClass(/paid/);

// Error state
await expect(page.locator('#email-input')).toHaveAttribute('aria-invalid', 'true');
await expect(page.locator('#email-input')).toHaveClass(/error/);
```

### Selected / Active States

Assert the selected item **and** that the previously-selected one deselected.

```typescript
// BAD — clicks the tab, verifies nothing
await page.click('[data-testid="tab-settings"]');

// GOOD
await page.click('[data-testid="tab-settings"]');
await expect(page.locator('[data-testid="tab-settings"]')).toHaveAttribute('aria-selected', 'true');
await expect(page.locator('[data-testid="tab-overview"]')).toHaveAttribute('aria-selected', 'false');
```

## User Flow Assertions

### Form Submission Success

```typescript
// BAD
await page.fill('#name', 'Test Customer');
await page.click('#submit');
await expect(page.locator('.success')).toBeVisible();

// GOOD — message, navigation, and the resulting data
await page.fill('#name', 'Test Customer');
await page.fill('#email', 'test@customer.com');
await page.click('#submit');

await expect(page.locator('[data-testid="success-message"]')).toHaveText(
  'Customer created successfully'
);
await expect(page).toHaveURL(/\/customers\/[a-f0-9-]+$/);
await expect(page.locator('[data-testid="customer-name"]').first()).toHaveText('Test Customer');
```

### Form State After Submit

```typescript
await page.fill('#invoice-customer', 'Acme Corp');
await page.fill('#invoice-amount', '500.00');
await page.click('#create-invoice');

await expect(page.locator('[data-testid="success-toast"]')).toBeVisible();

// If the form should clear:
await expect(page.locator('#invoice-customer')).toHaveValue('');
await expect(page.locator('#invoice-amount')).toHaveValue('');

// If it should show the created item:
await expect(page.locator('[data-testid="invoice-number"]')).toContainText('INV-');
```

### Loading State Transitions

```typescript
await page.click('#load-data');

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

await page.click('[data-testid="send-invoice-button"]');

await expect(page.locator('[data-testid="invoice-status"]')).toHaveText('Sent');

// Related affordances updated
await expect(page.locator('[data-testid="send-invoice-button"]')).toBeHidden();
await expect(page.locator('[data-testid="record-payment-button"]')).toBeVisible();
```

## Form & Input Assertions

### Validation Messages

```typescript
// BAD
await page.click('#submit');
await expect(page.locator('.error')).toBeVisible();

// GOOD — per-field, exact text
await page.click('#submit');

await expect(page.locator('#name-error')).toHaveText('Name is required');
await expect(page.locator('#email-error')).toHaveText('Please enter a valid email address');
await expect(page.locator('#amount-error')).toHaveText('Amount must be greater than 0');

// And that the error is programmatically associated with its field
await expect(page.locator('#email-input')).toHaveAttribute('aria-describedby', 'email-error');
```

### Errors Clear After Correction

```typescript
await page.fill('#email', 'invalid');
await page.click('#submit');
await expect(page.locator('#email-error')).toHaveText('Please enter a valid email address');

await page.fill('#email', 'valid@email.com');

await expect(page.locator('#email-error')).toBeHidden();
await expect(page.locator('#email-input')).not.toHaveClass(/error/);
```

### Prefill Verification

```typescript
await page.goto(`/customers/${customerId}/edit`);

await expect(page.locator('#name')).toHaveValue('Acme Corp');
await expect(page.locator('#email')).toHaveValue('billing@acme.com');
await expect(page.locator('#credit-limit')).toHaveValue('10000.00');
await expect(page.locator('#status')).toHaveValue('active');
```

## Navigation & URL Assertions

```typescript
// BAD — clicks, never verifies navigation
await page.click('[data-testid="view-details"]');

// GOOD
await page.click('[data-testid="view-details"]');
await expect(page).toHaveURL(`/invoices/${invoiceId}`);

// For dynamic IDs, constrain the shape
await expect(page).toHaveURL(/\/invoices\/[a-f0-9-]{36}$/);
```

Verify the page actually rendered, not only that the URL changed — a client-side route
can change the URL and then fail to load.

```typescript
await page.click('[data-testid="nav-settings"]');

await expect(page).toHaveURL('/settings');
await expect(page).toHaveTitle('Settings | MyApp');
await expect(page.locator('h1')).toHaveText('Settings');
```

Redirects after an action:

```typescript
await page.fill('#username', 'testuser');
await page.fill('#password', 'testpass');
await page.click('#login');

await expect(page).toHaveURL('/dashboard');
await expect(page.locator('[data-testid="welcome-message"]')).toContainText('Welcome, testuser');
```

## Anti-Patterns and Fixes

### 1. Count-Only

```typescript
// BAD
await expect(page.locator('.item')).toHaveCount(3);

// GOOD
await expect(page.locator('.item')).toHaveCount(3);
const items = page.locator('.item');
await expect(items.nth(0)).toContainText('Expected Item 1');
await expect(items.nth(1)).toContainText('Expected Item 2');
await expect(items.nth(2)).toContainText('Expected Item 3');
```

### 2. Visibility-Only for Messages

```typescript
// BAD — an error toast satisfies this too
await expect(page.locator('.success-message')).toBeVisible();

// GOOD
await expect(page.locator('[data-testid="success-message"]')).toHaveText(
  'Invoice INV-001 created successfully'
);
```

### 3. Missing State Verification After an Action

```typescript
// BAD — deletes but never verifies removal
await page.click('[data-testid="delete-invoice-btn"]');
await page.click('[data-testid="confirm-delete"]');

// GOOD — prove it was there, then prove it's gone
const invoiceRow = page.locator(`[data-testid="invoice-row-${invoiceId}"]`);
await expect(invoiceRow).toBeVisible();

await page.click('[data-testid="delete-invoice-btn"]');
await page.click('[data-testid="confirm-delete"]');

await expect(invoiceRow).toBeHidden();
await expect(page.locator('[data-testid="invoice-row"]')).toHaveCount(2);  // was 3
```

### 4. Filters Tested Only Positively

```typescript
// BAD — count alone; doesn't prove the filter excluded anything
await page.selectOption('#status-filter', 'paid');
await expect(page.locator('.invoice-row')).toHaveCount(2);

// GOOD — inclusions AND exclusions
await page.selectOption('#status-filter', 'paid');

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
await page.click('#submit');
await page.waitForTimeout(2000);
// assumes success...

// GOOD — wait on the condition, then assert it
await page.click('#submit');
await expect(page.locator('[data-testid="success-message"]')).toBeVisible({ timeout: 5000 });
await expect(page.locator('[data-testid="success-message"]')).toHaveText('Saved successfully');
```

## Complete Examples

### Create and Verify in List

```typescript
test('create invoice appears in invoice list', async ({ page }) => {
  await page.goto('/invoices/new');

  await page.selectOption('#customer', 'Acme Corp');
  await page.fill('#line-item-1-description', 'Consulting Services');
  await page.fill('#line-item-1-amount', '1500.00');

  await page.click('[data-testid="create-invoice-btn"]');

  // Success feedback, exact text
  await expect(page.locator('[data-testid="toast"]')).toHaveText('Invoice created successfully');

  // Navigation
  await expect(page).toHaveURL(/\/invoices\/[a-f0-9-]+$/);

  // Detail view shows the right data
  await expect(page.locator('[data-testid="invoice-customer"]')).toHaveText('Acme Corp');
  await expect(page.locator('[data-testid="invoice-total"]')).toHaveText('$1,500.00');
  await expect(page.locator('[data-testid="invoice-status"]')).toHaveText('Draft');

  // And it shows up in the list
  await page.goto('/invoices');
  const row = page.locator('[data-testid="invoice-row"]').filter({ hasText: 'Acme Corp' });
  await expect(row).toBeVisible();
  await expect(row).toContainText('$1,500.00');
});
```

### Filter and Verify Results

```typescript
test('filter invoices by date range shows correct results', async ({ page }) => {
  // Setup: January → INV-001, INV-002.  February → INV-003.
  await page.goto('/invoices');

  await page.fill('#date-from', '2024-01-01');
  await page.fill('#date-to', '2024-01-31');
  await page.click('[data-testid="apply-filters"]');

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
