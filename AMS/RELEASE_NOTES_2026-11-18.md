# AMS — release notes, 2026-11-18

Build: `AMS_app_2026-11-18.zip` · 517 files · 294 Python · 139 templates
Supersedes every earlier archive. No database is included — this is source only.

## How to run the regression suite (60 seconds)

```bash
cd AMS/APPLICATION_AUDIT/STRESS_TEST
python3 run_all_tests.py --app /path/to/AMS
```

Exit code 0 means every known defect is still fixed. `--quick` skips the volume
section (~10 s). `NEXT_TESTING_PROMPT.md` in the same folder is the brief for the
next testing round.

## Fixes in this build

| # | File | What changed |
|---|---|---|
| 1 | `app/services/void_rebuild.py` | **Deleting a payment now reopens the bills it had settled.** Before, the customer's debt vanished with the payment: a 5,000 sale whose 5,000 receipt was deleted stayed at 0.00 owed. The reopened amount is recomputed from the source document, so an overpayment cannot fabricate a debt. |
| 2 | `app/blueprints/misc/extra.py`, `templates/system_report.html` | **"Auto-Fix Issues" no longer writes.** It used to re-bill money already collected — one click on a fully paid 1,000 sale made it 1,000 owed again, and every click added another copy of the pending-bill table. The route is now read-only diagnosis and the button is gone. |
| 3 | `app/services/schema.py` | **Four expression indexes** on `lower(trim(coalesce(client_name,'')))`. The receivables and ledger queries could not use a plain index, so every settlement pass scanned the whole table. `SCAN pending_bill` → `SEARCH … USING INDEX`, 0.80 ms → 0.11 ms. |
| 4 | `app/blueprints/reports/_profit_profit_reports.py` | **An empty P&L period now says so** instead of showing a page of zeros that reads like a broken report. |

## Retraction

The audit's headline finding — "customer payments never reduce receivables, a
24.6 M error" — was **wrong and is withdrawn**. It compared *money still owed*
against *all money ever received*, which measure different things; 2,478 bills had
already been settled to zero. The identity that has to hold does hold, to the rupee:

```
sold + booked      81,038,521
money received     55,538,279
still owed         25,500,242
received + owed    81,038,521   → difference 0.00
```

The wrong figure came from reading uncommitted state through the session under test.
Every assertion in the new suite reads through a separate read-only connection.

## Delete behaviour — what to expect

| delete this | what is removed | what stays |
|---|---|---|
| sale / booking | its lines, its stock movement, its own bill, the amount paid at the counter | money paid *separately* against it |
| a payment | that cash, its ledger row, and it **reopens** the bill it had settled | the sale |
| an advance | that cash | — |

Deleting a document never deletes money that genuinely changed hands. If a customer
paid against a booking and the booking is deleted, the payment remains visible as a
client credit (green balance). Destroying it would make the ledger disagree with the
cash drawer.

## Still open (not fixed in this build)

1. **`get_next_bill_no()` lock ordering** — takes the write lock before the conflict
   check, which can trigger a repair path that writes and rolls back beneath it.
   `utils/audit.py`'s second connection only sets `busy_timeout=1500`. Recorded, not
   applied: reordering trades away the bill-number uniqueness guarantee.
2. **Quadratic rebuild on payment save** — every receipt rebuilds the client's whole
   history (10,417 statements / 12.4 s on a large client). The indexes made it cheaper;
   the design is still quadratic. The real fix is a targeted allocation of one payment.
3. **97 float money columns** — not all mirrored to integer minor units.
4. **`journal_mode=DELETE`** on the production path.

Details: `APPLICATION_AUDIT/STRESS_TEST/PROPOSED_CHANGES.md`.
