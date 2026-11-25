# Moving from hostel to classroom
## Reconnect, update addresses, and restore the working project

Your IPs MAY change; reconnecting does not guarantee a change. DHCP may return
the same lease or assign another address. Always check all three Macs after
arriving. Do not manually assign yesterday's IPs: another machine may own them.

## 1. Before leaving the hostel

- Commit/push the new study guides and helpers; all three teammates pull them.
- Keep the working certificates and local CA trust. A changed IP does not require
  a new certificate because the certificate identifies app/api.team1.test.
- Keep Docker images built locally on both DNS Macs. No download is needed for
  normal classroom startup. Keep the encrypted certificate backup and import
  password available privately; do not put them in Git.
- Finish any demonstration rollback: primary DNS restored, backend PF rules
  cleared and their enable references released, standby stopped, Supportdesk
  restored if it was paused. Do not enable isolation before basic startup works.
- Expect a sleeping laptop or closed Terminal to interrupt services. Recheck
  everything in class; do not assume last night's processes are still running.

## 2. In class: get the NEW three-IP inventory

All three join the same intended Wi-Fi or controlled hotspot. On each Mac:

```bash
route -n get default | awk '/interface:/{print $2}'
ipconfig getifaddr en0
ipconfig getoption en0 subnet_mask
```

If the active Wi-Fi interface is not en0, use its actual name in the commands
and in the helper's --interface option. Turn off a VPN that is changing the
intended path. A shared Wi-Fi name does not guarantee devices can communicate;
college networks may isolate clients. Check service connectivity before blaming
the application. A 169.254.x.x address is a reason to investigate LAN/DHCP setup.

Write this down before any edits:

| Role | Actual classroom IP |
|---|---|
| Anshuman: edge + primary DNS | __________________ |
| Bineet: Backend A | __________________ |
| Vridhi: Backend B + backup DNS | __________________ |

Last session used 10.7.31.231 / 10.7.1.112 / 10.7.2.240. Those are historical
values until rechecked. If all three are unchanged, skip configuration edits
and go to service startup. If any changed, continue to the helper below.

Open a Terminal in the repo first: type `cd `, drag the repo folder from Finder
into Terminal, then press Enter. On Anshuman's Mac the existing path is
/Users/anshumanmehta/Documents/venv-cn-project. The other Macs may use different
paths. Never copy another person's absolute repo path.

<!-- pagebreak -->

## 3. Update files after IP changes

The helper updates native team.env, role-specific Docker .env, both firewall
templates, standby upstreams, and the installed nginx config relevant to that
Mac. It backs up changed files under ~/.cn-project-backups/classroom-... . It
does NOT alter certificate/key files, start/reload services, set client DNS or
enable PF. Historical report/evidence files are not rewritten.

Everyone uses the SAME three current addresses. In a repo Terminal set these
three values. Replace the quoted text with the actual IPs, not the role names:

```bash
CN_NEW_EDGE='PUT_ANSHUMANS_CURRENT_IP_HERE'
CN_NEW_A='PUT_BINEETS_CURRENT_IP_HERE'
CN_NEW_B='PUT_VRIDHIS_CURRENT_IP_HERE'
```

The helper rejects placeholders, invalid/duplicate IPs and a role whose expected
IP does not match this Mac's current interface. Choose ONE role command per Mac.

Anshuman previews:

```bash
python3 03-Source-Code/classroom-prepare.py --role edge \
  --edge-ip "$CN_NEW_EDGE" --backend-a-ip "$CN_NEW_A" \
  --backend-b-ip "$CN_NEW_B"
```

Bineet previews the same command with `--role backend-a`. Vridhi previews it
with `--role backend-b`. If interface differs, add `--interface en1` (using the
actual interface, not necessarily en1).

If the preview matches the intended Mac/addresses, repeat that SAME command
with `--apply` at its end. Do not run this whole helper with sudo. Example for
Anshuman, after the variables above contain real addresses:

```bash
python3 03-Source-Code/classroom-prepare.py --role edge \
  --edge-ip "$CN_NEW_EDGE" --backend-a-ip "$CN_NEW_A" \
  --backend-b-ip "$CN_NEW_B" --apply
```

The helper preserves an existing UPSTREAM_DNS value. If the new network blocks
it, identify a reachable original Wi-Fi resolver with `scutil --dns` before
changing system DNS, and add `--upstream-dns ACTUAL_IP` to the helper command.
If old manual project DNS settings obscure the new network's DHCP-provided
resolver, inspect `ipconfig getoption en0 domain_name_server` using the actual
Wi-Fi interface; choose a reachable resolver from that network's DHCP settings.
Project .test answers are local; unrelated internet DNS requires an upstream.

Classroom update: if /etc/resolver/team1.test exists, update its nameserver lines
to the two CURRENT DNS host IPs with sudo too. The helper intentionally does not
write root-owned system resolver files. That per-domain file takes precedence
over the general Wi-Fi DNS setting. Use SIR-LIVE-DEMO-CURRENT-IPS.md for the
current session's complete resolver-aware demonstration commands.

## 4. Load current addresses in EVERY new Terminal

From anywhere inside the repo:

```bash
source "$(git rev-parse --show-toplevel)/03-Source-Code/classroom-shell.sh"
```

It prints the loaded edge, A and B addresses. Commands in LIVE-COMMANDS.md use
CN_EDGE_IP, CN_A_IP, CN_B_IP and both DNS variables from this file. Re-source
after changing team.env. Empty CN_ variables mean you skipped this step.

<!-- pagebreak -->

## 5. Restart/recheck services in this order

Bineet, in a dedicated Terminal (leave it open):

```bash
cd "$CN_REPO/03-Source-Code"
/usr/bin/python3 backend/server.py A 3001
```

Vridhi, in a dedicated Terminal (leave it open):

```bash
cd "$CN_REPO/03-Source-Code"
python3 backend/server.py B 3002
```

If a backend is already listening, check its identity with curl before starting
a duplicate. Address already in use usually means that port already has a process.

Anshuman AND Vridhi: open Docker Desktop and wait until the engine is ready:

```bash
open -a Docker
docker version
cd "$CN_DNS_DIR"
docker compose up -d --no-build --pull never --force-recreate --wait
docker compose ps
```

Both Docker Client and Server should appear; containers should be healthy.
Recreation is needed when bind IP/environment changed; `docker compose restart`
alone does not apply a changed .env. Do this before timed demonstrations. If
the image is missing, build while internet is available, not during evaluation.

Anshuman: confirm direct backend reachability, then validate/reload nginx:

```bash
curl --max-time 5 "http://$CN_A_IP:3001/api/status"
curl --max-time 5 "http://$CN_B_IP:3002/api/status"
sudo "$CN_NGINX" -t
```

Only after the config test succeeds: if primary nginx is already running, use
`sudo "$CN_NGINX" -s reload`; if it is stopped, use `sudo "$CN_NGINX"`.
Check with `pgrep -lf nginx` and `sudo lsof -nP -iTCP:443 -sTCP:LISTEN`.
Do not launch an extra copy if the port is already occupied.

All clients: keep the original DNS backup from before the project. If there is
no backup, save the pre-project settings before the first change. If current
settings already contain old project IPs, they are not the original DHCP values.

```bash
sudo networksetup -setdnsservers Wi-Fi "$CN_PRIMARY_DNS_IP" "$CN_BACKUP_DNS_IP"
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
```

If internet access was needed to pull files but old project DNS is unreachable,
temporarily restore the known original DNS setting first. Use `Empty` only if
the original setting was DHCP; do not overwrite a known custom DNS list blindly.

For original DHCP DNS, the temporary restoration command is
`sudo networksetup -setdnsservers Wi-Fi Empty`; flush the host cache afterward.
Once the two project DNS servers are ready on their new IPs, configure both
project resolver addresses again with the command above.

<!-- pagebreak -->

## 6. Five-minute readiness check

From a remote client, preferably Bineet:

```bash
dig @"$CN_PRIMARY_DNS_IP" app.team1.test +noall +answer +time=2 +tries=1
dig @"$CN_BACKUP_DNS_IP" app.team1.test +noall +answer +time=2 +tries=1
dig @"$CN_PRIMARY_DNS_IP" app.team1.test +tcp +short +time=2 +tries=1
dig @"$CN_BACKUP_DNS_IP" app.team1.test +tcp +short +time=2 +tries=1
curl -v --max-time 15 https://app.team1.test/api/status
bash "$CN_REPO/03-Source-Code/common/lb-test.sh" 6
bash "$CN_REPO/03-Source-Code/common/cache-test.sh"
```

Both resolvers should return the NEW edge IP; TLS should verify; HTTP should
return 200; both A and B should appear; the cache test should show 304.
Run `bash "$CN_REPO/03-Source-Code/common/netinfo.sh"` on each available Mac and
save current pairwise ping results. Explain the combined DNS+edge role clearly.

## 7. If reconnecting breaks something

| Symptom | First checks / action |
|---|---|
| DNS container cannot bind old IP | Recheck this host IP, Docker .env DNS_BIND_IP, then recreate container |
| Privileged port 53 forwarding denied | Docker Desktop Settings > Advanced: allow privileged forwarding to external machines |
| Healthy container but client DNS timeout | Test internal DNS and host UDP/TCP separately; healthy is only an internal check |
| UDP host lookup fails but TCP/internal work | Investigate Docker forwarding. Recreate before the demo; if still stale, a Docker Desktop restart helped in our session. It affects all containers on that Mac; account for unrelated services |
| dig works but curl cannot resolve | Compare system resolver settings/cache and /etc/resolver overrides; flush after correcting them |
| DNS resolves old edge | Both resolvers may differ, client cache may be old, or runtime cutover was not restored |
| HTTP 502 | Check NEW backend upstream IPs, running Python servers and PF policy from edge |
| Certificate error | Check active CA/name/dates and current Mac time; do not generate a new CA just for an IP change |
| Standby cannot bind 443 | Vridhi's Supportdesk may own it; inspect docker ps before pausing only the conflicting container |
| Same Wi-Fi, hosts cannot communicate | Check subnet/interface, client isolation, Local Network permission and firewall; use a faculty-approved controlled LAN/hotspot if needed |

Normal baseline is primary edge active, both DNS servers and both backends active,
standby stopped, PF demo rules rolled back. Reapply demonstration faults only
when asked, one at a time. Re-enable the standby with its UPDATED upstreams only
for migration, then restore Supportdesk after cleanup.

Official references: [Apple DHCP renewal](https://support.apple.com/en-ae/guide/mac-help/mchlp1545/27/mac/27),
[Docker Compose startup flags](https://docs.docker.com/reference/cli/docker/compose/up).
