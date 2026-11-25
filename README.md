# Team .venv — Private Network Service Platform

Computer Networks course project. Four macOS laptops on one Wi-Fi network serve `https://app.team1.test`
through a private DNS server, an nginx TLS edge / load balancer and two REST backends. No cloud.

Team name: **.venv** · private DNS zone: **team1.test** (the zone name used in all configs and evidence).

## Phase 2 — demonstrated 24–25 November 2026

Three available Macs retain the native nginx/Python platform. Docker replaces the
absent DNS laptop and provides backup DNS on a second physical Mac.

| Member | Current LAN IP | Roles |
|---|---|---|
| Anshuman | 10.7.31.231 | Primary Docker DNS + original nginx edge |
| Bineet | 10.7.1.112 | Backend A :3001 + client |
| Vridhi (hostname Pranshus-MacBook-Pro) | 10.7.2.240 | Backup Docker DNS + Backend B :3002 + temporary standby edge |

Extensions A–E have recorded demonstrations, including rollback. The actual
faculty-injected troubleshooting challenge and individual viva remain pending.
Firewall rules and standby nginx were rolled back after testing; they are not
claimed to be continuously active. DHCP addresses must be checked before reuse.

| Start here | File |
|---|---|
| Current classroom demo scenes | [Start with Scene 00, then choose TTL, DNS, failover, isolation or migration by name](05-Phase2-Final-Report/SIR-LIVE-DEMO-CURRENT-IPS.md) |
| Results and learning summaries | [Phase 2 report](05-Phase2-Final-Report/REPORT.md) / [printable PDF](output/pdf/Phase2-Final-Report.pdf) |
| Evaluation-day commands | [Demo checklist](05-Phase2-Final-Report/DEMO-CHECKLIST.md) |
| Viva study | [53 questions and answers](05-Phase2-Final-Report/VIVA-GUIDE.md) / [PDF](output/pdf/Viva-Guide.pdf) |
| Beginner Wireshark demo | [Vridhi's capture steps, filters and Hinglish explanation](05-Phase2-Final-Report/WIRESHARK-QUICKSTART.md) |
| Full teacher-request command book | [Roles, expected results and rollback](05-Phase2-Final-Report/LIVE-COMMANDS.md) / [PDF](output/pdf/Live-Commands.pdf) |
| Hostel-to-classroom / changed IPs | [Restart procedure](05-Phase2-Final-Report/CLASSROOM-RESTART.md) / [PDF](output/pdf/Classroom-Restart.pdf), [configuration helper](03-Source-Code/classroom-prepare.py) |
| Updated topology and inventory | [Phase 2 architecture](01-Architecture-Document/Phase2-Architecture.md) |
| Requirement-to-evidence map | [Phase 2 evidence index](04-Evidence/Phase2/EVIDENCE-INDEX.md) |
| Sanitized configuration snapshots | [Phase 2 configuration bundle](02-Configuration-Bundle/Phase2/README.md) |
| Explanation and detailed procedure | [Phase 1 understanding / Phase 2 guide](03-Source-Code/PHASE2-GUIDE.md) |
| Docker DNS setup / live record helpers | [DNS runbook](03-Source-Code/docker-dns/README.md) |

The classroom session changed Anshuman's IP to 10.7.16.25 and Bineet's to
10.7.17.47; Vridhi remains 10.7.2.240. Use live team.env and the current-demo guide
for those addresses. The recorded report/table above retains the previous
session's addresses. macOS on Anshuman required /etc/resolver/team1.test to route
private-domain system lookups correctly. The current-demo guide uses that
per-domain resolver consistently; it takes precedence over Wi-Fi DNS settings.

## Phase 1 — historical October deployment

The original four-Mac platform and evidence are preserved below. The IPs in this
section describe the October deployment, not the current three-Mac setup.

| Machine | Member | Role | IP |
|---|---|---|---|
| Mac 1 | Rudraksh | Private DNS (dnsmasq) + test client | 10.7.6.171 |
| Mac 2 | Anshuman | Edge: nginx reverse proxy, TLS termination, load balancer | 10.7.20.28 |
| Mac 3 | Bineet | Backend A (port 3001) | 10.7.20.243 |
| Mac 4 | Vridhi Chaudhary | Backend B (port 3002) + main test client | 10.7.8.144 |

## Where everything is

| Deliverable (Section 9 of the brief) | Folder | Start with |
|---|---|---|
| **Architecture Document** — topology, roles/IP/service table, request-flow diagram per protocol layer | [`01-Architecture-Document/`](01-Architecture-Document/) | `Architecture-Document.pdf` |
| **Configuration Bundle** — dnsmasq config, nginx config, TLS setup notes, backend launch instructions | [`02-Configuration-Bundle/`](02-Configuration-Bundle/) | `README.md` |
| **Backend Source Code** — both backends and all helper scripts | [`03-Source-Code/`](03-Source-Code/) | `README.md`, `backend/server.py` |
| **Evidence Folder** — DNS, curl/browser headers, Wireshark (DNS, TCP, TLS), caching, all failure scenarios | [`04-Evidence/`](04-Evidence/) | `EVIDENCE-INDEX.md` |

## Phase 1 at a glance

| Task | Result | Evidence |
|---|---|---|
| A · Private LAN | Pass | `04-Evidence/A-LAN-Setup/` |
| B · Private DNS | Pass — `app.team1.test → 10.7.20.28`, used by Mac 2, 3, 4 | `B-Private-DNS/` |
| C · Two backends | Pass — :3001 and :3002 with `X-Backend` | `C-Backends/` |
| D · Edge + load balancing | Pass — A/B round robin over HTTP/2 | `D-Load-Balancing/` |
| E · HTTPS / TLS | Pass — own CA, `SSL certificate verify ok`, no `-k` | `E-HTTPS-TLS/` |
| F · HTTP caching | Pass — `Cache-Control`, `ETag`, `304 Not Modified` | `F-HTTP-Caching/` |
| G · Protocol capture | Pass — DNS → TCP → TLS → encrypted HTTP in Wireshark | `G-Packet-Capture/` |
| 6.3 Failure scenarios (all 5) | Done — wrong DNS server, wrong record, one backend down, both down (502), wrong port | `H-Failure-Scenarios/` |
