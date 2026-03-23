# Blackbox Exporter README: Ping, Port, and URL Monitoring

This README explains how to use **Prometheus Blackbox Exporter** for three common monitor types:

- **ICMP / Ping monitoring** (`icmp` module)
- **TCP / Port monitoring** (`tcp` module, commonly named `tcp_connect`)
- **HTTP/HTTPS / URL monitoring** (`http` module, commonly named `http_2xx`)

It also compares them side by side, explains what each one can and cannot prove, and lists the main configuration options you can use.

---

## 1) What Blackbox Exporter can probe

Blackbox Exporter supports blackbox probing of endpoints over:

- HTTP / HTTPS
- TCP
- ICMP
- DNS
- gRPC
- Unix sockets

For your local Mac + Docker use case, the three most practical starting points are:

1. `icmp` for reachability like ping
2. `tcp` for port/service reachability
3. `http` for URL and web/API checks

---

## 2) Quick mental model

### `icmp`
Asks: **"Can I reach this host at the network layer?"**

This is closest to traditional `ping`.

### `tcp_connect`
Asks: **"Can I open a TCP connection to this host:port?"**

This is ideal for checking if a service port is reachable, such as 22, 80, 443, 5432, 3306, etc.

### `http_2xx`
Asks: **"Can I successfully make an HTTP request to this URL, and does the response match my expectations?"**

This is ideal for websites, APIs, health endpoints, redirects, TLS checks, auth checks, and response validation.

---

## 3) Comparison table

| Module / Prober | Typical module name | Example target | What it proves | What it does **not** prove | Best for | Common result to watch |
|---|---|---|---|---|---|---|
| ICMP | `icmp` | `8.8.8.8` or `google.com` | Host is reachable via ICMP echo | App/service may still be down even if ping works | Basic network reachability | `probe_success` |
| TCP | `tcp_connect` | `google.com:443` | Host and port accepted a TCP connection | App logic may still fail after connect | Port/service reachability | `probe_success`, connect timing |
| HTTP | `http_2xx` | `https://example.com/health` | URL responded over HTTP/HTTPS and matched expected conditions | Deep business logic may still be broken unless you validate body/headers/status | Websites, APIs, health endpoints | `probe_success`, status code, duration |

---

## 4) What each module can probe

### ICMP (`icmp`)
Can probe:
- Routers
- Servers
- Load balancer IPs or hostnames
- Public IPs / hostnames
- Internal network hosts reachable from the exporter

Good questions it answers:
- Is the machine reachable at all?
- Did routing change?
- Is a host reachable within a certain TTL / hop count?

Not good for:
- Verifying an application is healthy
- Verifying a specific service is listening
- Verifying a URL, TLS certificate, redirect, or API response

### TCP (`tcp` / `tcp_connect`)
Can probe:
- `host:22` for SSH reachability
- `host:80` / `:443` for HTTP service reachability
- `host:5432` for PostgreSQL listener reachability
- `host:3306` for MySQL listener reachability
- `host:6379` for Redis listener reachability
- `host:25` / `:587` / `:993` / `:110` etc. for mail service reachability
- Any custom application port

Good questions it answers:
- Is the port open and reachable?
- Can I perform a banner or protocol handshake check?
- Can I start TLS after connecting?

Not good for:
- Verifying the full web/app behavior unless you script query/response carefully
- Checking HTTP status codes directly
- Checking redirects or body content like a browser/client

### HTTP (`http` / `http_2xx`)
Can probe:
- Websites
- REST APIs
- Health endpoints (`/health`, `/ready`, `/live`)
- Authenticated endpoints
- Redirecting endpoints
- TLS endpoints
- Endpoints requiring specific headers
- Responses whose body, JSON, or headers must match patterns

Good questions it answers:
- Did the URL return an acceptable status code?
- Was TLS present or absent as expected?
- Did the body include the expected content?
- Did the response header match policy?
- Did JSON satisfy a CEL expression?

Not good for:
- Low-level network reachability when ICMP/TCP are more direct
- Non-HTTP protocols

---

## 5) Example `blackbox.yml`

```yaml
modules:
  icmp:
    prober: icmp
    timeout: 5s
    icmp:
      preferred_ip_protocol: ip4

  tcp_connect:
    prober: tcp
    timeout: 5s
    tcp:
      preferred_ip_protocol: ip4

  http_2xx:
    prober: http
    timeout: 5s
    http:
      method: GET
      preferred_ip_protocol: ip4
      valid_status_codes: []
      follow_redirects: true
```

Notes:
- `valid_status_codes: []` means the default HTTP success behavior is used, which is effectively 2xx.
- `preferred_ip_protocol: ip4` is often a practical choice on local Docker/Mac setups.

---

## 6) Example probe calls

### ICMP / ping
```bash
curl "http://localhost:9115/probe?target=8.8.8.8&module=icmp"
```

### TCP / port
```bash
curl "http://localhost:9115/probe?target=google.com:443&module=tcp_connect"
```

### HTTP / URL
```bash
curl "http://localhost:9115/probe?target=https://example.com&module=http_2xx"
```

### Debug version
```bash
curl "http://localhost:9115/probe?target=https://example.com&module=http_2xx&debug=true"
```

---

## 7) How to read the result

The most important metric is:

- `probe_success 1` = success
- `probe_success 0` = failure

Then look at supporting metrics such as:

### ICMP-related
- `probe_duration_seconds`

### TCP-related
- `probe_duration_seconds`

### HTTP-related
- `probe_http_status_code`
- `probe_duration_seconds`
- redirect / TLS / timing-related metrics depending on the probe

---

## 8) Exhaustive options you should know

Below are the main configurable options for the three modules you asked about, plus module-level controls that affect all of them.

---

## 9) Module-level options (apply to every module)

| Option | Applies to | Meaning | Why you would use it |
|---|---|---|---|
| `prober` | all modules | Selects the protocol engine: `http`, `tcp`, `dns`, `icmp`, `grpc`, `unix` | Tells Blackbox what kind of probe to run |
| `timeout` | all modules | Maximum time the probe can take before failing | Prevents slow targets from hanging too long |

Example:

```yaml
modules:
  http_2xx:
    prober: http
    timeout: 5s
```

---

## 10) ICMP options (`icmp_probe`)

These are the key options for the ICMP module.

| Option | Type | Meaning | Practical explanation |
|---|---|---|---|
| `preferred_ip_protocol` | string (`ip4` or `ip6`) | Which IP family Blackbox should prefer | Use `ip4` if IPv6 is unreliable or unavailable in your Docker/Mac path |
| `ip_protocol_fallback` | boolean | If preferred family fails or is unavailable, allow fallback to the other family | Keep `true` if you want resiliency; set `false` if you want strict IPv4-only or IPv6-only checks |
| `source_ip_address` | string | Source IP address to send probes from | Useful on multi-homed systems or when testing routing from a specific interface/IP |
| `dont_fragment` | boolean | Sets the DF bit in IPv4 packets | Useful for MTU and fragmentation troubleshooting; requires raw socket capability |
| `payload_size` | int | Payload size of the ICMP echo | Lets you test bigger packets rather than default tiny ping payloads |
| `ttl` | int | Time-to-live for outbound packets | Lets you restrict max hops or detect routing path changes |

### Example ICMP module

```yaml
modules:
  icmp:
    prober: icmp
    timeout: 5s
    icmp:
      preferred_ip_protocol: ip4
      ip_protocol_fallback: false
      payload_size: 56
      ttl: 64
```

### When to use each ICMP option

- Use `preferred_ip_protocol: ip4` when local Docker networking behaves better on IPv4.
- Use `ip_protocol_fallback: false` when you want strict verification that IPv4 specifically works.
- Use `dont_fragment: true` when testing path MTU behavior.
- Use `payload_size` when small pings succeed but large packets may reveal a network issue.
- Use `ttl` when you want to know whether a route length changed.

### Important ICMP caveat on Mac + Docker

ICMP often needs raw socket privileges. Even with container capabilities, ICMP can be less predictable in Docker Desktop environments than TCP or HTTP checks.

---

## 11) TCP options (`tcp_probe`)

These are the key options for TCP probes.

| Option | Type | Meaning | Practical explanation |
|---|---|---|---|
| `preferred_ip_protocol` | string (`ip4` or `ip6`) | Preferred IP family for the TCP connection | Often set to `ip4` locally |
| `ip_protocol_fallback` | boolean | Whether fallback to the other IP family is allowed | Set to `false` for strict family checks |
| `source_ip_address` | string | Source IP to originate the connection from | Helpful on machines with multiple interfaces/IPs |
| `query_response` | list | Script-like send/expect sequence for the TCP session | Lets you go beyond simple connect and validate protocol/banner text |
| `tls` | boolean | Start the TCP connection using TLS immediately | Useful for services expecting TLS from the first byte |
| `tls_config` | object | TLS options for TCP probes | Used when you need cert validation behavior, SNI, or custom CA/cert settings |

### Understanding `query_response`

`query_response` is the most powerful part of TCP probes.

It lets you define a sequence such as:
1. Connect to the TCP port
2. Expect a banner or greeting
3. Send a command or payload
4. Expect a matching response
5. Optionally upgrade with STARTTLS

Inside each step you can use:

| Field | Meaning |
|---|---|
| `expect` | Regex that must match the server response |
| `expect_bytes` | Exact byte-for-byte response match |
| `labels` | Extra labels exported via `probe_expect_info` |
| `send` | Text to send to the server |
| `starttls` | Upgrade the connection to TLS during the session |

### Example: simple TCP connect only

```yaml
modules:
  tcp_connect:
    prober: tcp
    timeout: 5s
    tcp:
      preferred_ip_protocol: ip4
```

### Example: check an SSH banner

```yaml
modules:
  ssh_banner:
    prober: tcp
    timeout: 5s
    tcp:
      preferred_ip_protocol: ip4
      query_response:
        - expect: "^SSH-2.0-"
```

### Example: SMTP-style greeting and command

```yaml
modules:
  smtp_banner:
    prober: tcp
    timeout: 5s
    tcp:
      preferred_ip_protocol: ip4
      query_response:
        - expect: "^220"
        - send: "EHLO example.com\r\n"
        - expect: "^250"
```

### When to use TCP instead of HTTP

Use TCP when:
- you only care that a service port is reachable
- the protocol is not HTTP
- you want to verify a simple banner or text handshake
- you want to test SMTP, SSH, custom line protocols, or DB listeners at a basic level

---

## 12) HTTP options (`http_probe`)

HTTP has the richest set of options because it can validate behavior, not just reachability.

### Core HTTP request/response options

| Option | Type | Meaning | Practical explanation |
|---|---|---|---|
| `valid_status_codes` | list of ints | Accepted HTTP status codes | Leave empty/default for normal 2xx success, or list exact accepted codes like `[200, 204]` |
| `valid_http_versions` | list of strings | Accepted HTTP versions | Useful if you want to require or allow only specific protocol versions |
| `method` | string | HTTP method to use | `GET`, `POST`, `HEAD`, etc. |
| `headers` | map | Static headers to send | Add `Host`, `User-Agent`, auth headers, etc. |
| `http_headers` | structured map | Injectable headers from values, secrets, or files | More advanced secure/header injection model |
| `body_size_limit` | size | Maximum body size to process | Prevents reading huge responses |
| `compression` | string | Decompression algorithm to use | Useful if testing gzip/br/deflate handling |
| `follow_redirects` | boolean | Whether redirects are followed | Turn off if you want to detect unexpected redirects |
| `fail_if_ssl` | boolean | Fail if SSL/TLS is present | Useful for plain HTTP policies |
| `fail_if_not_ssl` | boolean | Fail if SSL/TLS is absent | Useful for enforcing HTTPS |

### HTTP content validation options

| Option | Type | Meaning | Practical explanation |
|---|---|---|---|
| `fail_if_body_json_matches_cel` | string | Fail if JSON body matches the CEL expression, or body is not JSON | Good for “bad state” rules in API JSON |
| `fail_if_body_json_not_matches_cel` | string | Fail unless JSON body matches the CEL expression, or body is not JSON | Good for requiring fields/values in JSON responses |
| `fail_if_body_matches_regexp` | list of regex | Fail if body contains any forbidden pattern | Useful for detecting error pages, stack traces, maintenance messages |
| `fail_if_body_not_matches_regexp` | list of regex | Fail unless body contains expected text | Good for health endpoints or key content validation |
| `fail_if_header_matches` | list | Fail if a response header matches a regex | Enforce header policies |
| `fail_if_header_not_matches` | list | Fail unless a response header matches a regex | Require a header/value like cache, content-type, security headers |

### HTTP auth / proxy / TLS options

| Option | Type | Meaning | Practical explanation |
|---|---|---|---|
| `tls_config` | object | TLS settings for the HTTP probe | Lets you control cert verification, CA, client certs, SNI, etc. |
| `basic_auth` | object | Username/password basic auth | For endpoints protected by HTTP Basic Auth |
| `authorization` | object | Authorization header configuration | Often used for Bearer tokens |
| `proxy_url` | string | Explicit proxy URL | Route the request through a proxy |
| `no_proxy` | string | Comma-separated exclusions from proxying | Skip proxy for certain hosts/IPs/CIDRs |
| `proxy_from_environment` | boolean | Use proxy env vars | Honor `HTTP_PROXY`, `HTTPS_PROXY`, and `NO_PROXY` from environment |

### What `headers` vs `http_headers` means

- `headers` is the simpler option for fixed key/value headers.
- `http_headers` is the more advanced option that supports values, secrets, and files.

### Example: basic URL monitoring

```yaml
modules:
  http_2xx:
    prober: http
    timeout: 5s
    http:
      method: GET
      preferred_ip_protocol: ip4
      valid_status_codes: [200]
      follow_redirects: true
      fail_if_not_ssl: true
```

### Example: API JSON validation

```yaml
modules:
  api_health:
    prober: http
    timeout: 5s
    http:
      method: GET
      preferred_ip_protocol: ip4
      valid_status_codes: [200]
      fail_if_body_json_not_matches_cel: 'body.status == "ok"'
```

### Example: body text validation

```yaml
modules:
  website_content:
    prober: http
    timeout: 5s
    http:
      method: GET
      preferred_ip_protocol: ip4
      fail_if_body_not_matches_regexp:
        - "Welcome"
      fail_if_body_matches_regexp:
        - "Internal Server Error"
        - "Maintenance"
```

---

## 13) Address family behavior: IPv4 vs IPv6

Blackbox does **not** do “happy eyeballs” parallel retrying like a browser. It picks one address based on its module settings.

| Option | Meaning |
|---|---|
| `preferred_ip_protocol` | Choose which family Blackbox prefers first: `ip4` or `ip6` |
| `ip_protocol_fallback` | Whether Blackbox may fall back to the other family |

### Practical advice

For your Mac + Docker learning environment:
- start with `preferred_ip_protocol: ip4`
- keep `ip_protocol_fallback: true` unless you specifically want strict family testing

If you want to prove IPv6 is healthy, create a separate module dedicated to IPv6.

---

## 14) What to choose for common monitoring goals

| Goal | Best module | Why |
|---|---|---|
| Check whether a machine is reachable at all | `icmp` | Closest to ping |
| Check whether a service port is open | `tcp_connect` | Directly tests `host:port` |
| Check website or API uptime | `http_2xx` | Verifies actual HTTP behavior |
| Confirm HTTPS is enforced | `http_2xx` | Use `fail_if_not_ssl: true` |
| Detect wrong redirects | `http_2xx` | Use `follow_redirects: false` and inspect failure/debug |
| Validate page/API content | `http_2xx` | Body, header, JSON validation support |
| Check SMTP/SSH banner | `tcp` with `query_response` | Better fit than HTTP |
| Test path MTU or routing hops | `icmp` | Use `dont_fragment` or `ttl` |

---

## 15) Recommended starter modules

```yaml
modules:
  icmp:
    prober: icmp
    timeout: 5s
    icmp:
      preferred_ip_protocol: ip4

  tcp_connect:
    prober: tcp
    timeout: 5s
    tcp:
      preferred_ip_protocol: ip4

  http_2xx:
    prober: http
    timeout: 5s
    http:
      method: GET
      preferred_ip_protocol: ip4
      follow_redirects: true
      fail_if_not_ssl: false
```

Then test them like this:

```bash
curl "http://localhost:9115/probe?target=8.8.8.8&module=icmp"
curl "http://localhost:9115/probe?target=google.com:443&module=tcp_connect"
curl "http://localhost:9115/probe?target=https://google.com&module=http_2xx"
```

---

## 16) Best practice for your environment

For local Docker on a Mac:

1. Start with **HTTP** and **TCP** because they are usually the most reliable.
2. Add **ICMP** once the container has the privileges it needs.
3. Prefer **HTTP** for real application health.
4. Prefer **TCP** for basic port/service checks.
5. Prefer **ICMP** only for network reachability and routing-style checks.

A very practical stack is:
- `icmp` for host reachability
- `tcp_connect` for core ports
- `http_2xx` for URLs and APIs

That gives you a full layered view:
- network up?
- port up?
- app up?

---

## 17) Source notes

This README is based on the official Prometheus Blackbox Exporter documentation and configuration schema, especially:

- README for supported probe types, Docker usage, `probe_success`, `/probe`, `/metrics`, and `debug=true`
- CONFIGURATION for module schema and HTTP/TCP/ICMP options

