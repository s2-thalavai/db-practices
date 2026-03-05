
# Range Partitioning (Best for Transaction Tables)

Range partitioning is ideal for **time-based financial data**.

Typical tables:

-   payments
    
-   invoices
    
-   purchase_orders
    
-   ledger_entries
    
-   audit logs
    

Example table:

```sql
CREATE  TABLE payments (  
 payment_id BIGSERIAL,  
 supplier_id BIGINT,  
 amount NUMERIC,  
 created_at DATE  
) PARTITION BY RANGE (created_at);
```

### Partitions:

```sql
CREATE  TABLE payments_2024  
PARTITION OF payments  
FOR  VALUES  FROM ('2024-01-01') TO ('2025-01-01');  
  
CREATE  TABLE payments_2025  
PARTITION OF payments  
FOR  VALUES  FROM ('2025-01-01') TO ('2026-01-01');
```

### Why Range Works Well

Most P2P queries look like:

```sql
SELECT  *  
FROM payments  
WHERE created_at >=  '2025-01-01';
```

PostgreSQL performs **partition pruning**, scanning only relevant partitions.

Benefits:

-   fast reporting
    
-   fast archiving
    
-   easy year/month cleanup
    

### Example archive:

```sql
DROP  TABLE payments_2022;
```

---------------------


# List Partitioning (Best for Organizational Segmentation)

List partitioning is good when data is separated by **business categories**.

Common P2P use cases:

-   company
    
-   region
    
-   business unit
    
-   cost center
    

Example:

```sql
CREATE  TABLE invoices (  
 invoice_id BIGINT,  
 company_code TEXT,  
 amount NUMERIC  
) PARTITION BY LIST (company_code);
```

Partitions:

```sql
CREATE  TABLE invoices_us  
PARTITION OF invoices  
FOR  VALUES  IN ('US');  
  
CREATE  TABLE invoices_india  
PARTITION OF invoices  
FOR  VALUES  IN ('IN');  
  
CREATE  TABLE invoices_eu  
PARTITION OF invoices  
FOR  VALUES  IN ('EU');
```

Benefits:

-   data isolation by region/company
    
-   easier compliance (e.g., EU data residency)
    
-   faster regional queries


For a Procure-to-Pay system:

-   **Range partitioning** is used for transaction tables like payments and ledger entries based on date.
    
-   **List partitioning** is used for organizational separation such as company or region.
    
-   **Hash partitioning** is used for high-volume tables like purchase orders to distribute writes evenly.

  
------------

```sql
CREATE TABLE suppliers (
    supplier_id BIGSERIAL PRIMARY KEY,
    supplier_name TEXT,
    tax_id TEXT,
    created_at TIMESTAMP DEFAULT now()
);

CREATE INDEX idx_supplier_tax
ON suppliers(tax_id);
```

## Purchase Orders (High Write Table)

Use hash partitioning for balanced inserts.

```sql
CREATE TABLE purchase_orders (
    po_id BIGSERIAL,
    supplier_id BIGINT,
    total_amount NUMERIC,
    created_at TIMESTAMP,
    status TEXT,
    PRIMARY KEY (po_id)
) PARTITION BY HASH (supplier_id);
```

## Example partitions:

```sql
CREATE TABLE purchase_orders_p1
PARTITION OF purchase_orders
FOR VALUES WITH (MODULUS 4, REMAINDER 0);

CREATE TABLE purchase_orders_p2
PARTITION OF purchase_orders
FOR VALUES WITH (MODULUS 4, REMAINDER 1);
```


## Invoices Table

Invoices grow quickly, so use range partitioning by date.

```sql
CREATE TABLE invoices (
    invoice_id BIGSERIAL,
    supplier_id BIGINT,
    po_id BIGINT,
    amount NUMERIC,
    invoice_date DATE,
    created_at TIMESTAMP
) PARTITION BY RANGE (invoice_date);
```

## Example partitions:

```sql
CREATE TABLE invoices_2025_01
PARTITION OF invoices
FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');
```

## Index:

```sql
CREATE INDEX idx_invoice_supplier
ON invoices(supplier_id);
```
--------------

## Payments Table

Payments are critical financial transactions.

```
CREATE TABLE payments (
    payment_id BIGSERIAL,
    invoice_id BIGINT,
    amount NUMERIC,
    payment_method TEXT,
    payment_status TEXT,
    created_at TIMESTAMP
) PARTITION BY RANGE (created_at);
```

## Example partitions:

```
payments_2025_01
payments_2025_02
payments_2025_03
```

-------------------------

## Ledger Table (Source of Truth)

Ledger is append-only and uses double-entry accounting.

```sql
CREATE TABLE ledger_entries (
    entry_id BIGSERIAL,
    transaction_id UUID,
    account_id BIGINT,
    cost_center TEXT,
    amount NUMERIC,
    created_at TIMESTAMP
) PARTITION BY RANGE (created_at);
```

Example partitions:

```
ledger_2025_01
ledger_2025_02
ledger_2025_03
```

Indexes:

```sql
CREATE INDEX idx_ledger_account
ON ledger_entries(account_id);

CREATE INDEX idx_ledger_tx
ON ledger_entries(transaction_id);
```

------------------

## Double Entry Enforcement

Every financial transaction must balance.

## Example logic:

```
Transaction TX1
Account A → -100
Account B → +100
```

## Validation:

```
SUM(amount) = 0
```

-------------------

## Budget Projection Table

Used by Budget Service.

```sql
CREATE TABLE budget_projection (
    cost_center TEXT PRIMARY KEY,
    budget_limit NUMERIC,
    spent_amount NUMERIC,
    remaining_budget NUMERIC
);
```

## Updated using ledger events.

10. Audit Log Table

Required for compliance.

```sql
CREATE TABLE audit_log (
    log_id BIGSERIAL,
    entity_type TEXT,
    entity_id BIGINT,
    action TEXT,
    performed_by TEXT,
    created_at TIMESTAMP DEFAULT now()
);
```

## 11. Idempotency Table

Prevents duplicate payments.

```sql
CREATE TABLE idempotency_keys (
    key TEXT PRIMARY KEY,
    created_at TIMESTAMP
);
```

Application enforces this rule before insert.

---------

| Table           | Strategy                 |
| --------------- | ------------------------ |
| ledger_entries  | Range partition by month |
| payments        | Range partition by month |
| invoices        | Range partition by month |
| purchase_orders | Hash partition           |
| audit_log       | Range partition          |

-------------

| Query                        | Index          |
| ---------------------------- | -------------- |
| Find invoices by supplier    | supplier_id    |
| Find payments by invoice     | invoice_id     |
| Ledger lookup by transaction | transaction_id |
| Ledger by account            | account_id     |

--------

## Example Query Patterns

Supplier invoice history

```sql
SELECT *
FROM invoices
WHERE supplier_id = 100
ORDER BY invoice_date DESC;
```

## Payment reconciliation

```sql
SELECT *
FROM payments
WHERE payment_status = 'FAILED';
```

## Ledger balance

```sql
SELECT SUM(amount)
FROM ledger_entries
WHERE account_id = 101;
```
--------

