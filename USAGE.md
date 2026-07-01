# Usage & Examples

How the tool is used day-to-day, with anonymized example interactions. All data
below is fabricated for illustration.

---

## Desktop app

1. Launch the app. A local model server is already running on-premise.
2. Type a question in plain English and press **Query** (or Enter).
3. The result appears as a sortable, formatted table.
4. Hover the table (or click **Copy SQL**) to see the exact query that ran.
5. Past questions stack up in a collapsible **history** panel — click any entry
   to instantly re-display its result without re-running it.

Themes (light/dark) and query history make it comfortable for repeated,
exploratory use.

---

## Command-line interface

For scripting and quick checks, the same pipeline is exposed as a CLI:

```
ask "the 10 most recent orders with their total"

# generate the SQL without running it
ask "revenue by region last fiscal year" --no-exec

# cap the number of rows returned
ask "all active customers in the West territory" --limit 50

# target the production database instead of the local dev snapshot
ask "open orders over $10k" --production
```

---

## Example interactions

Each example shows a natural-language question and the *kind* of query the
system produces. Table and column names below are generic placeholders.

### "Top 10 customers by revenue last fiscal year"

The system:

- routes to the customer + order-line tables,
- injects the exact fiscal-year date range (computed in Python),
- applies the "active customers only" mandatory filter automatically,
- aggregates revenue as the summed line subtotal (not price × quantity).

```sql
SELECT c.company_name, SUM(CAST(l.line_subtotal AS REAL)) AS revenue
FROM order_line l
JOIN order_header o ON l.order_id = o.id
JOIN customer c     ON o.customer_id = c.id
WHERE c.active = 1
  AND o.ship_date >= '2024-12-01' AND o.ship_date < '2025-12-01'
GROUP BY c.company_name
ORDER BY revenue DESC
LIMIT 10
```

### "How many open orders does each sales rep have?"

The system recognizes "open orders" as a pre-built report snapshot and queries
it directly — no joins, no active filter re-applied (it's already baked in):

```sql
SELECT sales_rep, COUNT(*) AS open_orders
FROM open_orders_report
GROUP BY sales_rep
ORDER BY open_orders DESC
```

### "List products in [ X ] category, excluding fees and adjustments"

The system maps the category word to its internal code, and applies the
non-product exclusion so metadata rows (fees, credits, service charges) never
appear in a product list:

```sql
SELECT product_code, product_name
FROM item_master
WHERE product_type = 'X'
  AND product_code NOT IN ('SERVICE FEE', 'SHIPPING', 'CREDIT ADJUSTMENT', ...)
LIMIT 100
```

---

## What gets rejected

The read-only guardrails reject anything that isn't a single, safe `SELECT`.
These never reach the database:

| Attempt                            | Result                                                   |
| ---------------------------------- | -------------------------------------------------------- |
| `DELETE FROM customer WHERE ...` | Rejected — forbidden write keyword                      |
| `SELECT ...; DROP TABLE ...`     | Rejected — multiple statements                          |
| `EXEC some_procedure`            | Rejected — not a`SELECT`                              |
| A valid`SELECT` with no row cap  | Accepted — a`LIMIT`/`TOP` is injected automatically |

---

## Running the model locally

The language model is served on-premise through a`llama.cpp` server that
exposes an OpenAI-compatible endpoint. The app points at that endpoint, so
no request ever leaves the machine. Swapping in a hosted model (for a quality
comparison) is a one-line configuration change — the rest of the pipeline is
unchanged.
