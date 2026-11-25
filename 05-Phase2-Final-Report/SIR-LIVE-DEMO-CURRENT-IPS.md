# Sir's live-demo scenes - choose what he asks

Current classroom setup, 25 November 2026. IPs for this guide:

| Person | IP | Normal work |
|---|---|---|
| Anshuman / YOU | 10.7.16.25 | Primary Docker DNS + original nginx edge |
| Bineet | 10.7.17.47 | Backend A :3001 + main remote test client |
| Vridhi / Pranshu hostname | 10.7.2.240 | Backup Docker DNS + Backend B :3002 |

**Do Scene 00 once before sir. Then jump directly to the scene he requests.**
You do NOT need to run all previous scenes first. People not named in a step
keep their services running and do not type that person's commands. Finish the
scene's RESTORE block before choosing another scene.

| Sir says | Go here | Who actively operates |
|---|---|---|
| Get the project ready | [Scene 00](#scene-00-one-time-ready-check) | All three |
| Show IPs / topology / ping | [Scene 01](#scene-01-lan-and-topology) | All three |
| Show DNS | [Scene 02](#scene-02-dns) | Bineet; DNS hosts show containers |
| Show HTTPS / certificate / website | [Scene 03](#scene-03-https-and-certificate) | Bineet |
| Show load balancing | [Scene 04](#scene-04-load-balancing) | Bineet |
| Show TTL / cached DNS answer | [Scene 05](#scene-05-dns-ttl) | YOU + Bineet |
| Show manual DNS cache flush | [Scene 06](#scene-06-manual-dns-cache-flush) | YOU + Bineet |
| Show backup DNS | [Scene 07](#scene-07-backup-dns-failover) | YOU + Bineet + Vridhi |
| Stop a backend / show recovery | [Scene 08](#scene-08-backend-failover-and-recovery) | Bineet + YOU |
| Show HTTP caching / ETag / 304 | [Scene 09](#scene-09-http-caching-and-304) | Bineet |
| Show packets / TCP / TLS handshake | [Scene 10](#scene-10-wireshark-packet-capture) | Bineet |
| Block direct backend access | [Scene 11](#scene-11-backend-isolation) | Selected backend owner + remote client + YOU |
| Move to standby / show edge migration | [Scene 12](#scene-12-standby-edge-migration) | All three |
| Diagnose my injected fault | [Scene 13](#scene-13-faculty-fault-diagnosis) | Bineet tests; YOU inspect edge; Vridhi helps DNS |
| Show wrong DNS / port / record / both backends down | [Scene 14](#scene-14-extra-failures-only-if-asked) | Named people in the subscene |
| Check normal state after any scene | [Scene 99](#scene-99-normal-state-check) | All three |

## Terminal names used below

| Name | Purpose |
|---|---|
| Anshuman CONTROL | All your commands; nginx/DNS run in background |
| Bineet CONTROL | Tests, DNS resolver edits, PF commands |
| Bineet A-SERVER | Python A; keep open; Control+C only when told |
| Vridhi CONTROL | DNS, standby, PF commands and client tests |
| Vridhi B-SERVER | Python B; keep open; Control+C only when told |
| Bineet CAPTURE | Extra Terminal only for Wireshark scene |

In EVERY NEW Terminal that uses `$CN_REPO` or `$CN_NGINX`, first open that
Terminal inside your own repo root and run:

```bash
source ./03-Source-Code/classroom-shell.sh
```

It prints the three current IPs and creates the shell variables. Variables in
one Terminal do not automatically appear in another. Keep CONTROL Terminals
open throughout. If a command says a script is not found, check the folder and
whether this initialization ran; do not assume a network failure.

On YOUR Mac the repo root is /Users/anshumanmehta/Documents/venv-cn-project.
Teammates use their own local repo paths. Copy only code, not prompts/output.

## Scene 00: One-time ready check

**When:** once before evaluation, or after a disconnection/service failure.
**Do not rerun it between every working scene.** With unchanged IPs, do not run
the IP-change helper, rebuild images or regenerate certificates.

### 00-A. Everyone - CONTROL Terminal

Load the shell file above. Confirm the actual Wi-Fi IP:

```bash
ipconfig getifaddr en0
```

Configure both project DNS servers for the private domain on ALL THREE Macs:

```bash
sudo mkdir -p /etc/resolver
printf 'nameserver 10.7.16.25\nnameserver 10.7.2.240\n' |
  sudo tee /etc/resolver/team1.test
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
```

**Work:** directs team1.test lookups to our DNS servers. It does not replace
normal internet DNS. For TTL/wrong-DNS tests, change THIS FILE, not only Wi-Fi's
DNS setting. The file takes precedence for the project domain.

### 00-B. YOU and Vridhi - each on their OWN Mac

```bash
docker ps --filter name=docker-dns-dns-1
```

If the DNS container is already running/healthy, leave it running. If Docker
is stopped, open Docker Desktop and wait for its engine to be ready. If the
existing container is stopped, start it:

```bash
docker start docker-dns-dns-1
```

This starts only the project DNS container, not all unrelated containers.
The new bind IPs/configurations must already be prepared, as they are now.

### 00-C. Bineet and Vridhi - check backend processes

Bineet CONTROL:

```bash
lsof -nP -iTCP:3001 -sTCP:LISTEN
curl --max-time 5 http://127.0.0.1:3001/api/status
```

Vridhi CONTROL:

```bash
lsof -nP -iTCP:3002 -sTCP:LISTEN
curl --max-time 5 http://127.0.0.1:3002/api/status
```

If already running with the correct identity, do NOT start a duplicate.
If A is stopped, Bineet opens A-SERVER, initializes the shell variables, then:

```bash
/usr/bin/python3 "$CN_REPO/03-Source-Code/backend/server.py" A 3001
```

If B is stopped, Vridhi opens B-SERVER, initializes variables, then:

```bash
python3 "$CN_REPO/03-Source-Code/backend/server.py" B 3002
```

Leave both SERVER Terminals open. The other scenes use CONTROL Terminals.

### 00-D. YOU - check nginx

```bash
sudo "$CN_NGINX" -t
pgrep -lf nginx
sudo lsof -nP -iTCP:443 -sTCP:LISTEN
```

If nginx is already running and the website works, leave it running. If stopped,
start it ONLY after the config test succeeds:

```bash
sudo "$CN_NGINX"
```

If only its configuration changed, reload the checked existing process with
`sudo "$CN_NGINX" -s reload` instead. Do not start another process on occupied 443.

### 00-E. Bineet - confirm the whole baseline

```bash
dig @10.7.16.25 app.team1.test +short +time=2 +tries=1
dig @10.7.2.240 app.team1.test +short +time=2 +tries=1
curl -v --max-time 15 https://app.team1.test/api/status
```

**Ready means:** both DNS answers are 10.7.16.25; certificate verifies; HTTP 200.
Also show A/B using Scene 04's loop once. Keep PF demo restrictions rolled back
and standby nginx stopped until their scenes. With this baseline ready, jump
directly to any requested scene below.

## Scene 01: LAN and topology

**People:** all three. **Changes to running system:** none.
**Say:** "Three physical Macs share the service roles; mine combines DNS and edge."

Show current IP table and the classroom topology diagram. Every Mac:

```bash
ipconfig getifaddr en0
ipconfig getoption en0 subnet_mask
route -n get default
ifconfig en0 | awk '/ether/{print $2}'
```

YOU:

```bash
ping -c 2 10.7.17.47
ping -c 2 10.7.2.240
```

Bineet:

```bash
ping -c 2 10.7.16.25
ping -c 2 10.7.2.240
```

Vridhi:

```bash
ping -c 2 10.7.16.25
ping -c 2 10.7.17.47
```

**Expected:** correct inventory and replies between physical Macs. If ICMP is
filtered, explain it and test a service port; ping failure alone is not proof
TCP is down. **Restore:** none.

## Scene 02: DNS

**People:** Bineet tests; YOU and Vridhi show their own DNS container if asked.
**Say:** "DNS maps the private website name to the edge; it does not forward HTTP."

YOU and Vridhi, each on your own Mac:

```bash
docker ps --filter name=docker-dns-dns-1
```

Bineet:

```bash
dig @10.7.16.25 app.team1.test
dig @10.7.2.240 app.team1.test
dig @10.7.16.25 app.team1.test +tcp +short
scutil --dns | grep -A 8 -B 1 'team1.test'
dscacheutil -q host -a name app.team1.test
```

**Expected:** both servers return 10.7.16.25, TTL 30; SERVER in full dig output
identifies the answering resolver. System lookup uses the domain resolver file.
**Restore:** none.

## Scene 03: HTTPS and certificate

**People:** Bineet only; others keep services running.

```bash
curl -v --max-time 15 https://app.team1.test/api/status
open https://app.team1.test
```

**Expected:** SSL certificate verify ok, HTTP 200, X-Edge edge-mac2, A or B.
Browser should trust the CA; do not click through a warning or use curl -k.
**Say:** "The client verifies our trusted certificate and encrypts to nginx.
TLS ends there; nginx creates a separate plain-HTTP backend connection."
**Restore:** none.

## Scene 04: Load balancing

**People:** Bineet only. **Changes:** none.

```bash
for i in {1..6}; do
  curl -sS --max-time 10 -o /dev/null -D - \
    https://app.team1.test/api/status |
    grep -iE '^(HTTP/|x-backend:|x-edge:|x-upstream-addr:)'
done
```

**Expected:** successful responses include both A and B. **Say:** "nginx selects
from the upstream pool. Clients do not need backend addresses." Exact universal
alternation is not promised with multiple workers/clients or failures.
**Restore:** none.

## Scene 05: DNS TTL

**People:** YOU change the record; Bineet observes the cache. Vridhi types nothing
and keeps her DNS/backend running. This scene is independent of Scene 06.

### 05-A. YOU - prepare the starting answer

```bash
sh "$CN_REPO/03-Source-Code/docker-dns/set-ttl-record.sh" 10.7.16.25
dig @10.7.16.25 ttl-demo.team1.test +noall +answer
```

Expected starting answer: 10.7.16.25 with TTL 30. Have step 05-C ready BEFORE
Bineet begins the loop, so the record changes during his cache window.

### 05-B. Bineet - use primary only for this isolated experiment, then sample

```bash
printf 'nameserver 10.7.16.25\n' | sudo tee /etc/resolver/team1.test
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
for i in {1..12}; do
  date '+%H:%M:%S'
  dscacheutil -q host -a name ttl-demo.team1.test
  sleep 5
done
```

The first answer should be 10.7.16.25. Tell YOU as soon as it appears.

### 05-C. YOU - immediately change the server answer

```bash
sh "$CN_REPO/03-Source-Code/docker-dns/set-ttl-record.sh" 10.7.2.240
date '+%H:%M:%S'
dig @10.7.16.25 ttl-demo.team1.test +noall +answer
```

**Expected:** direct server answer is now 10.7.2.240 while Bineet may still show
cached 10.7.16.25; later his answer changes. He must NOT flush during the loop.
**Say:** "DNS cache can retain the old mapping after the server changes. TTL
limits caching; we report actual observed timestamps, not an invented exact
30-second client switch." If it changes immediately, report that honestly.

### 05-RESTORE - YOU first, then Bineet

YOU restore the test record:

```bash
sh "$CN_REPO/03-Source-Code/docker-dns/set-ttl-record.sh" 10.7.16.25
```

Bineet restores both DNS servers and clears the experimental cache:

```bash
printf 'nameserver 10.7.16.25\nnameserver 10.7.2.240\n' |
  sudo tee /etc/resolver/team1.test
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
```

app.team1.test was not changed. Normal project remains available.

## Scene 06: Manual DNS cache flush

**People:** YOU + Bineet. Vridhi leaves services running. Can be chosen directly;
you do NOT need Scene 05 first.

### 06-A. YOU - seed a demonstration answer

```bash
sh "$CN_REPO/03-Source-Code/docker-dns/set-ttl-record.sh" 10.7.2.240
```

### 06-B. Bineet - prime the cache from primary only

```bash
printf 'nameserver 10.7.16.25\n' | sudo tee /etc/resolver/team1.test
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
date '+%H:%M:%S'
dscacheutil -q host -a name ttl-demo.team1.test
```

Expected: 10.7.2.240. Do the next two steps immediately, within 30 seconds.

### 06-C. YOU - change the server back

```bash
sh "$CN_REPO/03-Source-Code/docker-dns/set-ttl-record.sh" 10.7.16.25
date '+%H:%M:%S'
dig @10.7.16.25 ttl-demo.team1.test +noall +answer
```

### 06-D. Bineet - flush without waiting for expiry

```bash
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
date '+%H:%M:%S'
dscacheutil -q host -a name ttl-demo.team1.test
```

**Expected:** 10.7.16.25 before the old advertised TTL expires. **Say:** "Flushing
makes the next lookup obtain the changed server answer without waiting."

### 06-RESTORE - Bineet

```bash
printf 'nameserver 10.7.16.25\nnameserver 10.7.2.240\n' |
  sudo tee /etc/resolver/team1.test
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
```

The record is already restored to the edge IP by step 06-C.

## Scene 07: Backup DNS failover

**People:** YOU stop primary, Bineet tests, Vridhi shows backup logs.
**Before:** Bineet's resolver file must list BOTH servers; primary and backup
must both be running and agree on the original edge IP.

### 07-A. YOU - stop ONLY primary DNS

```bash
docker stop docker-dns-dns-1
docker ps -a --filter name=docker-dns-dns-1
```

Keep your nginx, both Python servers and Vridhi's DNS running.

### 07-B. Bineet - force a fresh system lookup

```bash
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
dscacheutil -q host -a name api.team1.test
curl --max-time 15 https://api.team1.test/api/status
```

### 07-C. Vridhi - show the query on backup

```bash
docker logs --since=2m docker-dns-dns-1 2>&1 | grep 'api.team1.test'
```

**Expected:** system lookup gets 10.7.16.25; HTTPS works; backup logs the API
query. API queries are distinct from the edge.team1.test healthcheck.
**Say:** "Backup DNS keeps name resolution working during a DNS-service failure.
It does not fix a failed application." Do not promise strict resolver-list ordering.

### 07-RESTORE - YOU

```bash
docker start docker-dns-dns-1
dig @10.7.16.25 app.team1.test +short +time=2 +tries=1
```

Wait for the correct answer before choosing another scene.

## Scene 08: Backend failover and recovery

**People:** Bineet operates A; YOU test. Vridhi keeps B running.

### 08-A. Bineet - A-SERVER Terminal

Press Control+C to stop ONLY Backend A. Do not stop Docker or the laptop.

### 08-B. YOU - CONTROL Terminal

```bash
for i in {1..6}; do
  curl -sS --max-time 10 -o /dev/null -D - \
    https://app.team1.test/api/status |
    grep -iE '^(HTTP/|x-backend:|x-upstream-addr:)'
done
```

**Expected:** 200 from B. Initial upstream header may show A then B due to retry.
**Say:** "nginx passively detects a failed request and retries a healthy target."

### 08-RESTORE - Bineet, then YOU

In A-SERVER, where variables were initialized in Scene 00:

```bash
/usr/bin/python3 "$CN_REPO/03-Source-Code/backend/server.py" A 3001
```

Leave it running. After roughly 10-15 seconds and new traffic, YOU repeat 08-B.
Expected: both A/B return. nginx is not actively polling /health; recovery is
observed on later traffic after the passive failure exclusion window.

## Scene 09: HTTP caching and 304

**People:** Bineet only. No DNS record edits or service stops.

```bash
curl -sSI https://app.team1.test/api/cached
CN_ETAG="$(curl -sSI https://app.team1.test/api/cached |
  awk -F': ' 'tolower($1)=="etag"{gsub("\r","",$2);print $2;exit}')"
curl -i -H "If-None-Match: $CN_ETAG" https://app.team1.test/api/cached
curl -sSI https://app.team1.test/api/status
```

**Expected:** cached endpoint has max-age=60 and ETag; conditional request gives
304 with no body; status endpoint is no-store. **Say:** "The ETag validates a
saved representation. DNS TTL caches addresses; HTTP caching caches content."
curl is not automatically storing a browser cache. **Restore:** none.

## Scene 10: Wireshark packet capture

**People:** Bineet uses two test Terminals; everyone else leaves services up.
Do not stop Bineet's A-SERVER by mistake.

### 10-A. Bineet CAPTURE Terminal

```bash
CN_CAPTURE="$HOME/Desktop/cn-demo-$(date +%Y%m%d-%H%M%S).pcap"
sudo tcpdump -i en0 -nn -s 0 -w "$CN_CAPTURE" \
  '(port 53 and (host 10.7.16.25 or host 10.7.2.240)) or (tcp port 443 and host 10.7.16.25)'
```

### 10-B. Bineet CONTROL Terminal

```bash
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
curl -v --tls-max 1.2 --http1.1 --max-time 15 \
  https://app.team1.test/api/status
```

### 10-C. Bineet CAPTURE Terminal

Press Control+C to stop this capture only, then:

```bash
open -a Wireshark "$CN_CAPTURE"
```

| Filter | Explain |
|---|---|
| dns | Client query, resolver address/port, answer and TTL |
| tcp.flags.syn == 1 | SYN and SYN-ACK; point to following ACK separately |
| tls.handshake | TLS 1.2 ClientHello, ServerHello, certificate |
| tls.record.content_type == 23 | Encrypted application data |

**Say:** "We force TLS 1.2 to show the certificate without decrypting the capture.
TLS 1.3 encrypts it. HTTP payload on the client-edge path is encrypted."
**Restore:** stop only this capture; no persistent service change was made.

## Scene 11: Backend isolation

**People:** select A or B below. For both, finish A including cleanup, then B.
**Before:** normal direct access works, no unfinished project PF test, current
rule file has the expected edge/backend IPs, main rules contain com.apple/*.
Do not run this scene while a backend or DNS server is deliberately stopped.

### 11-A. If sir asks to protect Backend A - Bineet is PF OWNER

Bineet CONTROL sets these variables:

```bash
CN_ANCHOR='com.apple/cn-team1-a'
CN_RULES="$CN_REPO/03-Source-Code/firewall/backend-a.pf"
CN_PF_SAVE="$HOME/cn-pf-sir-a-$(date +%Y%m%d-%H%M%S)"
```

Remote client is Vridhi. Before protection, she confirms this returns A:

```bash
curl --max-time 5 http://10.7.17.47:3001/api/status
```

### 11-B. If sir asks to protect Backend B - Vridhi is PF OWNER

Vridhi CONTROL sets these instead:

```bash
CN_ANCHOR='com.apple/cn-team1-b'
CN_RULES="$CN_REPO/03-Source-Code/firewall/backend-b.pf"
CN_PF_SAVE="$HOME/cn-pf-sir-b-$(date +%Y%m%d-%H%M%S)"
```

Remote client is Bineet. Before protection, he confirms this returns B:

```bash
curl --max-time 5 http://10.7.2.240:3002/api/status
```

### 11-C. Selected PF OWNER ONLY - backup, inspect and syntax-check

```bash
mkdir -m 700 "$CN_PF_SAVE"
cp /etc/pf.conf "$CN_PF_SAVE/pf.conf.before"
sudo pfctl -s info | tee "$CN_PF_SAVE/status.before"
sudo pfctl -sr > "$CN_PF_SAVE/main-rules.before"
cat "$CN_RULES"
sudo pfctl -a "$CN_ANCHOR" -nf "$CN_RULES"
echo "Syntax-check exit code: $?"
```

Only continue on exit code 0 and the expected baseline. The generic -f warning
and No ALTQ messages do not mean syntax failure. If current main rules differ
from the tested com.apple/* layout or a previous test is unfinished, inspect
and restore that state first. Do not overwrite other components' rules.

### 11-D. Selected PF OWNER ONLY - load and enable ONCE

```bash
sudo pfctl -a "$CN_ANCHOR" -f "$CN_RULES"
sudo pfctl -E 2>&1 | tee "$CN_PF_SAVE/enable.txt"
sudo pfctl -a "$CN_ANCHOR" -sr
```

Stop if loading fails; do not repeatedly enable PF. Keep this SAME control
Terminal and its fresh token file until cleanup.

### 11-E. YOU and the selected remote client - demonstrate restriction

For A, YOU run direct A (must succeed), Vridhi runs direct A (must fail):

```bash
curl --max-time 5 http://10.7.17.47:3001/api/status
```

For B, YOU run direct B (must succeed), Bineet runs direct B (must fail):

```bash
curl --max-time 5 http://10.7.2.240:3002/api/status
```

The selected remote client then runs (must succeed through YOU):

```bash
curl --max-time 10 https://app.team1.test/api/status
```

**Say:** "Only our edge source can reach this backend port directly. Clients can
still use the service through nginx." A returned RST can fail immediately; a
long timeout is not required to prove the block.

### 11-RESTORE. Selected PF OWNER ONLY - SAME Terminal

```bash
sudo pfctl -a "$CN_ANCHOR" -vvsr
sudo pfctl -a "$CN_ANCHOR" -F rules
CN_TOKEN="$(awk '/Token/ {print $NF; exit}' "$CN_PF_SAVE/enable.txt")"
```

If CN_TOKEN is nonempty, release THAT reference:

```bash
sudo pfctl -X "$CN_TOKEN"
sudo pfctl -s info
```

If the token is missing, do not guess/reuse an old token. Do not flush main
rules or use blanket pfctl -d. Remote client repeats its direct request; it
must work again. For the other backend, return to 11-A or 11-B and do a fresh
separate test. Finish all PF cleanup BEFORE Scene 12.

## Scene 12: Standby edge migration

**People:** all three. **Before:** Scene 00 baseline, PF restrictions rolled
back, standby config has current A/B IPs, local tls.pem is present. Do NOT
stop the original nginx during this scene.

### 12-A. Vridhi CONTROL - inspect config and port ownership

```bash
grep 'server .*300' "$HOME/cn-standby/nginx.conf"
lsof -nP -iTCP:443 -sTCP:LISTEN
docker ps --format 'table {{.Names}}\t{{.Ports}}'
```

Upstreams must be 10.7.17.47:3001 and 10.7.2.240:3002. If stale, use the
classroom helper before presenting; do not start a stale standby.

Only if supportdesk_nginx is the conflicting running container and may be
paused, Vridhi records that fact and stops ONLY that container:

```bash
CN_RESTORE_SUPPORTDESK=yes
docker stop supportdesk_nginx
```

If it was already stopped and port 443 is free, she instead sets:

```bash
CN_RESTORE_SUPPORTDESK=no
```

Do not execute both choices. If another unknown service owns 443, inspect it
instead of stopping unrelated services blindly.

### 12-B. Vridhi CONTROL - check and start standalone nginx

```bash
sudo "$CN_NGINX" -p "$HOME/cn-standby/" -c nginx.conf -t
```

Only after the test succeeds (and only if standby is not already running):

```bash
sudo "$CN_NGINX" -p "$HOME/cn-standby/" -c nginx.conf
```

### 12-C. YOU - preparation test, before DNS change

```bash
curl -v --resolve app.team1.test:443:10.7.2.240 \
  https://app.team1.test/api/status
```

Expected: verified TLS, 200, X-Edge edge-standby. This is a DNS-bypass preparation
check, not the actual DNS migration proof.

### 12-D. Bineet CONTROL - start normal requests

```bash
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
for i in {1..20}; do
  date '+%H:%M:%S'
  curl -sS --max-time 10 -o /dev/null -D - \
    -w 'Connected-IP: %{remote_ip}\n' \
    https://app.team1.test/api/status |
    grep -iE '^(HTTP/|x-edge:|x-backend:|Connected-IP:)'
  sleep 5
done
```

### 12-E. YOU AND Vridhi - after his first original-edge answer

Each runs on their OWN Mac:

```bash
sh "$CN_REPO/03-Source-Code/docker-dns/set-edge-record.sh" 10.7.2.240
```

YOU check BOTH servers:

```bash
dig @10.7.16.25 app.team1.test +noall +answer
dig @10.7.2.240 app.team1.test +noall +answer
```

Expected: both now publish 10.7.2.240. Bineet's loop moves from edge-mac2 /
10.7.16.25 to edge-standby / 10.7.2.240 while requests succeed. Keep both
edges running; do not flush his cache during the observation.
**Say:** "Same name, same HTTPS port and trusted identity; DNS moves clients
from the old edge to standby. This is manual cutover, not automatic detection."

### 12-RESTORE-1. YOU AND Vridhi - restore original DNS

Each runs on their OWN Mac:

```bash
sh "$CN_REPO/03-Source-Code/docker-dns/set-edge-record.sh" 10.7.16.25
```

YOU check both direct answers again. All clients then flush:

```bash
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
```

Bineet verifies return to original edge:

```bash
curl -sS --max-time 10 -o /dev/null -D - \
  -w 'Connected-IP: %{remote_ip}\n' \
  https://app.team1.test/api/status |
  grep -iE '^(HTTP/|x-edge:|Connected-IP:)'
```

Expected: 200, edge-mac2, 10.7.16.25.

### 12-RESTORE-2. Vridhi - ONLY after the return check succeeds

```bash
sudo "$CN_NGINX" -p "$HOME/cn-standby/" -c nginx.conf -s stop
sleep 2
if [ "$CN_RESTORE_SUPPORTDESK" = yes ]; then
  docker start supportdesk_nginx
fi
```

Use the SAME Vridhi CONTROL Terminal so the restore variable is retained.
Backup DNS and Backend B stay running. Never restart Supportdesk before the
standby releases its port. The original edge remains the normal single point
of failure; extra health detection/steering is needed for automatic edge HA.

## Scene 13: Faculty fault diagnosis

**People:** Bineet tests the client path; YOU inspect nginx/upstreams; Vridhi
checks backup DNS if needed. Do not inject an unrelated fault yourself.

Bineet:

```bash
cat /etc/resolver/team1.test
scutil --dns | grep -A 8 -B 1 'team1.test'
dig @10.7.16.25 app.team1.test +time=2 +tries=1
dig @10.7.2.240 app.team1.test +time=2 +tries=1
dscacheutil -q host -a name app.team1.test
nc -vz -G 3 10.7.16.25 443
curl -v --max-time 10 https://app.team1.test/api/status
```

YOU:

```bash
sudo "$CN_NGINX" -t
curl --max-time 5 http://10.7.17.47:3001/health
curl --max-time 5 http://10.7.2.240:3002/health
sudo tail -n 20 "$CN_BREW_PREFIX/var/log/nginx/error.log"
```

DNS hosts, each on their own Mac, if resolution is the failing layer:

```bash
docker ps -a --filter name=docker-dns-dns-1
docker logs --tail=20 docker-dns-dns-1
docker exec docker-dns-dns-1 dig @127.0.0.1 app.team1.test +short
```

**Say:** "We check DNS, then TCP port, then TLS verification, then HTTP/backends."
Wrong DNS answer, connection refusal, certificate error and 502 are different
layers. Check the intended selected edge, not always the original IP, if sir
has intentionally migrated it. State the actual finding and retest after the
smallest correction. Record the faculty fault/outcome after evaluation.

## Scene 14: Extra failures only if asked

### 14-A. Wrong DNS server - Bineet only changes his project resolver

```bash
printf 'nameserver 10.255.255.254\n' | sudo tee /etc/resolver/team1.test
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
dscacheutil -q host -a name app.team1.test
curl --max-time 8 https://app.team1.test/api/status
ping -c 2 10.7.16.25
```

Expected: name lookup fails while IP reachability may still work. RESTORE on
Bineet immediately:

```bash
printf 'nameserver 10.7.16.25\nnameserver 10.7.2.240\n' |
  sudo tee /etc/resolver/team1.test
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
```

### 14-B. Wrong DNS record - YOU change primary; Bineet isolates his lookup

Bineet:

```bash
printf 'nameserver 10.7.16.25\n' | sudo tee /etc/resolver/team1.test
```

YOU:

```bash
sh "$CN_REPO/03-Source-Code/docker-dns/set-edge-record.sh" 10.7.17.47
dig @10.7.16.25 app.team1.test +noall +answer
```

Bineet:

```bash
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
curl -v --max-time 8 https://app.team1.test/api/status
```

Expected: DNS answers successfully but sends the client to the wrong host. If
A has no service on 443, the connection fails; another service there could give
a TLS/HTTP error instead. Explain the actual result.

RESTORE on YOU:

```bash
sh "$CN_REPO/03-Source-Code/docker-dns/set-edge-record.sh" 10.7.16.25
```

Bineet restores the two resolver entries and flushes using 14-A's RESTORE block.

### 14-C. Both backends stopped

Bineet presses Control+C in A-SERVER; Vridhi presses Control+C in B-SERVER.
Keep DNS/nginx up. YOU:

```bash
dig @10.7.16.25 app.team1.test +short
curl -v --max-time 10 https://app.team1.test/api/status
```

Expected: DNS and TLS may succeed, then nginx returns 502 with no reachable
upstream. RESTORE A and B in their SERVER Terminals with Scene 00-C's start
commands, leave them running, then confirm A/B with Scene 04.

### 14-D. Wrong port - Bineet

```bash
nc -vz -G 3 10.7.16.25 443
curl -v --connect-timeout 3 --max-time 5 \
  https://app.team1.test:8443/api/status
```

Expected: normal 443 works, unused 8443 fails. Different ports identify different
services on the same host. RESTORE: none; it was only a client request.

## Scene 99: Normal state check

Use after each scene, or if sir says "now show it working normally."

- Primary/backup DNS are active and agree on 10.7.16.25.
- A and B are running. Original nginx is active.
- Each client /etc/resolver/team1.test lists BOTH current DNS servers.
- Test PF rules were cleared and each test's enable token released using
  Scene 11's SAME-Terminal cleanup; never guess/reuse a token.
- app/api and ttl-demo are restored to the normal edge IP.
- Standby is stopped after migration rollback; Supportdesk is restored only
  if it was paused. Backup DNS and B stay running throughout cleanup.

Bineet runs:

```bash
dig @10.7.16.25 app.team1.test +short +time=2 +tries=1
dig @10.7.2.240 app.team1.test +short +time=2 +tries=1
curl -sS --max-time 10 -o /dev/null -D - \
  https://app.team1.test/api/status |
  grep -iE '^(HTTP/|x-edge:|x-backend:|x-upstream-addr:)'
```

Expected: both answers 10.7.16.25, HTTP 200, edge-mac2. Use Scene 04 if asked
to show both backend identities again. Keep current resolver files while using
the project. If IPs change later, update those files too; the classroom helper
does not write root-owned /etc files.
