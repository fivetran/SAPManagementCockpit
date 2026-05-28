# SAP Management Cockpit

The **SAP Management Cockpit** is the single-pane-of-glass entry point for the entire Fivetran SAP demo estate. It is a static HTML dashboard hosted on `sapidesecc8` that shows the live reachability and service health of every SAP system, links to each system's individual cockpit page, and provides access to monitoring, email alerting, and administrative tools — all from one URL.

Any Sales Engineer working with the SAP demo environment starts here.

```
https://sapidesecc8.fivetran-internal-sales.com/sap_skills/docs/SAP_Management_Cockpit.html
```

---

## What's on the dashboard

### System status grid

Six server cards, each with a live reachability indicator that updates on page load:

| Server card | System | What it links to |
|---|---|---|
| **S/4HANA 2023** | `sapidess4` — SAP S/4HANA SPS05 | S/4HANA cockpit + setup guide |
| **ECC 6.0 EHP8** | `sapidesecc8` — SAP ECC on Oracle 19c | ECC cockpit + setup guide |
| **ECC SQ1** | `sap-sql-ides` — SAP ECC on SQL Server 2012 | SQ1 cockpit + setup guide |
| **SAPRouter** | `saprouter` (34.46.174.105) | SAPRouter cockpit + documentation |
| **HVR Hub** | `saphvrhub` — HVR/HVA replication control plane | HVR Hub cockpit + documentation |
| **SSH Tunnel** | `ts-sap-hana-s4-ssh-tunnel` (10.142.0.37) | SSH Tunnel guide |

Each card shows a colored dot — **green** (reachable), **red** (unreachable), or **gray** (check in flight) — updated automatically when the page loads via 6 parallel API calls to `/sap_skills/api/reachability?system={system}`.

### External links

Two quick-access link cards are hardcoded into the page:

**SAP links**: SAP Notes search (`me.sap.com`), Software Downloads Center, S/4HANA SSO login via Okta SAML.

**Fivetran links**: Fivetran Dashboard, HVR Hub Console (`saphvrhub:4340`), POV App, Slab.

### Monitoring tabs

The dashboard includes tabs that surface:
- **Service Monitor** — per-service health across all 6 servers (see below)
- **Monitoring Agent** — email distribution list management

---

## How the reachability checks work

On page load, JavaScript calls the backend for each server in parallel:

```
GET /sap_skills/api/reachability?system=s4hana
→ { "reachable": true }
```

The backend (`gcs_explorer_server.py` on `sapidesecc8`) maps each `system` identifier to a probe appropriate for that server's stack:

| `system` param | Server | Probe method |
|---|---|---|
| `s4hana` | `sapidess4` | SSH + `sapcontrol GetProcessList` |
| `ecc8` | `sapidesecc8` | Local `sapcontrol` |
| `sq1` | `sap-sql-ides` | SOAP `sapcontrol` port 50013 + WinRM |
| `saprouter` | `saprouter` (34.46.174.105) | SSH + `systemctl is-active saprouter` |
| `hvrhub` | `saphvrhub` | SSH + `systemctl is-active` |
| `sshtunnel` | `ts-sap-hana-s4-ssh-tunnel` (10.142.0.37) | SSH |

---

## Service Monitor

The **Service Monitor** tab shows fine-grained health — not just "is the server up?" but "is the SAP Dispatcher running? Is the Oracle DB process alive? Is the HVR Hub service active?" — updated every 5 minutes by a cron-driven bash script.

**How it works:**

1. `/usr/local/bin/sap_health_check.sh` runs every 5 minutes on `sapidesecc8`
2. It probes all 6 servers and writes a snapshot to `/usr/sap/sap_skills/health_status.json`
3. The `SAP_Monitoring.html` page reads that JSON and renders status cards with green/red/gray dots

**Services probed per server:**

| Server | Services checked |
|---|---|
| `sapidess4` | SAP Dispatcher, Gateway, ICM, IGS; HANA tenant FIV; HANA tenant PIT |
| `sapidesecc8` | SAP Dispatcher, Gateway, ICM; Oracle DB (`ora_pmon` process); Web Server (HTTPS curl) |
| `saprouter` | Server reachability; `saprouter` systemd service |
| `sap-sql-ides` | Server ping; SAP instance (SOAP port 50013); SQL Server DB (WinRM) |
| `saphvrhub` | Server reachability; HVR Hub service; PostgreSQL 14 |
| SSH Tunnel | Server reachability |

**Status indicators:**

| Indicator | Meaning |
|---|---|
| Green dot | Service running |
| Red dot | Service down or unreachable |
| Gray dot | Check in progress or initial state |
| `REACHABLE` pill | Server responded to SSH/ping |
| `UNREACHABLE` pill | Server timed out |

> `health_status.json` is also stored in the Postgres `doc_pages` table — that's why its version number is in the thousands (overwritten every 5 minutes by the cron).

---

## Email monitoring and alerts

The **Monitoring Agent** section of the cockpit manages the mailing list for system alerts, certificate expiry warnings, and health notifications. All outbound mail from the SAP estate is relayed through **smtp2go** using the verified domain `fivetran-internal-sales.com`.

| Property | Value |
|---|---|
| SMTP provider | smtp2go (`mail.smtp2go.com:2525`) |
| From address | `sap-skills@fivetran-internal-sales.com` |
| Distribution list | `SAPSpecialists` → `antonio.carbone@fivetran.com`, `richard.brouwer@fivetran.com` |
| Vault key | `smtp2go` |

**Email APIs** (all POST):

| Endpoint | Purpose |
|---|---|
| `/sap_skills/api/get_mailing_list` | Read a list by name |
| `/sap_skills/api/update_mailing_list` | Update a list (rewrites `mailing_lists.json`) |
| `/sap_skills/api/send_test_email` | Send a test message to a list |

---

## Backend architecture

The cockpit is a **static HTML + JavaScript** page — it has no server-side rendering. All dynamic behaviour comes from JavaScript calling backend APIs on `gcs_explorer_server.py`.

| Property | Value |
|---|---|
| URL | `https://sapidesecc8.fivetran-internal-sales.com/sap_skills/docs/SAP_Management_Cockpit.html` |
| Hosting server | `sapidesecc8`, port 443 |
| Filesystem path | `/usr/sap/sap_skills/docs/SAP_Management_Cockpit.html` |
| Authoritative copy | Postgres `doc_pages` table, path `/sap_skills/docs/SAP_Management_Cockpit.html` |
| Backend service | `gcs-explorer.service` (`gcs_explorer_server.py`) |
| Theme colour | Teal (`#0d7377`) |

**Two HTML files** make up the cockpit:
- `SAP_Management_Cockpit.html` — the live dashboard (server cards, status grid, monitoring tabs)
- `SAP_Management_Cockpit_Documentation.html` — internal documentation page

---

## Deploying changes

HTML-only edits take effect immediately — no restart needed. The portal canonicalises content through Postgres first:

```bash
# 1. Edit locally
vim ~/SAP_Skills/docs/SAP_Management_Cockpit.html

# 2. Upsert into Postgres
ssh sapidesecc8 "sudo -u postgres psql -d portal" <<EOF
INSERT INTO doc_pages (path, content, content_type, sha256, version, updated_at, updated_by)
VALUES ('/sap_skills/docs/SAP_Management_Cockpit.html',
        pg_read_file('/tmp/cockpit.html'), 'text/html; charset=utf-8',
        '$(sha256sum < /tmp/cockpit.html | cut -d' ' -f1)', 1, now(),
        'antonio.carbone@fivetran.com')
ON CONFLICT (path) DO UPDATE SET version = doc_pages.version + 1,
  content = EXCLUDED.content, sha256 = EXCLUDED.sha256, updated_at = now();
EOF

# 3. Copy to disk
scp ~/SAP_Skills/docs/SAP_Management_Cockpit.html \
    root@sapidesecc8:/usr/sap/sap_skills/docs/
```

After editing `gcs_explorer_server.py` (backend changes — e.g. new reachability probe):

```bash
ssh sapidesecc8 'systemctl restart gcs-explorer'
```

---

## Adding a new server

Five steps — order matters (the UI will 404 if HTML ships before the backend handler):

1. **Create a per-server cockpit page** at `~/SAP_Skills/docs/SAP_NewServer.html` (model after an existing one such as `SAP_HVRHub.html`)
2. **Add a server card** to `SAP_Management_Cockpit.html` inside `.status-grid`, including `<span id="reach-newserver">` for the status dot
3. **Add a reachability handler** in `gcs_explorer_server.py` — extend the `system` → probe function map with the appropriate check method for that server's stack
4. **Add the JS call** in `DOMContentLoaded`:
   ```js
   checkReachability('newserver', 'reach-newserver');
   ```
5. **Deploy** — upsert both HTMLs into `doc_pages`, scp to disk, then `systemctl restart gcs-explorer`

To also add the server to the **Service Monitor**, additionally:
- Extend `/usr/local/bin/sap_health_check.sh` with a probe block for the new server
- Add a rendering section in `SAP_Monitoring.html` to display the new key from `health_status.json`

---

## Common issues

| Symptom | Cause | Fix |
|---|---|---|
| All status dots stay gray | `gcs-explorer` not running, or reachability handler crashing | `systemctl status gcs-explorer`; check `journalctl -u gcs-explorer -n 50` |
| One server shows red but is alive | Wrong probe method for that server's stack | Inspect the handler in `gcs_explorer_server.py`; switch to an SSH-based check |
| HTML edits don't appear after scp | Browser cache or ETag mismatch | Hard-reload (`Cmd-Shift-R`); verify `doc_pages.sha256` matches the deployed file |
| `sapcontrol` checks spuriously red on `sapidess4` | XSA / instance not started yet after HANA restart | Wait 60–120 s after `HDB start`; run `sapcontrol -nr 03 -function GetProcessList` manually |
| SQ1 reachability flips green/red intermittently | WinRM timeout race; SOAP port 50013 slow under load | Increase per-check timeout in `gcs_explorer_server.py` |
| `health_status.json` has stale `last_update` | Cron not running or `sap_health_check.sh` failing | `crontab -l` on `sapidesecc8`; `tail -50 /var/log/sap_health_check.log` |
| Mailing list edit not reflected in UI | Cached browser state | Hard-reload; verify `mailing_lists.json` `updated_at` advanced in `doc_pages` |
| New server card breaks layout | `.status-grid` `auto-fit minmax(280px, 1fr)` doesn't balance 7 cards | Use explicit `grid-template-columns` or split into two grids |

---

## Individual system cockpits

Each server card on the Management Cockpit links to a dedicated per-system cockpit. Those pages have their own documentation and skills — don't edit them here:

| Server | Cockpit HTML | Skill |
|---|---|---|
| `sapidess4` (S/4HANA) | `SAP_S4HANA_2023.html` + `_Guide.html` | `SAP_Skills:sap-skills-portal` |
| `sapidesecc8` (ECC Oracle) | `SAP_ECC6_EHP8.html` + `_Guide.html` | `SAP_Skills:sap-skills-portal` |
| `sap-sql-ides` (ECC SQL Server) | `SAP_ECC6_SQ1.html` + `_Guide.html` | `sq1-license` |
| `saprouter` | `SAP_SAPRouter.html` + `SAP_SAPRouter_Documentation.html` | `saprouter` |
| `saphvrhub` | `SAP_HVRHub.html` + `_Documentation.html` | — |
| SSH Tunnel | `SAP_SSHTunnel.html` + `_Guide.html` | — |

---

## Related

| Resource | Link |
|---|---|
| SAP Skills Portal | [sapidesecc8.fivetran-internal-sales.com/sap_skills](https://sapidesecc8.fivetran-internal-sales.com/sap_skills/) |
| SAP Skills Portal repo | [fivetran/SAPSkillsPortal](https://github.com/fivetran/SAPSkillsPortal) |
| Service Monitor page | `/sap_skills/docs/SAP_Monitoring.html` |

---

## Contacts

**Antonio Carbone** and **Richard Brouwer** — Fivetran SAP Specialists.
