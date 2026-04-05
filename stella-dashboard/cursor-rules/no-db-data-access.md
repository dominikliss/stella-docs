# Cursor rule export: `no-db-data-access.mdc`

_Source of agent rule in theme repo: `.cursor/rules/no-db-data-access.mdc`._

---

# No Production Data Access

**Never query, read, display, or log actual data from the database.** The database contains sensitive business data (clients, invoices, transactions, emails, contacts, financial figures) that must not be passed to third-party services.

## Forbidden

- `SELECT` / `get_posts` / `get_post_meta` / `get_field` / `$wpdb->get_results` that **return row content** (names, notes, amounts, emails, addresses, etc.)
- Dumping REST API responses that contain real records
- Reading `dls_email`, `dls_mailbox`, `dls_pm_*`, or any custom table rows
- Printing or logging real field values from any post type (transaction, invoice, client, person, file, product, etc.)

## Allowed

- **Schema / structure queries:** `DESCRIBE`, `SHOW TABLES`, `SHOW COLUMNS`, `SHOW CREATE TABLE`, checking table existence, column types, indexes
- **Count queries:** `SELECT COUNT(*)` (no content)
- **DDL / migrations:** `CREATE TABLE`, `ALTER TABLE`, `dbDelta`, adding indexes
- **Code that reads data at runtime** (PHP services, REST callbacks) — writing the code is fine; executing it to inspect real values is not

## If you need to verify data-dependent behaviour

- Ask the user to describe the data or provide a sanitised example
- Use placeholder / mock values in test scripts
- Check code logic and SQL structure, not live output
