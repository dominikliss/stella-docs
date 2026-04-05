# Nachrichten (IMAP): WordPress-Cron & Hosting-Panel

The theme registers a WP-Cron event **`dls_mail_imap_sync_cron`** on interval **`dls_every_five_minutes`** (300 seconds). Each run imports up to **50** messages per **active** mailbox (filter: `dls_mail_cron_sync_limit`).

## 1. Trigger WordPress cron from the hosting panel

WordPress only runs scheduled tasks when something hits **`wp-cron.php`**. On many hosts, **page visits** alone are unreliable, so add a **real cron job** in the panel (every **1–5 minutes**).

Replace `https://YOUR-DOMAIN.example/wp-cron.php` with your site URL (same domain as WordPress).

### wget

```text
*/5 * * * * wget -q -O - "https://YOUR-DOMAIN.example/wp-cron.php?doing_wp_cron" >/dev/null 2>&1
```

### curl

```text
*/5 * * * * curl -s "https://YOUR-DOMAIN.example/wp-cron.php?doing_wp_cron" >/dev/null 2>&1
```

### PHP-CLI (if the panel offers “run PHP” with a path)

```text
*/5 * * * * /usr/bin/php -q /path/to/your/site/wp-cron.php >/dev/null 2>&1
```

Use the **absolute path** to `wp-cron.php` in your document root (or WordPress root) as shown by the host.

## 2. Optional: disable pseudo-cron on page load

In **`wp-config.php`**, **above** the line `/* That's all, stop editing! */`, add:

```php
define( 'DISABLE_WP_CRON', true );
```

Then **only** the panel cron (or manual requests) will trigger scheduled tasks. This avoids extra work on every visitor request.

## 3. First-time registration

After deploying the theme, load the site once (front or admin) or save **Settings → Permalinks** so `init` runs and the event is scheduled. You can also open `wp-cron.php` once in the browser to fire the queue.

## 4. Verify

- Plugin **WP Crontrol** (optional): look for **`dls_mail_imap_sync_cron`** and interval **Every 5 minutes (DLS IMAP)**.
- Or check `wp_options` for `cron` (advanced).

## 5. Changing the interval later

If you already had a different schedule registered, clear and reschedule once (e.g. WP-CLI):  
`wp cron event delete dls_mail_imap_sync_cron` then visit the site again, or use WP Crontrol to delete/reschedule.
