# Cursor rule export: `saldeo-api.mdc`

_Source of agent rule in theme repo: `.cursor/rules/saldeo-api.mdc`._

---

# SaldeoSMART API Reference (v5.0.1 / API XML 3.0)

## Base URLs
- **Production**: `https://saldeo.brainshare.pl`
- **Test**: `https://saldeo-test.brainshare.pl`
- Minimum API version from 2026: **2.0** (use 3.0 for new features)

## Credentials (stored as WordPress options or constants)
| Credential | Where to get it |
|---|---|
| `username` | SaldeoSMART login |
| `api_token` | SaldeoSMART → Ustawienia konta (never sent in requests) |
| `company_program_id` | Arbitrary unique string set by integrator, identifies the company |

## Authentication — `req_sig` Generation

Every request must include `username`, `req_id` (unique per user, max 255 chars), and `req_sig`.

```
req_sig = HEX( MD5( URL_ENCODING( sorted_params_string ) + api_token ) )
```

Steps:
1. Collect all request params (exclude `req_sig` itself; no empty or duplicate params)
2. Sort params **alphabetically by key**
3. Concatenate as `key=value` pairs (no separator between pairs)
4. URL-encode the result using the custom encoding below
5. Append `api_token` (raw, not encoded)
6. Compute MD5, output as uppercase hex

**Custom URL encoding** (differs from RFC-3986):
- Space → `+` (not `%20`)
- `*` → `*` (not `%2A`)
- `~` → `%7E` (not `~`)
- Hex is **UPPERCASE**, UTF-8 encoded (e.g. `Ł` → `%C5%81`)
- PHP equivalent: `urlencode()` is close but verify case

Example:
```
username=user, api_token=token, req_id=request-id
→ URL_ENCODE("req_id=request-idusername=user") + "token"
→ MD5 → req_sig
```

## Request Format (POST)

All POST bodies are XML → gzip → base64, sent as the `command` HTTP parameter.

```php
$xml = '<?xml version="1.0" encoding="UTF-8"?><ROOT>...</ROOT>';
$compressed = gzencode($xml);
$command = base64_encode($compressed);
// POST: username=...&req_id=...&req_sig=...&command=<encoded>
```

GET requests pass all params as query string (no `command`).

## Response Format

XML UTF-8, optionally gzip-compressed (send `Accept-Encoding: gzip` header).

```xml
<RESPONSE>
  <STATUS>OK</STATUS>  <!-- or ERROR -->
  <ERROR_CODE>...</ERROR_CODE>
  <ERROR_MESSAGE>...</ERROR_MESSAGE>
</RESPONSE>
```

## Rate Limits
- Max **20 requests/minute** per user
- Max **1 concurrent request** — wait for response before next request
- Max POST size: **100MB** (~70MB effective for files due to base64 overhead)

---

## Key Operations for Invoice Integration

### SSK06 — Add Invoice (issue in SaldeoSMART)
```
POST /api/xml/3.0/invoice/add
Params: company_program_id, command (XML)
```
Requires API ≥ 3.0. The company must have invoice issuance enabled in settings.
After successful call, wait ~30 seconds before fetching the PDF (generation is async).
Returns `INVOICE_ID`.

Invoice numbering:
- Automatic: omit `NUMBER`, optionally send `SUFFIX`
- Manual: send `NUMBER` explicitly

Date of sale: single `SALE_DATE` or range `SALE_DATE_FROM` / `SALE_DATE_TO` (start must not be >60 days before end).

XSD: `https://saldeo.brainshare.pl/static/doc/api/3_0/invoice/invoice_add_request.xsd`

### SSK07 — List Invoice IDs
```
POST /api/xml/3.0/invoice/getidlist
Params: company_program_id, command (XML with MONTH + YEAR)
```
Returns IDs grouped by type: `INVOICES`, `CORRECTIVE_INVOICES`, `PRE_INVOICES`, `CORRECTIVE_PRE_INVOICES`.

### SSK08 — List Invoices by ID
```
POST /api/xml/3.0/invoice/listbyid
Params: company_program_id, command (XML — same structure as SSK07 response)
Max: 300 IDs per request
```
Returns full invoice data including PDF URL (`SOURCE`).

### SSK05 — Search Invoices (existing invoices, read-only)
```
GET /api/xml/1.8/invoice/list?company_program_id=...&policy=SALDEO
```
Returns invoices marked for export in SaldeoSMART.

### SS24 — Import Document (upload PDF with metadata)
```
POST /api/xml/3.0/document/import
Params: company_program_id, command (XML metadata), attmnt_[ID] (file, base64+gzip)
Max: 50 files, 70MB
```
Returns `DOCUMENT_ID` per file (`VALID` or `NOT_VALID` + error message).
After import, fetch document via SS23.

### SS05 — Add Document (upload for OCR)
```
POST /api/xml/1.0/document/add
Params: company_program_id, attmnt_[ID]
```
File attachment naming: `attmnt_` prefix + alphanumeric ID matching `[a-zA-Z0-9]{1,255}`.

---

## PHP Implementation Pattern

```php
function saldeo_request(string $operation, string $method, array $params, ?string $xml = null): array {
    $username  = get_option('saldeo_username');
    $api_token = get_option('saldeo_api_token');
    $base_url  = 'https://saldeo.brainshare.pl';

    $req_id = date('YmdHis') . rand(1000, 9999);

    $all_params = array_merge($params, [
        'username' => $username,
        'req_id'   => $req_id,
    ]);
    if ($xml !== null) {
        $all_params['command'] = base64_encode(gzencode($xml));
    }

    ksort($all_params); // sort alphabetically by key

    $base_string = '';
    foreach ($all_params as $k => $v) {
        $base_string .= $k . '=' . $v;
    }

    $req_sig = strtolower(md5(urlencode($base_string) . $api_token));
    $all_params['req_sig'] = $req_sig;

    if ($method === 'GET') {
        $url = $base_url . $operation . '?' . http_build_query($all_params);
        $response = wp_remote_get($url, ['headers' => ['Accept-Encoding' => 'gzip']]);
    } else {
        $url = $base_url . $operation;
        $response = wp_remote_post($url, [
            'body'    => $all_params,
            'headers' => ['Accept-Encoding' => 'gzip'],
            'timeout' => 30,
        ]);
    }

    $body = wp_remote_retrieve_body($response);
    return simplexml_load_string($body, 'SimpleXMLElement', LIBXML_NOCDATA);
}
```

> **Note**: PHP's `urlencode()` matches the SaldeoSMART custom URL encoding in most cases. Verify with example from the docs (username=user, api_token=token, req_id=request-id → `req_sig=d73710fdff6acc96361f5b9cb3425cee`).

---

## WordPress Integration Notes

- Store `saldeo_username`, `saldeo_api_token`, `saldeo_company_program_id` as WordPress options (Settings page or `wp_options`)
- Create route file `inc/routes/saldeo.php` using `dls/v1` namespace for browser-triggered actions
- Keep `api_token` server-side only — never expose to JS
- New route files must be `require`d in `functions.php`

## Error Codes
| Code | Meaning |
|---|---|
| 4302 | User does not exist or is blocked |
| 4303 | User has no active API |
| 4304 | req_id already used |
| 4307 | Quota exceeded (20 req/min) |
| 4308 | Too many concurrent requests |
| 4401 | XML structure error |
| 4402 | XML data type error |
| 6001 | Access denied |
| 6101 | Validation error |
