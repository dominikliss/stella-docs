# osgar.datahub — Client IP Access

**Status:** Implemented 2026-09-03. Temporary access grant — see "Removal" below.

**Purpose:** A client needs to open `osgar.datahub.foxcraft.digital` in a browser, without gaining access to SSH (`2201`), any other Caddy-routed subdomain, or any other Stella service.

**Related:** [`infrastructure.md`](infrastructure.md), [`dev-ssh-access.md`](dev-ssh-access.md)

---

## Client IPs currently allowed (osgar.datahub only)

| IP | Added |
|---|---|
| `213.47.151.242` | 2026-09-03 |
| `89.67.29.69` | 2026-09-03 |

⚠️ **These are temporary and should be removed once the client no longer needs access.** Removal = delete the IP from both places below, then `docker compose restart caddy` (and reapply the firewall script if the IP is being fully retired, not just re-scoped).

---

## Why two layers were needed

Port 443 is Docker-published, so it's filtered by `DOCKER-USER` (iptables), not UFW — and that filter is **port-wide**, not per-domain. Before this change, only the two static team IPs (`194.126.177.181`, `23.88.90.12`) could reach port 443 at all; Caddy itself did no IP filtering per site.

Simply adding the client IP to `DOCKER-USER` would let it reach **every** Caddy-routed site on the server (`stella.foxcraft.digital`, `advoapp.finditoo`, `stella-deployment-api`, `stella-health-api`), not just `osgar.datahub`. So the fix needed two layers:

1. **`DOCKER-USER`** — client IP allowed through port 443 at all (prerequisite, applies port-wide).
2. **Caddy, per-site `remote_ip` matcher** — every *other* site explicitly blocks the client IP; `osgar.datahub` explicitly allows it. This is what actually scopes access to one subdomain.

---

## 1. Firewall — `/usr/local/bin/docker-user-firewall.sh`

Client IPs added as ACCEPT rules in the port-443 block, alongside the two static team IPs, before the final DROP:

```bash
iptables -A DOCKER-USER -i $WAN_IF -s 194.126.177.181 -p tcp --dport 443 -j ACCEPT
iptables -A DOCKER-USER -i $WAN_IF -s 23.88.90.12 -p tcp --dport 443 -j ACCEPT
iptables -A DOCKER-USER -i $WAN_IF -s 213.47.151.242 -p tcp --dport 443 -j ACCEPT
iptables -A DOCKER-USER -i $WAN_IF -s 89.67.29.69 -p tcp --dport 443 -j ACCEPT
iptables -A DOCKER-USER -i $WAN_IF -p tcp --dport 443 -j DROP
```

No entry was added for `2201` (osgar-datahub-ssh) or `2202` (advoapp-ssh) — SSH access remains restricted to the two static team IPs only.

Script is systemd-applied on boot; reapply manually after edits with:

```bash
sudo /usr/local/bin/docker-user-firewall.sh
```

Verify order (ACCEPT before DROP) with:

```bash
sudo iptables -L DOCKER-USER -n -v --line-numbers | grep 443
```

---

## 2. Caddy — `/opt/services/caddy/Caddyfile`

Every site block **except** `osgar.datahub` gets an explicit deny-list matcher for the client IPs:

```caddyfile
@blocked remote_ip 213.47.151.242 89.67.29.69
respond @blocked 403
```

`osgar.datahub.foxcraft.digital` gets an explicit allow-list matcher instead (static team IPs + client IPs; everyone else 403s):

```caddyfile
@blocked not remote_ip 194.126.177.181 23.88.90.12 213.47.151.242 89.67.29.69
respond @blocked 403
```

Apply with:

```bash
cd /opt/services
docker compose restart caddy
```

---

## Verification

From the client's own connection (not from Stella or a whitelisted team IP — those succeed everywhere regardless):

```bash
curl -s -o /dev/null -w "%{http_code}\n" https://osgar.datahub.foxcraft.digital
# expect 200

curl -s -o /dev/null -w "%{http_code}\n" https://advoapp.finditoo.foxcraft.digital
curl -s -o /dev/null -w "%{http_code}\n" https://stella-health-api.foxcraft.digital
# expect 403 on both
```

---

## Removal

When the client no longer needs access:

1. Remove the two `iptables -A ... -j ACCEPT` lines for the client IP(s) from `docker-user-firewall.sh`, then `sudo /usr/local/bin/docker-user-firewall.sh`.
2. Remove the client IP(s) from every `@blocked remote_ip ...` / `@blocked not remote_ip ...` line in the Caddyfile, then `docker compose restart caddy`.
3. Update or delete this file.

---

## Pattern for future per-site client grants

This same two-layer approach applies any time a new external IP needs access to exactly one subdomain:

1. Add the IP to the port-443 `ACCEPT` rules in `docker-user-firewall.sh` before the DROP, then reapply the script.
2. Add the IP to the deny-list (`@blocked remote_ip ...`) on every site block except the target, and add it to the allow-list (`@blocked not remote_ip ...`) on the target.
3. `docker compose restart caddy`.
4. Document the IP and expiry in the table at the top of this file.
