# Wireshark demo for Backend B and the test client

**Presenter:** Vridhi (Pranshu-named Mac). Show one request's DNS lookup,
TCP connection, TLS handshake and encrypted application data. Keep the normal
edge, both backends and both DNS services running during this demonstration.
Use current LAN addresses; screenshots from Phase 1 have historical addresses.

## 1. Load settings in each Terminal

Open Terminal inside this repository. This block also works in a checkout
without `classroom-shell.sh`. Confirm `03-Source-Code/team.env` contains the
current team addresses before using it.

```bash
export CN_REPO="$(git rev-parse --show-toplevel)"
source "$CN_REPO/03-Source-Code/team.env"
export CN_EDGE_IP="$MAC2_IP"
export CN_PRIMARY_DNS_IP="$MAC1_IP"
export CN_BACKUP_DNS_IP="$MAC4_IP"
export CN_IFACE="$(route -n get default | awk '/interface:/{print $2; exit}')"
ipconfig getifaddr "$CN_IFACE"
echo "Edge=$CN_EDGE_IP Primary=$CN_PRIMARY_DNS_IP Backup=$CN_BACKUP_DNS_IP"
```

`source` loads configuration; `export` prepares variables for later commands.
`ipconfig` shows this Mac's actual IP. Compare printed addresses with the live
three-Mac inventory. Start with working, verified HTTPS:

```bash
curl -v --max-time 15 https://app.team1.test/api/status
```

Expected: verified certificate, HTTP 200 and backend A/B JSON. Resolve failures
before attempting a successful-flow demonstration. Do not add `-k`.

## 2. Capture in Terminal 1

```bash
mkdir -p "$CN_REPO/04-Evidence/Phase2"
CN_PCAP="$CN_REPO/04-Evidence/Phase2/viva-$(date +%Y%m%d-%H%M%S).pcap"
sudo tcpdump -i "$CN_IFACE" -nn -s 0 -w "$CN_PCAP" \
  'port 53 or tcp port 443'
```

`mkdir` prepares the evidence directory. `tcpdump` captures on the active
interface; `-nn` preserves numeric addresses/ports, `-s 0` captures full packets,
and `-w` writes the file. Leave this Terminal running after `listening on ...`.

## 3. Generate traffic in Terminal 2

Load the settings block from step 1 in this new Terminal too. Temporarily
select the remote primary DNS so the lookup crosses Wi-Fi rather than staying
inside this Mac's backup resolver. This assumes the normal two-team-resolver
configuration; restore both addresses immediately afterward.

```bash
sudo networksetup -setdnsservers Wi-Fi "$CN_PRIMARY_DNS_IP"
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
curl -v --tls-max 1.2 --http1.1 --max-time 15 \
  https://app.team1.test/api/status
```

The first three commands select the resolver and clear cached answers. The curl
flags show connection details, permit TLS up to 1.2, and request HTTP/1.1. TLS 1.2
makes the certificate visible in the capture; certificate validation stays on.
Keep the successful curl output for the instructor.

Restore the client resolver list even if the request failed:

```bash
sudo networksetup -setdnsservers Wi-Fi \
  "$CN_PRIMARY_DNS_IP" "$CN_BACKUP_DNS_IP"
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
```

Use the actual network service name if it is not `Wi-Fi`.

## 4. Open the capture

In **Terminal 1**, press **Control+C** to stop this capture, then run:

```bash
open -a Wireshark "$CN_PCAP"
```

Wireshark's top pane lists packets; the middle pane expands protocol details;
the bottom pane shows bytes. Enter the following filters in the **Wireshark
display-filter bar**, not Terminal. Focus on your client and intended edge.

| Show | Display filter and action | What to say in Hinglish |
|---|---|---|
| DNS | `dns`; select the A query for app.team1.test and matching response; expand Queries and Answers | Sir, client domain ka IPv4 poochta hai. Response mein edge IP aur TTL mila. DNS backend select nahi karta. |
| TCP | `tcp.port == 443 && tcp.flags.syn == 1`; select the client-to-edge SYN, then right-click > Conversation Filter > TCP | Sir, SYN, SYN-ACK, ACK se TCP connection establish hua. Client ephemeral port use karta hai; edge port 443 hai. |
| TLS | `tls.handshake`; expand ClientHello, ServerHello and Certificate | Sir, client TLS options offer karta hai. Server parameters aur certificate bhejta hai; client CA, domain aur dates verify karta hai. |
| Encrypted data | `tls.record.content_type == 23`; select Application Data for the same connection | Sir, HTTP traffic TLS mein encrypted hai, isliye plain JSON readable nahi hai. |

The SYN-only filter hides the third ACK: use the TCP conversation filter to show
all three packets. In TCP details, explain that ACK is the next expected **byte**,
not a packet count. Certificate details may share a packet with ServerHello.
Showing the certificate alone does not prove verification succeeded: show curl's
successful verification and HTTP status too.

## 5. Explain the backend boundary and save evidence

Say: **Sir, client se nginx tak HTTPS hai. nginx TLS terminate karke Backend A
ya mere Backend B port 3002 ko separate plain HTTP request bhejta hai.**

This client-side capture does not automatically reveal both backend connections.
HTTP headers and JSON are readable in curl because it is a TLS endpoint. TLS 1.3
encrypts the certificate; this demonstration deliberately uses TLS 1.2.

Save screenshots of DNS answer/TTL, the three-way handshake and ports, certificate
details, encrypted application data, and successful curl output. The saved pcap
is evidence of the observed run, not a claim that every later session works.

If DNS is missing, check cache, resolver selection and interface. If SYN is
missing, start capture before a new curl request. If HTTPS fails, diagnose
DNS, then TCP, then TLS, then HTTP. See [the full command book](LIVE-COMMANDS.md)
for other members' commands, controlled failures and rollback.
