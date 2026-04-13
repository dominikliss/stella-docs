# Cursor rule export: `acf-field-map.mdc`

_Source of agent rule in theme repo: `.cursor/rules/acf-field-map.mdc`._

---

# ACF Field Map

All fields are accessed via the REST API as `record.acf.<field_name>`.
All entities are fetched with `acf_format: 'standard'` — use `record.acf.*` paths, never `record.transactionFields.*` or other old GraphQL paths.

## transaction
ACF text fields return **strings** — use `parseFloat()` before arithmetic.
ACF `date_picker` fields return **`Y-m-d`** strings (e.g. `"2026-03-07"`) — compare directly with `>=` / `<=`, no Date parsing needed.

**Preferred date for accounting/display:** `invoice_date` (falls back to `transaction_date`). Use `getAccountingDate(t)` from `helpers.js` — never read `transaction_date` directly for filtering or chart grouping.

| Field name                             | ACF type     | Return value / notes                                      |
|----------------------------------------|--------------|-----------------------------------------------------------|
| `type`                                 | radio        | `"Einnahme"` or `"Ausgabe"`                               |
| `note`                                 | text         | string                                                    |
| `invoice_date`                         | date_picker  | `"Y-m-d"` string — **primary date** for accounting       |
| `transaction_date`                     | date_picker  | `"Y-m-d"` string — payment date                          |
| `price_net`                            | text         | string — `parseFloat(t.acf.price_net)`                    |
| `currency`                             | radio/select | `"EUR"`, `"PLN"`, **`USD`** (Einnahme und Ausgabe; bei nicht-USD werden EUR-Umrechnungsfelder auf Save geleert) |
| `pln_conversion_rate_invoice_date`     | text         | **EUR/PLN:** NBP mid PLN per 1 EUR. **USD:** PLN per 1 USD. Auto via `dls/v1/nbp-rate` or `dls/v1/nbp-expense-rates` / KSeF import |
| `pln_conversion_rate_transaction_date`   | text         | Same semantics for Zahlungsdatum |
| `eur_conversion_rate_invoice_date`       | text         | **USD only:** multiply USD net by this for EUR (NBP USD/PLN ÷ EUR/PLN). Auto via `dls/v1/nbp-expense-rates` |
| `eur_conversion_rate_transaction_date`   | text         | Same for payment date |
| `vat_percent`                          | text         | string                                                    |
| `vat_free`                             | radio        | `"nein"`, `"Reverse Charge"`, `"Innengemeinschaftlicher Erwerb"`, `"Drittland"` |
| `expense_category`                     | radio        | `"N/A"`, `"Werbung"`, `"Material"`, `"Beratung"`, `"GWG"`, `"Investition"`, `"Erlöskorrektur"` |
| `internal_category`                    | radio        | `"N/A"`, `"Investition"`                                 |
| `ksef_reference`                       | text         | KSeF invoice number — set on import, used for deduplication. Field key: `field_6829ksef000r1` |
| `ksef_seller_nip`                      | text         | Seller NIP from KSeF XML. Field key: `field_6829ksef000n2` |

Note: `invoice_url` and `invoice_url_pl` are stored via import (post meta) — used in open-invoices. Not in ACF field group; may require custom REST exposure.

## invoice
ACF `date_picker` fields return `"Y-m-d"` strings. Positions repeater items use `product_id` (post ID string), `quantity` (number), `unit_price` (number).

| Field name                 | ACF type    | Return value / notes                          |
|----------------------------|-------------|------------------------------------------------|
| `number`                   | text        | string — invoice number                       |
| `client_id`                | select      | client post ID as string                       |
| `transaction_id`           | select      | transaction post ID as string (allow_null: 1)  |
| `service_start_date`       | date_picker | `"Y-m-d"` string                              |
| `service_end_date`         | date_picker | `"Y-m-d"` string                              |
| `language_code`            | select      | `"DE"`, `"EN"`, `"PL"`                         |
| `bank_details_country_code`| select      | `"AT"`, `"PL"`, `"USA"` — **legacy, no longer used for bank account selection**; bank account is now selected via `bank_account_id` post meta or `BankAccountDbService::find_account()` |
| `invoice_url`              | url         | string — PDF URL (from import or manual)      |
| `positions`                | repeater    | array of `{ product_id, description, quantity, unit_price, vat_percent }` |
| `ksef_invoice_number`      | text        | KSeF number assigned after successful send. Field key: `field_6961inv_ksef_number` |
| `ksef_send_status`         | select      | `""` / `"pending"` / `"sent"` / `"error"`. Field key: `field_6961inv_ksef_status` |
| `ksef_sent_at`             | text        | ISO 8601 timestamp of last send attempt. Field key: `field_6961inv_ksef_sent_at` |
| `ksef_error_message`       | text        | Last error from KSeF API. Field key: `field_6961inv_ksef_error` |
| `ksef_reference_number`    | text        | JSON string `{"session":"…","invoice":"…"}` for status polling. Field key: `field_6961inv_ksef_ref` |

### WP post meta on `invoice` (not ACF)

| Meta key         | Type    | Notes |
|------------------|---------|-------|
| `bank_account_id`| integer | ID of the pinned `dls_bank_account` row; `0` = no pin. Registered via `register_post_meta` with `show_in_rest: true`. Saved via direct `apiFetch POST wp/v2/invoice/{id}` — do **not** rely on `saveEntityRecord` for this field. Takes priority over transaction pin and auto-select. |
| `_ksef_sent_env` | string  | `"production"` or `"test"` — environment of the last successful KSeF send. Allows re-sending to a different environment without HTTP 409. |

## client
| Field name             | ACF type     | Return value / notes                          |
|------------------------|--------------|-----------------------------------------------|
| `official_name`        | text         | string                                        |
| `status`               | radio        | `"active"`, `"lead"`, `"inactive"`             |
| `registration_number`  | text         | string                                        |
| `vat_number`           | text         | string                                        |
| `tracking_time_pro_id` | number       | number                                        |
| `website`              | url          | string                                        |
| `phone`                | text         | string                                        |
| `email`                | email        | string                                        |
| `address`              | text         | string                                        |
| `zip`                  | text         | string                                        |
| `city`                 | text         | string                                        |
| `country`              | text         | string                                        |
| `people`               | select (multiple) | array of person post IDs as strings       |
| `socials`              | repeater     | array of `{ social: url }`                    |
| `invoice_emails`       | repeater     | array of `{ email: string }`                  |
| `logo`                 | image        | array (`url`, `width`, `height`, …)           |

## person
| Field name      | ACF type     | Return value / notes  |
|-----------------|--------------|-----------------------|
| `name`          | text         | string                |
| `phone`         | text         | string                |
| `email`         | email        | string                |
| `birthday`      | date_picker  | `"Y-m-d"` string     |
| `profile_image` | image        | array                 |

## file
In **`file-form.js`**, the **Dokumentart** (`document_type`) control is shown **only when `type === "document"`**. On save, if `type` is not `document`, send **`document_type: "normal"`** so AGB vs normal only applies to real documents.

| Field name                   | ACF type     | Return value / notes                                 |
|-----------------------------|--------------|------------------------------------------------------|
| `type`                      | radio        | `"document"`, `"proposal"`, `"contract"`             |
| `document_type`             | radio        | `"normal"`, `"terms_of_service"` (AGB / Nutzungsbedingungen) |
| `contract_type`             | radio        | `"normal"`, `"wp_maintenance"` (when type=contract)  |
| `client_id`                 | select       | client post ID as string                             |
| `public_url`                | url          | string                                               |
| `accepted_date`             | date_picker  | `"d.m.Y"` string (different format from transaction!) |
| `accepted_method`           | radio        | `"na"`, `"customer_area"`, `"email"`                 |
| `accepted_ip_address`       | text         | string                                               |
| `contract_start_date`       | date_picker  | `"d.m.Y"` string                                    |
| `maintenance_price_override`| text         | string                                               |
| `hourly_rate_override`      | text         | string                                               |
| `client_websites_override`  | text         | string                                               |
| `current_hosting_provider`   | text         | string                                               |

## product
ACF `number` returns number. `base_price` can be used directly for arithmetic.

| Field name               | ACF type     | Return value / notes                    |
|--------------------------|--------------|-----------------------------------------|
| `name`                   | text         | string                                  |
| `description`            | textarea     | string                                  |
| `base_price`             | number       | number — use directly for arithmetic    |
| `is_recurring`           | true_false   | boolean                                 |
| `subscription_period`    | radio        | `"monthly"`, `"quarterly"`, `"yearly"`  |
| `subscription_start_date`| date_picker  | `"Y-m-d"` string                        |

## accountingconfig
All numeric fields are ACF `text` type — use `parseFloat()` before arithmetic.

| Field name                   | ACF type   |
|------------------------------|------------|
| `year`                       | text       |
| `is_completed`               | true_false |
| `open_costs`                 | text       |
| `vacation_budget`            | text       |
| `vacation`                   | text       |
| `vacation_used`              | text       |
| `sick_days`                  | text       |
| `sick_days_used`             | text       |
| `monthly_investment_budget`  | text       |
| `expenses_budget`            | text       |
| `existing_emergency_fund`    | text       |
| `emergency_fund`             | text       |
| `advance_payments`           | text       |
| `income_tax_paid`            | text       |
| `open_sv_contribution`      | text       |
| `vat_1_quarter_paid`         | true_false |
| `vat_2_quarter_paid`         | true_false |
| `vat_3_quarter_paid`         | true_false |
| `vat_4_quarter_paid`         | true_false |
