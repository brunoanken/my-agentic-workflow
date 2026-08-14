# API & Database Test Assertions

Guidance for asserting API responses and persisted state in backend tests. Examples
are in neutral pseudo-code — translate to the project's stack (Vitest, pytest, Go
testing, etc.).

> **These are defaults.** The user's instructions, `CLAUDE.md` / `AGENTS.md`, and the
> conventions of the test file you're editing all take precedence — see **Precedence** in
> [SKILL.md](./SKILL.md). The code below illustrates *what to assert*; match the project's
> existing style for *how* to assert it, and skip any section the task has ruled out.

Notation used below:
- `response = POST /api/x { ... }` — an HTTP request through the test client
- `db.table.find(id)` / `db.table.where(...)` — a fresh query against the database
- `reload(entity)` — re-read an entity from the database, discarding in-memory state

## CRUD Operation Assertions

### CREATE

After creating an entity, verify all four:

1. Input fields persisted correctly
2. Auto-generated fields have expected values (IDs, timestamps, sequences)
3. Default values applied
4. Related entities created (line items, log entries, junction rows)

```
test "create customer" {
    response = POST /api/customers {
        name: "Acme Corp",
        email: "billing@acme.com",
        credit_limit: "5000.00"
    }

    assert response.status == 201

    // 1. Input reflected in response
    assert response.body.name == "Acme Corp"
    assert response.body.email == "billing@acme.com"
    assert response.body.credit_limit == "5000.00"

    // 2. Auto-generated fields
    assert response.body.id is not null
    assert response.body.created_at is not null

    // 3. Defaults applied
    assert response.body.status == "active"
    assert response.body.balance == "0.00"

    // 4. Persisted state matches the response
    customer = db.customers.find(response.body.id)
    assert customer.name == "Acme Corp"
    assert customer.credit_limit == 5000.00
}
```

### READ

Verify the complete object shape, not just that a record came back.

```
test "get customer returns complete data" {
    customer = create_customer(
        name: "Test Corp",
        email: "test@corp.com",
        credit_limit: 10000.00
    )

    response = GET /api/customers/{customer.id}

    assert response.status == 200
    assert response.body.id == customer.id
    assert response.body.name == "Test Corp"
    assert response.body.email == "test@corp.com"
    assert response.body.credit_limit == "10000.00"
    assert response.body.status == "active"
    assert response.body.created_at is not null
    assert response.body.updated_at is not null
}
```

### UPDATE

Verify the changed field **and** that unchanged fields held. A partial update that
silently blanks other columns is a common bug that a single-field assertion misses.

```
test "update customer preserves unchanged fields" {
    customer = create_customer(
        name: "Original Name",
        email: "original@email.com",
        credit_limit: 5000.00
    )
    original_created_at = customer.created_at

    // Update only the name
    response = PATCH /api/customers/{customer.id} { name: "Updated Name" }
    assert response.status == 200

    // Changed field
    assert response.body.name == "Updated Name"

    // Unchanged fields preserved
    assert response.body.email == "original@email.com"
    assert response.body.credit_limit == "5000.00"

    // Persisted state
    reload(customer)
    assert customer.name == "Updated Name"
    assert customer.email == "original@email.com"
    assert customer.credit_limit == 5000.00

    // Audit fields
    assert customer.created_at == original_created_at   // unchanged
    assert customer.updated_at > original_created_at    // bumped
}
```

### DELETE

Verify the delete semantics the system actually implements (soft vs hard), cascade
effects, and exclusion from normal queries.

```
test "delete invoice soft-deletes and cascades" {
    invoice = create_invoice_with_lines(line_count: 3)
    line_ids = invoice.lines.map(l => l.id)

    response = DELETE /api/invoices/{invoice.id}
    assert response.status == 204

    // Soft delete, not physical removal
    reload(invoice)
    assert invoice.deleted_at is not null

    // Cascaded to children
    for line_id in line_ids {
        line = db.invoice_lines.find(line_id)
        assert line.deleted_at is not null
    }

    // Excluded from normal queries
    active = db.invoices.where(deleted_at: null)
    assert invoice.id not in active.map(i => i.id)
}
```

## Derived State & Multi-Entity Assertions

The highest-value assertions are on state the system *computes* rather than state you
passed in. A create endpoint that echoes your input is easy; a create endpoint that
correctly updates a parent total, writes a ledger pair, and stamps an audit row is
where bugs live.

Rule: **assert every entity the action touches**, and assert the invariant that ties
them together.

### Parent Totals Recalculated

```
test "updating a line recalculates the invoice total" {
    invoice = create_invoice_with_lines([{ amount: 100.00 }, { amount: 50.00 }])
    assert invoice.total_amount == 150.00     // precondition, stated explicitly

    line = invoice.lines.first()
    response = PATCH /api/invoices/{invoice.id}/lines/{line.id} { amount: "200.00" }
    assert response.status == 200

    // The point of the test: the parent recomputed
    reload(invoice)
    assert invoice.total_amount == 250.00      // 200 + 50
}
```

### Balance Applied Across Entities

Capture state before the mutation, then assert the exact delta — not just that the
value moved.

```
test "apply payment to invoice" {
    invoice = create_invoice(total_amount: 500.00)
    assert invoice.remaining_balance == 500.00        // precondition

    response = POST /api/payments {
        customer_id: invoice.customer_id,
        amount: "200.00",
        allocations: [{ invoice_id: invoice.id, amount: "200.00" }]
    }

    // Payment record
    assert response.status == 201
    assert response.body.amount == "200.00"
    payment_id = response.body.id

    // Target entity updated by the exact amount
    reload(invoice)
    assert invoice.remaining_balance == 300.00
    assert invoice.applied_amount == 200.00

    // Join/allocation record exists with the right amount
    allocation = db.payment_allocations.find_by(
        payment_id: payment_id, invoice_id: invoice.id
    )
    assert allocation.amount == 200.00
}
```

### Paired / Balanced Records

When an action must write records that balance (double-entry ledger, debit/credit,
inventory in/out), assert both sides *and* the invariant.

```
test "creating an invoice writes balanced ledger entries" {
    response = POST /api/invoices {
        customer_id: customer.id,
        lines: [
            { description: "Service", amount: "100.00", account_id: revenue.id },
            { description: "Product", amount: "50.00",  account_id: revenue.id }
        ]
    }

    assert response.status == 201
    assert response.body.total_amount == "150.00"
    invoice_id = response.body.id

    transaction = db.ledger_transactions.find_by(
        source_id: invoice_id, source_type: "invoice"
    )
    lines = transaction.lines

    // Each side, specifically
    receivable_line = lines.find(l => l.account_id == receivable.id)
    assert receivable_line.debit_amount == 150.00
    assert receivable_line.credit_amount in [null, 0]

    revenue_credit = sum(lines.where(account_id: revenue.id).map(l => l.credit_amount or 0))
    assert revenue_credit == 150.00

    // The invariant
    assert sum(lines.map(l => l.debit_amount or 0)) ==
           sum(lines.map(l => l.credit_amount or 0))
}
```

### Sign Conventions

For reversals, credits, refunds, and adjustments, assert the *direction*, not just
the magnitude. A sign bug produces the right number in the wrong column.

```
test "credit memo reverses the receivable direction" {
    response = POST /api/credit-memos { customer_id: customer.id, amount: "100.00" }
    assert response.status == 201

    lines = db.ledger_transactions.find_by(source_id: response.body.id).lines

    // Credit to receivable — reduces what's owed
    receivable_line = lines.find(l => l.account_type == "accounts_receivable")
    assert receivable_line.credit_amount == 100.00

    // Debit to the contra account
    contra_line = lines.find(l => l.account_type != "accounts_receivable")
    assert contra_line.debit_amount == 100.00
}
```

### Precision

Assert exact values on anything that goes through arithmetic. Floating-point drift and
rounding bugs pass any tolerance-based check.

```
test "decimal precision preserved through summation" {
    response = POST /api/invoices {
        customer_id: customer.id,
        lines: [{ amount: "33.33" }, { amount: "33.33" }, { amount: "33.34" }]
    }

    assert response.body.total_amount == "100.00"

    invoice = db.invoices.find(response.body.id)
    assert invoice.total_amount == 100.00     // exactly, not 99.99999...
}
```

## List / Filter Endpoint Assertions

### Assertion Priority

1. **Primary** — the IDs you created are present in the response
2. **Secondary** — the count matches
3. **Tertiary** — response structure and field shapes are correct

Count-only assertions pass when old test data lingers, when create silently failed but
unrelated records match, and when a coincidental record from a parallel test slips in.
They are the single most common weak assertion in list tests.

```
// BAD — passes with stale or coincidental data
response = GET /api/items
assert len(response.items) >= 3
```

### Verify Returned Items and Exclusions

A filter test that only checks the included items doesn't test the filter — it tests
that the endpoint returns something. Assert both directions.

```
test "list invoices filters by customer" {
    invoice1 = create_invoice(customer: customer_a)
    invoice2 = create_invoice(customer: customer_a)
    invoice3 = create_invoice(customer: customer_b)      // must NOT appear

    response = GET /api/invoices?customer_id={customer_a.id}
    assert response.status == 200

    returned_ids = {r.id for r in response.body.results}

    // Inclusions
    assert invoice1.id in returned_ids
    assert invoice2.id in returned_ids

    // Exclusion — this is what proves the filter works
    assert invoice3.id not in returned_ids

    // Count, secondary
    assert len(response.body.results) == 2
}
```

### Verify Ordering

Assert position, not just membership.

```
test "list invoices ordered by date desc" {
    old_invoice = create_invoice(date: "2024-01-01")
    new_invoice = create_invoice(date: "2024-03-01")
    mid_invoice = create_invoice(date: "2024-02-01")

    response = GET /api/invoices?ordering=-date
    assert response.status == 200

    result_ids = response.body.results.map(r => r.id)
    assert result_ids[0] == new_invoice.id
    assert result_ids[1] == mid_invoice.id
    assert result_ids[2] == old_invoice.id
}
```

### Verify Pagination

Assert the metadata, the page size, and — critically — that pages don't overlap.

```
test "list invoices pagination" {
    invoices = [create_invoice() for _ in range(25)]

    page1 = GET /api/invoices?page_size=10
    assert page1.status == 200
    assert page1.body.count == 25
    assert len(page1.body.results) == 10
    assert page1.body.next is not null
    assert page1.body.previous is null

    page2 = GET /api/invoices?page_size=10&page=2
    assert len(page2.body.results) == 10
    assert page2.body.previous is not null

    // No overlap — catches off-by-one offset bugs
    page1_ids = {r.id for r in page1.body.results}
    page2_ids = {r.id for r in page2.body.results}
    assert page1_ids.isdisjoint(page2_ids)
}
```

### Test Data Isolation for List Tests

Integration tests share a database. Scope list assertions to a parent entity you
created in this test.

```
test "filter payments by method" {
    // Isolated parent for this test
    vendor = POST /api/vendors { name: "Vendor {timestamp}" }

    payment_a = POST /api/payments { vendor_id: vendor.id, method: "CHECK" }
    payment_b = POST /api/payments { vendor_id: vendor.id, method: "ACH" }

    // Query scoped to our vendor — immune to other tests' data
    response = GET /api/payments?vendor_id={vendor.id}&method=CHECK

    assert payment_a.id in response.body.map(p => p.id)
    assert payment_b.id not in response.body.map(p => p.id)
    assert len(response.body) == 1
}
```

## Database State Assertions

### Don't Trust the Response Alone

An endpoint can return a value it never wrote — serializing from the in-memory object
instead of the committed row, or committing in a transaction that later rolls back.

```
test "update persists to database" {
    customer = create_customer(name: "Original")

    response = PATCH /api/customers/{customer.id} { name: "Updated" }
    assert response.status == 200
    assert response.body.name == "Updated"

    // Fresh query, not the response
    db_customer = db.customers.find(customer.id)
    assert db_customer.name == "Updated"
}
```

### Audit Fields

Freeze or inject time so timestamps are assertable exactly rather than with `is not null`.

```
test "update sets audit fields" {
    with frozen_time("2024-01-01 10:00:00") {
        customer = create_customer()
    }

    with frozen_time("2024-01-02 15:00:00") {
        PATCH /api/customers/{customer.id} { name: "Updated" } as user
    }

    reload(customer)
    assert customer.created_at == "2024-01-01 10:00:00 UTC"    // unchanged
    assert customer.updated_at == "2024-01-02 15:00:00 UTC"    // bumped
    assert customer.updated_by == user.id                      // if tracked
}
```

## Anti-Patterns and Fixes

### 1. Status-Only Assertion

```
// BAD — what was actually created?
test "create invoice" {
    response = POST /api/invoices { ...invoice_data }
    assert response.status == 201
}

// GOOD
test "create invoice" {
    response = POST /api/invoices { ...invoice_data }
    assert response.status == 201
    assert response.body.customer_id == invoice_data.customer_id
    assert response.body.total_amount == expected_total
    assert response.body.id is not null

    invoice = db.invoices.find(response.body.id)
    assert invoice.total_amount == expected_total
}
```

### 2. Missing Error Case Detail

A bare `assert status == 400` passes when the endpoint rejects the request for
completely the wrong reason.

```
// BAD — which error?
test "create invoice requires customer" {
    response = POST /api/invoices {}
    assert response.status == 400
}

// GOOD
test "create invoice requires customer" {
    response = POST /api/invoices {}
    assert response.status == 400
    assert "customer_id" in response.body.errors
    assert response.body.errors.customer_id[0] == "This field is required."
}
```

### 3. Ignoring Side Effects

```
// BAD — deletes the invoice but never checks the ledger reversal
test "delete invoice" {
    invoice = create_posted_invoice()
    response = DELETE /api/invoices/{invoice.id}
    assert response.status == 204
}

// GOOD
test "delete posted invoice writes a reversal" {
    invoice = create_posted_invoice()
    original = db.ledger_transactions.find_by(source_id: invoice.id)

    response = DELETE /api/invoices/{invoice.id}
    assert response.status == 204

    reversal = db.ledger_transactions.find_by(
        source_id: invoice.id, transaction_type: "reversal"
    )
    assert reversal.lines.count == original.lines.count

    // Amounts are mirrored, not merely present
    for orig in original.lines {
        rev = reversal.lines.find(l => l.account_id == orig.account_id)
        assert rev.debit_amount == orig.credit_amount
        assert rev.credit_amount == orig.debit_amount
    }
}
```

### 4. Truthy Check on Collections

```
// BAD — only proves the list isn't empty
test "list invoices" {
    create_invoice()
    create_invoice()
    response = GET /api/invoices
    assert response.body.results        // truthy
}

// GOOD
test "list invoices" {
    inv1 = create_invoice()
    inv2 = create_invoice()
    response = GET /api/invoices

    returned_ids = {r.id for r in response.body.results}
    assert returned_ids == {inv1.id, inv2.id}
    assert len(response.body.results) == 2
}
```

### 5. Unstated Preconditions

If the test's point is that a value *changed*, assert the starting value too. Otherwise
a test on already-correct data passes without exercising the logic.

```
// BAD — was it ever 500?
apply_payment(invoice, 200.00)
assert invoice.remaining_balance == 300.00

// GOOD
assert invoice.remaining_balance == 500.00      // precondition
apply_payment(invoice, 200.00)
reload(invoice)
assert invoice.remaining_balance == 300.00      // exact delta
```
