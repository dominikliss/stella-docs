# Cursor rule export: `php-conventions.mdc`

_Source of agent rule in theme repo: `.cursor/rules/php-conventions.mdc`._

---

# PHP Conventions

## Namespaces
- `DLS\Services\` — all services (FileService, ClientService, OptionService, BasicService, NbpExchangeRateService, EmailCrudService, EmailSpamService, EmailCategoryDbService, InvoicePdfService, CommissionReportService, …)
- Route callback functions use no namespace; prefix with `dls_` (e.g. `dls_get_nbp_rate`)

## Registering New Files

**Services** (`inc/services/`): Autoloaded via Composer classmap. After adding a new service class, run `composer dump-autoload` to update the classmap. No `require` in `functions.php` needed.

**Routes, install scripts, shortcodes, post types**: Add `require` calls in `functions.php`. Order:
1. Services load automatically via Composer autoload
2. Routes: add `require` in the routes block of `functions.php`
3. Follow the existing `require` block groupings

## REST Route Template (browser-facing)
```php
// inc/routes/your-route.php
add_action( 'rest_api_init', function () {
    register_rest_route( 'dls/v1', '/your-endpoint', [
        'methods'             => 'GET',
        'callback'            => 'dls_your_callback',
        'permission_callback' => function () { return is_user_logged_in(); },
        'args'                => [
            'param' => [
                'required'          => true,
                'sanitize_callback' => 'sanitize_text_field',
            ],
        ],
    ] );
} );

function dls_your_callback( WP_REST_Request $request ) {
    // ...
    return new WP_REST_Response( [ 'key' => $value ], 200 );
    // or:
    return new WP_Error( 'error_code', 'Message', [ 'status' => 404 ] );
}
```

Use `dls/v1` for browser-facing routes. The `api/v1` namespace is IP-restricted — only for the external app.

## Application Constants

`inc/constants.php` defines app-wide constants loaded early in `functions.php`:
- `DLS_ALLOWED_IPS` — IP whitelist for `api/v1` routes
- `DLS_OWNER_EMAILS` — owner email addresses for client-matching exclusions
- `DLS_OWNER_DOMAINS` — owner domains for client-matching exclusions
- `DLS_MAIL_CLASSIFICATION_ENABLED` — when `true`, `MailClassificationService` may run (main INBOX inbound only); default `false`
- `DLS_ANALYSIS_TIMEOUT` — max runtime for async email analysis jobs

Use these constants instead of hardcoding values.

## REST Permission Callbacks

For browser-facing routes that only require authentication, use the shared callback:

```php
'permission_callback' => 'dls_require_login',
```

Defined in `inc/routes/functions.php`. Do not create inline `function () { return is_user_logged_in(); }` closures.

For named feature-specific permission callbacks (e.g. `dls_mail_permission`, `dls_pm_permission`), keep those as-is.

## Transient Caching
Cache expensive external API calls (e.g. NBP exchange rates) with WordPress transients.

```php
$key    = 'my_cache_' . $identifier;
$cached = get_transient( $key );
if ( $cached !== false ) {
    return (float) $cached;
}
// ... fetch ...
set_transient( $key, $value, WEEK_IN_SECONDS );
```

## Service Class Pattern
```php
namespace DLS\Services;

class MyService {
    public function do_something(): ?string {
        // ...
    }
}
```

Do **not** self-instantiate at the bottom of service files (`new MyService();`). Services are instantiated in `functions.php` or on demand inside route callbacks.

## ACF Data
- ACF field values saved via REST API must go inside the `acf` key of the request body
- Dates stored by ACF date_picker internally as `Ymd` in the database, but returned via REST per the field's `return_format` setting
- NBP rate fields (`pln_conversion_rate_invoice_date`, `pln_conversion_rate_transaction_date`) on `transaction` post type are auto-populated by `NbpExchangeRateService` via the `dls/v1/nbp-rate` endpoint or `NbpExchangeRateService::get_rate_for_input_date($date_string)` in PHP

## ACF Select Fields — Scalar vs Array Storage
ACF `select` fields with `"ui": 1` in the JSON config save as **serialized arrays**, not scalars.
For `client_id`, `transaction_id`, `product_id` and any single-value select that stores a post ID:
- Set `"ui": 0` in the ACF JSON config file (`inc/acf/settings/group_*.json`)
- Retrieve with `get_post_meta( $post_id, 'field_name', true )` — **not** `get_field()` which returns the post object
- `intval( get_field('field_name') )` on a `"ui": 1` field returns `1` (wrong) — always check `"ui"` setting first

## Writing ACF Fields in PHP
Use `update_field( $field_key, $value, $post_id )` with the field key from the JSON config (e.g. `field_6829ksef000r1`). This stores both the value and the `_fieldname → field_key` reference meta that ACF admin UI requires.

```php
// ✅ Correct — ACF recognises the value in the admin UI
update_field( 'field_6829ksef000r1', $ksef_ref, $post_id );

// ❌ Avoid for ACF-registered fields — stores value without key reference
update_post_meta( $post_id, 'ksef_reference', $ksef_ref );
```

Use raw `update_post_meta()` only for meta keys that are **not** registered as ACF fields (e.g. `old_transaction_id`).

## XML Parsing (KSeF / External APIs)
Use `DOMDocument` + `DOMXPath` for all XML parsing. Always detect the namespace dynamically from the root element — never hardcode namespace URIs (schema versions change between years).

```php
$dom = new \DOMDocument();
if ( ! @$dom->loadXML( $xml ) ) {
    throw new \Exception( 'Failed to parse XML.' );
}

$xpath  = new \DOMXPath( $dom );
$ns     = $dom->documentElement->namespaceURI ?? '';
if ( empty( $ns ) ) {
    throw new \Exception( 'Could not detect XML namespace.' );
}
$xpath->registerNamespace( 'fa', $ns );

$get = fn( string $expr ) => $xpath->evaluate( 'string(' . $expr . ')' );

$value = $get( '//fa:Root/fa:Child/fa:Field' );
```

## Debug Logging
For external API integrations, write a debug log file alongside the import. Use `FILE_APPEND` so each run appends.

```php
$debug_file = get_stylesheet_directory() . '/ksef-debug.txt';
$log = fn( string $msg ) => file_put_contents(
    $debug_file,
    '[' . date( 'H:i:s' ) . '] ' . $msg . "\n",
    FILE_APPEND
);

file_put_contents( $debug_file, "\n=== Import " . date( 'Y-m-d H:i:s' ) . " ===\n" );
$log( 'Starting...' );
```

Remove or gate debug files before production deployment.

## Upsert Pattern (Create-or-Update)
When importing external data, separate create and update logic to avoid overwriting user edits:

```php
if ( ! empty( $existing ) ) {
    // Update: only refresh fields that come from the external source.
    // Never overwrite user-edited fields (note, post_title, categories).
    update_post_meta( $post_id, 'price_net', $data['price_net'] );
    update_field( 'field_6829ksef000r1', $ksef_ref, $post_id );
} else {
    // Create: write all fields including user-facing ones.
    $post_id = wp_insert_post( [ 'post_type' => 'transaction', ... ] );
    update_post_meta( $post_id, 'note', $note );
    update_post_meta( $post_id, 'price_net', $data['price_net'] );
    update_field( 'field_6829ksef000r1', $ksef_ref, $post_id );
}
```
