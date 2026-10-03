# VMS Cloud: ANPR video management with Windows site agents

```
 Site (Windows PC)                                   Cloud server (Ubuntu + Docker)
 ┌───────────────────────────────┐                  ┌──────────────────────────────────────┐
 │ IP cameras ──RTSP──► vms-agent │ ──HTTPS 443───► │ Caddy (auto HTTPS) ─► API (FastAPI)  │
 │   • reads every camera         │  events, photos │                        │             │
 │   • plate recognition (ONNX)   │  status, config │                    PostgreSQL        │
 │   • offline queue (SQLite)     │                 │                                      │
 │   • live video on demand       │ ──RTSP 8554───► │ MediaMTX ─► HLS live view, recording │
 └───────────────────────────────┘  (outbound only)└──────────────────────────────────────┘
                                                          ▲ browser: dashboard (login)
```

- **Plate recognition runs on the site PC.** Only small events (plate text + 2 JPEGs) go to the cloud.
- **Every connection is outbound from the site.** No port forwarding or static IP is needed at sites.
- **Everything is set up on the server.** Create a site with its cameras' RTSP links, download the agent
  made for that site, and run it on the site PC. There is nothing to type at the site.
- **Offline-safe.** If the internet drops, events queue on disk (up to 50,000) and upload when it returns.
- **Live view on demand.** Video streams only while someone watches, so idle sites use almost no upload bandwidth.
  Optional 24/7 recording per camera, stored on the server.

```
server/   Ubuntu server: docker-compose.yml, install.sh, API + dashboard source
agent/    Windows agent: Python source, build.ps1 (exe + installer), Inno Setup script
```

---

## 1. Server (Ubuntu 22.04 / 24.04)

**Size:** ANPR runs at the sites, so the server is light. Use at least **2 GB RAM** (AWS `t3.small`) and
**30 GB disk**. For 24/7 recording, add about **20 GB per camera per day** at 2 Mbit/s. A sub stream uses less.

**Firewall / AWS security group (inbound):** TCP **22** (SSH), **80**, **443** (dashboard + agent API),
**8554** (live video from agents).

```bash
# copy the server/ folder to the machine (scp, git, ...), then:
cd server
sudo bash install.sh
```

The installer installs Docker, asks how the server is reached (see HTTPS below) and for the admin password,
generates secrets into `.env`, and starts everything. Open the URL it prints and sign in.

**HTTPS.** Certificates come from Let's Encrypt and renew automatically. Ports 80 and 443 must be open.
Change the mode at any time with `set-address.sh`:

| Command (in `server/`) | Dashboard / agent address |
|---|---|
| `sudo bash set-address.sh vms.example.com` | `https://vms.example.com`: your domain, with a DNS A record pointing at the server (best for production) |
| `sudo bash set-address.sh sslip` | `https://3-110-14-104.sslip.io`: a free name that always resolves to your IP, no DNS setup |
| `sudo bash set-address.sh ip` | `https://3.110.14.104`: directly on the IP, using a short-lived (6-day) IP certificate |
| `sudo bash set-address.sh http` | `http://3.110.14.104`: no encryption, testing only |

Use an **Elastic IP** in AWS, so the IP (and the sslip name) never changes. The script also sets `PUBLIC_URL`,
so new agent downloads use the new address. Agents installed earlier keep the old address. Download the agent
again and run it at each site, or on the site PC run `vms-agent configure --server <new address>` and then
`vms-agent restart` (in an admin prompt).
- **Verify the deployment:** `pip install requests && python tests/smoke_test.py https://vms.example.com admin PASSWORD`.
  It creates a temporary site, checks every endpoint, and deletes the site again.

| Task | Command (in `server/`) |
|---|---|
| Logs | `docker compose logs -f` (or `logs -f api`) |
| Update after changing files | `docker compose up -d --build` |
| Stop / start | `docker compose down` / `docker compose up -d` |
| Reset a password | `docker compose exec api python -m app.manage reset-password admin 'NewPass123'` |
| Add a user from the command line | `docker compose exec api python -m app.manage add-user ravi 'Pass1234' operator` |
| List users | `docker compose exec api python -m app.manage list-users` |
| Back up the database | `docker compose exec db pg_dump -U vms vms > backup.sql` |

Settings live in `server/.env` (see `.env.example`): event retention (`EVENT_RETENTION_DAYS`, default 90),
recording retention (`RECORD_RETENTION`, default 7d), and ports.

## 2. Build the agent installer and upload it (once)

On any Windows 10/11 PC with internet, with Python 3.10–3.12 and
[Inno Setup 6](https://jrsoftware.org/isdl.php) (`winget install JRSoftware.InnoSetup`):

```powershell
cd agent
powershell -ExecutionPolicy Bypass -File build.ps1
```

This creates `agent\output\VMS-Agent-Setup-1.1.1.exe` (about 85 MB, includes the ANPR models and ffmpeg)
and a portable zip. In the dashboard open **Settings → Agent installer** and upload the `.exe`.
Upload a newer build whenever you update the agent.

## 3. Add a site and download its agent

1. **Sites → Add site**: enter the name and paste the cameras' RTSP links, one per line, for example
   `Entry gate, rtsp://admin:pass@192.168.1.64:554/Streaming/Channels/101`. Use the addresses
   *as seen from the site PC*. Fine-tune each camera later on the Cameras page (sub stream, frames per
   second, live mode, recording).
2. Click **Download agent**. You get `VMS-Agent-Setup-<site name>.exe`, the uploaded installer with this
   site's server address and key built in. The address comes from `PUBLIC_URL` in `.env`, and you can edit
   it in the download dialog. It must be reachable from the site: a domain or public IP, not localhost.
3. At the site, run that file as administrator and click **Next → Install**. There is nothing to type.
   The installer adds and starts the **VMS ANPR Agent** Windows service (auto-start, restarts on failure).
   The site turns **Online** within about 15 seconds and its cameras start. Silent install: add `/VERYSILENT`.

Cameras stay managed on the server: add, edit or remove them in the dashboard and the agent applies the change
within about 1 second. There's no need to touch the site PC again.

**Keys:** each site's key is derived from `SECRET_KEY` in `.env`. Never change `SECRET_KEY` after rollout,
because that disconnects every agent. **Sites → Key → Replace key** disconnects that site's agent and makes
earlier downloads useless. Download and install a fresh copy there afterwards.

Other ways to install: the generic `VMS-Agent-Setup-1.1.1.exe` asks for the server address and key
(or `/VERYSILENT /SERVER="https://vms.example.com" /KEY="s1.xxxx"`). Or unzip the portable zip and run
`vms-agent install --server https://vms.example.com --key s1.xxxx` in an admin prompt.
## Users and roles

Add users on the dashboard under **Users → Add user** (administrators only). Pick a role and the sites the
user may see:

| Role | Can do |
|---|---|
| **Administrator** | Everything: sites, cameras, agent downloads, users and settings. Always sees all sites. |
| **Operator** | Live plates, events and CSV export, live video, recordings, and adding or removing watchlist plates. |
| **Viewer** | View only: live plates, events, cameras, live video and recordings. |

Operators and viewers can be limited to **specific sites**. They then see only those sites' cameras, events,
live video and recordings, and never camera passwords or site keys. Under **Menu items this user sees** you choose
which pages appear for the user (Live ANPR, Events, Cameras, Watchlist, Recordings). Settings is always available.
The server enforces this too: for example, without *Recordings* the user's recording and playback requests are
refused, not just hidden. Changing a user's role, sites, menu items or status
signs them out, so the change applies immediately. **Disable** blocks sign-in without deleting the user. The
system always keeps at least one active administrator.

## Plate recognition (your trained model)

The agent reads plates with **your model** from `Model.zip`, kept unchanged in `agent/plate_ocr/`:
`ocr_api.py` (RapidOCR pipeline with Indian plate rules), the fallback helpers, and `models/plate_det.onnx`,
`plate_rec.onnx` and `en_dict.txt`. A small fast detector spots and tracks each vehicle (about 4 frames per
second) and keeps its two sharpest frames. When the vehicle has passed, your model reads the best frame, and the
second one if needed. That takes about 1 to 4 seconds per vehicle on a 4-core PC. The plate and your model's status
(`ok`, `uncertain` or `partial`) go to the server, where the dashboard shows the status and can filter by it.

- **Update the model:** replace the files in `agent/plate_ocr/`, run `build.ps1`, upload the new installer.
  On a single site PC you can also stop the service, replace the files in
  `C:\Program Files\VMS Agent\plate_ocr\` and start it again.
- **Check a picture:** `vms-agent test-image photo.jpg` shows what the detector finds and what your model reads.
- **Size:** the installer is about 230 MB, because your detector model alone is 175 MB.
- **Built-in recogniser:** set `"plate_engine": "fast"` in `C:\ProgramData\VMSAgent\agent.json` to use it instead.

## Integration API (data for another website)

Administrators create API keys under **Settings → Integration API**. Each key can be limited to sites and to
the other system's IP addresses, and contact details are optional. The API is read-only (`/api/v1/me`, `/events`,
`/events/{id}`, `/sites`, `/cameras`). It uses the header `X-API-Key`, allows 120 requests per minute, and returns
images as signed links valid for 24 hours. `GET /api/v1/events?since_id=N` returns only new detections,
oldest first. Give the developers **`docs/VMS-Cloud-API-Reference.pdf`**, which includes examples in curl, Python,
PHP and C#.

## Districts, categories and site details

- **Districts and categories.** All 33 Telangana districts are loaded automatically. Administrators add site
  categories (for example *Sand stockyard*, *Check post*) and can rename or add districts under **Settings**. Set a
  site's district and category under Sites → Edit or Site Details → Edit. The Cameras page can be filtered and
  grouped by district, category or site. Events can be filtered by district and category, and the CSV export
  includes both.
- **Site Details page.** Each site has owner and helper contacts, address, GPS location (paste coordinates or a
  Google Maps link), stockyard name, stock point ID, installation date, invoice uploads (PDF, JPG, PNG or WEBP, up
  to 20 MB each) and remote access details (UltraViewer ID and password, Tailscale IP, username and password).
- **Who can do what.** Administrators and operators create and edit details and upload invoices, for their
  assigned sites. Viewers only view, for their assigned sites. Remote access details are shown to administrators
  and operators only. Passwords are stored encrypted (derived from `SECRET_KEY`), stay masked until **Show** is
  clicked, and every reveal is logged on the server.

## 4. Troubleshooting at the site (Start menu → VMS Agent, or Command Prompt)

```
vms-agent status                   service, server connection, each camera's state and errors
vms-agent check                    server address, site key and live-video port 8554
vms-agent test-camera "rtsp://..." grabs a frame and tries to read a plate
```
The log is at `C:\ProgramData\VMSAgent\logs\agent.log`.

## 5. Tuning and limits

- **CPU:** a 4-core PC handles about 2–4 ANPR cameras at 4 fps. Recognition skips frames with no motion, so
  idle cameras cost little. Lower *ANPR frames per second* on busy PCs; raise it for fast traffic.
- **Duplicates:** one event per vehicle pass. The same plate is ignored while it stays in view and for
  *Ignore repeats* seconds afterwards (default 60).
- **Plate styles:** the bundled OCR model (`cct_xs_v2_global`) reads Latin-alphabet plates of up to 10
  characters. It was trained on 60+ countries, but India is not in its training list. Test it on your real
  cameras with `vms-agent test-camera` before rolling out, and raise *Minimum confidence* if you see misreads.
  Two-line plates (motorcycles, some trucks) are less reliable.
- **Camera placement matters most:** plates at least about 100 px wide in the image, not too steep an angle,
  and IR illumination at night.
- **Live video codec:** browsers play H.264. For H.265 cameras use the H.264 sub stream, or tick
  *Convert to H.264* (costs site CPU).
- **Security:** agent traffic uses HTTPS when you configure a domain. Live video from agents to port 8554 is
  RTSP with a per-site password and is not encrypted. Put the server behind a VPN if that matters for your sites.
"# Cloud-vms" 
"# VMS-CLOUD" 
