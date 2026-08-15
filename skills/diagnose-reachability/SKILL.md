---
name: diagnose-reachability
description: Debug "the site works from region/network A but times out or fails from region/network B" (e.g. ERR_TIMED_OUT for some users, not others). Layered elimination → bridge fix → durable CDN fix. Use for geographic/network-specific reachability, latency, or TLS-handshake stalls.
---

# Diagnose Reachability ("works here, fails there")

For the symptom: **some users can reach the site, others (a region, a carrier, a network) get
`ERR_TIMED_OUT` / hang / TLS stall** — while it serves fine at the origin. Diagnose by *eliminating
layers with checks, not guesses* (it's easy to be confidently wrong — verify each). Then apply a
tactical bridge fix and a durable CDN fix.

## First: read the error type — it narrows the cause immediately
- **`ERR_TIMED_OUT`** → TCP/handshake packets dropped. DNS resolved but the connection can't complete.
  Suspect firewall/SG dropping the source, or a **path-MTU blackhole** (see step 6). NOT a cert issue.
- **`ERR_CERT_*` / `ERR_SSL_*`** → certificate/trust problem (jump to step 5).
- **`DNS_PROBE_*` / `NXDOMAIN`** → DNS resolution problem (step 3).
- **Connection refused** (fast) → nothing listening / port closed (vs. *timeout* = packets dropped).

## Layered elimination (top-down; stop when you find it)

1. **Origin is actually up?** From the box: `curl -s -o /dev/null -w "%{http_code}" https://SITE
   --resolve SITE:443:127.0.0.1`. If not 200, the app/proxy is the problem — stop here.

2. **Host firewall** (ufw/iptables/security software): `sudo ufw status`; `sudo iptables -L INPUT -n`.
   Is 80/443 open to Anywhere, or scoped to specific source IPs? A rule scoped to *your* IP = "works
   for me, times out for everyone else."

3. **DNS** — resolves to the right IP, everywhere?
   - `dig +short A SITE @8.8.8.8` and `@1.1.1.1` — same correct IP?
   - **Geo/latency routing** returning a different (dead) IP for the affected region:
     `dig @8.8.8.8 +short A SITE +subnet=<affected-region-CIDR>/24` vs a known-good region's CIDR.
   - **IPv6**: `dig +short AAAA SITE` — a bad `AAAA` makes mobile clients (Happy Eyeballs) try a
     broken IPv6 path first → timeout. No AAAA / correct AAAA = fine.

4. **Cloud firewall (Security Group)** — inbound 80/443 must be `0.0.0.0/0` for a public site
   (launch-wizard SGs notoriously scope HTTP/HTTPS to "My IP"):
   `aws ec2 describe-security-groups --group-ids <sg> --query "SecurityGroups[0].IpPermissions[?FromPort==\`443\`].IpRanges[].CidrIp"`

5. **Certificate** (only if a cert error, or to rule it out) — a *timeout* is NOT a cert problem, but
   check chain validity against the **real public CA bundle**, not the local store (which may hold a
   private/MITM root): `curl --cacert <(curl -s https://curl.se/ca/cacert.pem) -sS -o /dev/null -w
   "%{http_code}" https://SITE --resolve SITE:443:<ORIGIN_IP>`. 200 = publicly trusted. Note: an
   over-long chain makes the TLS handshake bigger → worsens step 6.

6. **Network ACL (subnet, AWS)** — default is allow-all; a DENY covering the affected CIDRs would do it:
   `aws ec2 describe-network-acls --filters Name=association.subnet-id,Values=<subnet>
   --query "NetworkAcls[0].Entries[?Egress==\`false\`]"`.

7. **Path-MTU blackhole** (the usual culprit when 1–6 are all clean and it's *mobile/regional*).
   Many mobile/CGNAT paths have MTU < 1500 **and filter ICMP "frag needed"**, so the large TLS
   handshake packets are silently dropped → the handshake stalls → `ERR_TIMED_OUT`. Full-MTU paths
   (typical wired/US) are unaffected. Server signals: no MSS clamp
   (`iptables -t mangle -S | grep TCPMSS`), `sysctl net.ipv4.tcp_mtu_probing` = 0. Confirmation (needs
   an affected-side test): `curl -v` hangs *at the TLS handshake*, or `ping -M do -s 1472 <IP>` fails
   while `-s 1372` succeeds.

## Bridge fix (fast, reversible, server-side) — for the MTU blackhole
```bash
sudo sysctl -w net.ipv4.tcp_mtu_probing=1        # persist in /etc/sysctl.d/
# clamp advertised MSS so TLS packets survive a small path MTU (works for Docker/NAT hosts too):
sudo iptables -t mangle -A FORWARD    -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --set-mss 1400
sudo iptables -t mangle -A POSTROUTING -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --set-mss 1400
```
Persist across reboot (a systemd oneshot that re-adds the rules after docker, or ufw before.rules).
The affected user then just **refreshes in the browser** — that's the test (no CLI needed on their end).

## Durable fix — put a CDN in front (edge TLS near the affected users)
A single origin far from the affected market keeps hitting MTU/routing fragility. A CDN (Cloudflare
free, or CloudFront) terminates TLS at an edge PoP near them → eliminates the MTU blackhole *and* the
long-haul routing problem, and cuts latency.

**Cutover footguns (each of these has caused an outage):**
- **Verify DNS-record parity BEFORE the nameserver cutover.** Enumerate ALL records at the origin DNS
  (not just the ones you remember) and confirm the CDN has every live one. A missing record = that
  subdomain is **down globally** the moment NS points to the CDN. (Real incident: `medecin`/`pharma`
  were missing from the CDN import → both portals went down at cutover.)
- **Records must be PROXIED (not DNS-only) to get the fix.** A grey/DNS-only record resolves straight
  to the origin → the CDN isn't in the path → the problem persists. Proxied records return the CDN's
  edge IPs; verify: `dig @<cdn-nameserver> +short A SITE` shows edge IPs (e.g. 104.x/172.67.x), and
  `curl` headers show `server: cloudflare` + `cf-ray`.
- **WebRTC / media (LiveKit, SFUs, UDP) must stay DNS-only (direct).** CDNs can't carry the UDP media;
  proxying that hostname breaks video/audio.
- **CDN SSL mode = "Full (strict)"**, never "Flexible" — an origin that force-redirects HTTP→HTTPS
  loops forever under Flexible.
- **DNSSEC must be OFF** before a nameserver change, or resolution breaks. `whois DOMAIN | grep -i dnssec`.
- **Keep the origin's cert renewing** (Full-strict still validates the origin cert).
- **Drop dead records** (a subdomain pointing at a terminated box) rather than carrying them over.
- Nameservers are changed **at the registrar** (`whois DOMAIN` → Registrar), not in the DNS hosted zone.

## Verify it's actually live through the CDN
```
dig @<cdn-ns> +short NS DOMAIN              # delegation flipped to the CDN
dig @<cdn-ns> +short A SITE                 # proxied => edge IPs; media host => origin IP
curl -sS -D - https://SITE --resolve SITE:443:<edge-ip> | grep -iE "HTTP/|server:|cf-ray"
```
Healthy = edge IPs for proxied hosts, `server: cloudflare` + `cf-ray`, media host still direct, and
the affected user's browser refresh loads.

## Rollback
Switch the registrar nameservers back to the originals — the old DNS zone is untouched, so it reverts
instantly (the server-side bridge fix stays active).
