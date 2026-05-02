---
name: sap-management-cockpit
description: SAP Management Cockpit skill — the single-pane-of-glass landing page for the full Fivetran SAP estate. Use for: editing/extending the Management Cockpit page (`SAP_Management_Cockpit.html`), adding or removing a server card, wiring a new server into the reachability API, troubleshooting reachability false-reds, modifying the Service Monitor health check cron (`/usr/local/bin/sap_health_check.sh` on sapidesecc8), changing the JSON status payload (`health_status.json`), updating mailing lists (smtp2go-backed `SAPSpecialists` distribution), redeploying the static HTML, restarting the `gcs-explorer` service after backend changes, and any task that touches the cockpit's UI, reachability endpoints, monitoring agent, or email relay. NOT for individual cockpit pages (S/4HANA, ECC, SQ1, SAPRouter, HVR, SSH Tunnel) — those have their own skills.
model: sonnet
---

# SAP Management Cockpit — Expert Context

You are an expert on the **SAP Management Cockpit**, the top-level dashboard that gives a single-pane-of-glass view of the entire Fivetran SAP landscape. Use this context for any task involving the cockpit's UI, reachability checks, the Service Monitor health-check cron, the smtp2go email relay, or deployment of the static HTML.

---

## What the Management Cockpit is

A **static HTML + JavaScript dashboard** served from `sapidesecc8`, accessible at:

```
https://sapidesecc8.fivetran-internal-sales.com/sap_skills/docs/SAP_Management_Cockpit.html
```

It is the entry point for every other system cockpit (S/4HANA, ECC, SQ1, SAPRouter, HVR Hub, SSH Tunnel) and for monitoring + email-relay administration. The page itself is dumb — all "live" behaviour comes from JavaScript hitting backend APIs on `gcs_explorer_server.py`.

| Property | Value |
|---|---|
| URL | `https://sapidesecc8.fivetran-internal-sales.com/sap_skills/docs/SAP_Management_Cockpit.html` |
| Server | `sapidesecc8`, port 443 |
| Filesystem path | `/usr/sap/sap_skills/docs/SAP_Management_Cockpit.html` |
| Authoritative copy | Postgres `doc_pages` table (path `/sap_skills/docs/SAP_Management_Cockpit.html`) |
| Theme | Teal (`#0d7377`) |
| GitHub repo | [`fivetran/SAPManagementCockpit`](https://github.com/fivetran/SAPManagementCockpit) (PUBLIC) |
| Backend service | `gcs-explorer.service` (`gcs_explorer_server.py`) on sapidesecc8 |

**Two HTML files** in the cockpit family:
- `SAP_Management_Cockpit.html` — the dashboard itself (server cards, status grid, links, monitoring tabs)
- `SAP_Management_Cockpit_Documentation.html` — internal documentation page (this skill is the canonical source for it)

---

## Reachability API

The cockpit performs automatic health checks on page load by calling a backend endpoint for each of 6 servers.

| Item | Value |
|---|---|
| Method | `GET` |
| URL | `/sap_skills/api/reachability?system={system}` |
| Auth | None |
| Response | `{ "reachable": true | false }` |
| Backend handler | `gcs_explorer_server.py` on sapidesecc8 |

**System identifiers** (the `system` query param maps to a fixed server + frontend element ID):

| `system` param | Server checked | Status element ID |
|---|---|---|
| `s4hana` | sapidess4 | `reach-s4hana` |
| `ecc8` | sapidesecc8 | `reach-ecc8` |
| `sq1` | sap-sql-ides | `reach-sq1` |
| `saprouter` | saprouter (34.46.174.105) | `reach-saprouter` |
| `hvrhub` | saphvrhub | `reach-hvrhub` |
| `sshtunnel` | ts-sap-hana-s4-ssh-tunnel (10.142.0.37) | `reach-sshtunnel` |

**Status indicators (UI):**

| Color | Meaning |
|---|---|
| Gray (`#ccc`) | Initial state — check in flight |
| Green (`#27ae60`) | Server reachable |
| Red (`#e74c3c`) | Server unreachable or check failed |

**Frontend flow**: `DOMContentLoaded` → `checkReachability(system, elementId)` is called 6 times in parallel → each call hits the backend handler.

---

## External Links section

Two cards, hardcoded into the HTML:

**SAP Links**
- SAP Notes: `me.sap.com/servicessupport/search/`
- Software Downloads: `me.sap.com/softwarecenter`
- S/4HANA SSO (Okta SAML): `https://fivetran.okta.com/home/sapnetweaversaml/0oa1sojot83IxLiyy1d8/59625` — S/4 internal IP `10.128.15.239`, public SSH `35.229.110.30`

**Fivetran Links**
- Fivetran Dashboard: `fivetran.com/dashboard/connections`
- HVR Hub Console: `http://saphvrhub:4340/`
- POV App: `pov-app.fivetran-internal-sales.com`
- Slab: `fivetran.slab.com`

---

## Individual cockpit pages (linked from the dashboard)

Each server card on the Management Cockpit links to a per-system cockpit. **Don't edit those pages here** — they have their own skills:

| Server | Cockpit HTML | Owning skill |
|---|---|---|
| sapidess4 (S/4HANA) | `SAP_S4HANA_2023.html` + `_Guide.html` | `sap-skills-portal` (server doc) |
| sapidesecc8 (ECC Oracle) | `SAP_ECC6_EHP8.html` + `_Guide.html` | `sap-skills-portal` |
| sap-sql-ides (ECC SQL Server) | `SAP_ECC6_SQ1.html` + `_Guide.html` | `sq1-license` (license parts), `sap-skills-portal` (cockpit page) |
| saprouter | `SAP_SAPRouter.html` + `SAP_SAPRouter_Documentation.html` | `saprouter` |
| saphvrhub | `SAP_HVRHub.html` + `_Documentation.html` | (no dedicated skill yet) |
| SSH Tunnel | `SAP_SSHTunnel.html` + `_Guide.html` | (no dedicated skill yet) |

The Management Cockpit page itself only lists / links these — it doesn't own their content.

---

## Service Monitor

A real-time health dashboard embedded on `SAP_Monitoring.html` (linked from the Management Cockpit's Service Monitor tab). Driven by a cron-scheduled bash script.

| Component | Path | Purpose |
|---|---|---|
| Health-check script | `/usr/local/bin/sap_health_check.sh` on sapidesecc8 | Runs every 5 min via cron, probes all 6 servers + their key services |
| JSON status output | `/usr/sap/sap_skills/health_status.json` | Single-snapshot status read by the web UI |
| Append-only log | `/var/log/sap_health_check.log` | Last 2,000 lines retained |
| Health-log API | `POST /sap_skills/api/admin/health_log` | Returns last 50 log entries (admin-only) |
| Web UI | `SAP_Monitoring.html#service-monitor` | Auto-refreshing status cards |

**Health-status.json is also synced to `doc_pages`** — that's why you see version numbers in the thousands on path `/sap_skills/health_status.json` (it's overwritten every 5 min by the cron writer).

### Services probed per server

| Server | Services | Check method |
|---|---|---|
| **sapidess4** | SAP Dispatcher / Gateway / ICM / IGS; HANA FIV; HANA PIT | `sapcontrol` via SSH (instance nrs 03, 00, 96) |
| **sapidesecc8** | SAP Dispatcher / Gateway / ICM; Oracle DB; Web Server | local `sapcontrol`; `ora_pmon` process check; HTTPS curl |
| **saprouter** | Server reachability; SAPRouter service | SSH; `systemctl is-active saprouter` |
| **sap-sql-ides** | Server reachability; SAP Instance (SQ1); SQL Server DB | Ping; SOAP `sapcontrol` port 50013; WinRM via API |
| **saphvrhub** | Server reachability; HVR Hub; PostgreSQL 14 | SSH; `systemctl is-active` for both |
| **SSH Tunnel** | Server reachability | SSH to `10.142.0.37` |

### Status indicators (Service Monitor UI)

| Indicator | Meaning |
|---|---|
| Green dot (`#27ae60`) | Service running |
| Red dot (`#e74c3c`) | Service down or unreachable |
| Gray dot (`#95a5a6`) | Initial / check in progress |
| `REACHABLE` pill | Server responds to SSH / ping |
| `UNREACHABLE` pill | Server did not respond within timeout |

---

## Email Notifications / Monitoring Agent

The Management Cockpit's Monitoring Agent section manages email distribution lists. **All servers in the estate relay mail through smtp2go** via the verified `fivetran-internal-sales.com` domain.

| Property | Value |
|---|---|
| SMTP provider | smtp2go (`mail.smtp2go.com:2525`) |
| Portal "From" | `sap-skills@fivetran-internal-sales.com` |
| Verified domain | `fivetran-internal-sales.com` |
| Vault key | `smtp2go` |

### Distribution lists

| List | Recipients | Usage |
|---|---|---|
| `SAPSpecialists` | `antonio.carbone@fivetran.com`, `richard.brouwer@fivetran.com` | Alerts, notifications, reports |

**Persistence**: lists were originally rewritten to disk via the HTML page; current authoritative copy is in `mailing_lists.json` (path `/sap_skills/mailing_lists.json` in `doc_pages`).

### Email APIs

| Endpoint | Method | Description |
|---|---|---|
| `/sap_skills/api/get_mailing_list` | POST | Read a list by name |
| `/sap_skills/api/update_mailing_list` | POST | Update a list (rewrites HTML / `mailing_lists.json`) |
| `/sap_skills/api/send_test_email` | POST | Send a test email to a list |

---

## Deployment

The cockpit pages are **static HTML** served directly from disk by the `gcs-explorer` service. No restart required for HTML-only edits.

### Deploy a cockpit HTML edit

The portal canonicalises content in Postgres `doc_pages` first, then serves from disk. The clean path is: **edit local → upsert into doc_pages → backup-then-copy to /usr/sap/sap_skills/docs/**. Never write directly to `/usr/sap/sap_skills/docs/` without the doc_pages upsert step (see `feedback_never_modify_production_before_github`).

```bash
# 1. Edit local copy
vim ~/SAP_Skills/docs/SAP_Management_Cockpit.html

# 2. Upsert into Postgres (sapidesecc8)
ssh sapidesecc8 "sudo -u postgres psql -d portal" <<EOF
INSERT INTO doc_pages (path, content, content_type, sha256, version, updated_at, updated_by)
VALUES ('/sap_skills/docs/SAP_Management_Cockpit.html',
        pg_read_file('/tmp/cockpit.html'),
        'text/html; charset=utf-8',
        '$(sha256sum < /tmp/cockpit.html | cut -d' ' -f1)',
        1, now(), 'antonio.carbone@fivetran.com')
ON CONFLICT (path) DO UPDATE SET version = doc_pages.version + 1,
                                 content = EXCLUDED.content,
                                 sha256 = EXCLUDED.sha256,
                                 updated_at = now();
EOF

# 3. Sync to filesystem
scp ~/SAP_Skills/docs/SAP_Management_Cockpit.html root@sapidesecc8:/usr/sap/sap_skills/docs/
```

No `gcs-explorer` restart needed for HTML.

### Restart required after backend edits

If you modify `gcs_explorer_server.py` (e.g. to add a new reachability identifier or change the SOAP call to SQ1):

```bash
ssh sapidesecc8 'systemctl restart gcs-explorer'
```

---

## Adding a new server to the cockpit

Five-step recipe. The order matters — the UI will throw 404s if you ship the HTML before the backend handler.

1. **Create a per-server cockpit page** at `~/SAP_Skills/docs/SAP_NewServer.html` (model after one of the existing per-server HTMLs, e.g. `SAP_HVRHub.html`)
2. **Add the server card** to `SAP_Management_Cockpit.html` inside the `.status-grid` div. Card includes a `<span id="reach-newserver">` for the status dot.
3. **Add a reachability identifier** to `gcs_explorer_server.py` — extend the `system` → check function map. Choose a check method appropriate to the server (SSH+`systemctl`, ping, SOAP sapcontrol, SQL, HTTP probe, …).
4. **Add the JS call** in the `DOMContentLoaded` handler:
   ```js
   checkReachability('newserver', 'reach-newserver');
   ```
5. **Deploy** — upsert both HTMLs to `doc_pages` + scp to disk, then `systemctl restart gcs-explorer` (because step 3 changed the backend).

If the new server should also be in the **Service Monitor**, additionally:
6. Extend `/usr/local/bin/sap_health_check.sh` with a probe block for the new server
7. The script writes an extra key into `health_status.json` — make sure `SAP_Monitoring.html` reads + renders it

---

## Common gotchas

| Symptom | Cause | Fix |
|---|---|---|
| All status dots stay gray forever | `gcs-explorer` not running, or reachability handler crashing on every call | `systemctl status gcs-explorer`; tail journalctl for the service; if a single handler raises, **all** subsequent POST handlers in the same `do_POST` may break (see `feedback_portal_post_handlers_empty_reply`) |
| One server shows red but is alive | Check method is wrong for that server's actual stack (e.g. ping doesn't work over GCP private IPs without a route, SOAP port closed by firewall) | Inspect the handler in `gcs_explorer_server.py`; switch SSH-based check to one that doesn't require key-auth from the portal user |
| Edits to the HTML "don't appear" after scp | Browser cache or static-file ETag mismatch | Hard-reload (Cmd-Shift-R); verify `doc_pages.sha256` matches the file you scp'd |
| sapcontrol checks return spurious red on sapidess4 | XSA / instance not started yet after a HANA restart | Wait 60–120 s after `HDB start`; also check `sapcontrol -nr 03 -function GetProcessList` directly |
| SQ1 reachability flips green/red | WinRM session timeout race; SOAP port 50013 returns slowly under load | Increase the per-check timeout in `gcs_explorer_server.py`; SQ1 also has its own `Rotation#1` os-password drift to be aware of (see `sq1-license` skill) |
| `health_status.json` has stale `last_update` timestamp | Cron not running, or `sap_health_check.sh` failed | `crontab -l` on sapidesecc8; `tail -50 /var/log/sap_health_check.log` |
| Mailing-list edit doesn't show in cockpit UI | The `mailing_lists.json` was re-read but cached browser state shows old | Hard-reload; verify the JSON's `updated_at` advanced via doc_pages |
| New server card pushes layout off-screen | `.status-grid` uses CSS grid with `auto-fit minmax(280px, 1fr)` — adding a 7th breaks the visual balance | Either rebalance with explicit `grid-template-columns`, or split into two grids |

---

## Cross-references

- **Per-server cockpits**: `saprouter`, `sq1-license`, `sap-cds`, `bwextractors`, plus the per-system docs in the SAP Skills Portal
- **Backend / portal infra**: `SAP_Skills:sap-skills-portal` — the umbrella skill for `gcs_explorer_server.py`, `doc_pages`, `vault_secrets`, the mailing-list mechanics, and the systemd services
- **Backups + rotation**: `backint`, `saporaclebackup`
- **Memory hooks**:
  - `feedback_never_modify_production_before_github` — never write directly to `/usr/sap/sap_skills/docs/` without the doc_pages step
  - `feedback_portal_post_handlers_empty_reply` — adding a new `/api/...` POST handler can mask itself behind a NameError; always test
  - `project_otc_multilang_20260428` — example of the "translate + upsert + scp" workflow used for the per-server cockpit pages
