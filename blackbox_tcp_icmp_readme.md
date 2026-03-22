# Blackbox Exporter: `tcp_connect` vs `icmp`

A practical mini-README for running and understanding Blackbox Exporter on Docker for Mac.

## What a module is

In Blackbox Exporter, a **module** is a named probe definition inside `blackbox.yml`. Each module chooses a **prober** (`http`, `tcp`, `dns`, `icmp`, `grpc`, etc.) and can add protocol-specific options like timeout, preferred IP family, TLS, payload size, and request/response matching.

---

## High-level difference

- **`tcp_connect`** answers: **“Can I open a TCP socket to this host:port?”**
- **`icmp`** answers: **“Can I reach this host with ping-style network packets?”**

That means:
- `tcp_connect` is best for **port/service reachability**
- `icmp` is best for **basic host/network reachability**

---

## Comparison table

| Area | `tcp_connect` module | `icmp` module |
|---|---|---|
| What it checks | Whether a **TCP connection** can be established to a target port | Whether the target replies to **ICMP echo** requests (ping-style) |
| Typical target format | `host:port` such as `google.com:443` | hostname or IP such as `8.8.8.8` or `google.com` |
| Main question answered | “Is this **service/port** reachable?” | “Is this **host/network path** reachable?” |
| Layer focus | More **service-level / transport-level** | More **network-level** |
| Good for | HTTPS port 443, SSH 22, DB ports like 3306/5432, Redis 6379 | Router/server reachability, basic network checks |
| Not enough by itself for | HTTP correctness, API status codes, app health semantics | Port availability, service correctness |
| Config section | `tcp:` | `icmp:` |
| Key options | `preferred_ip_protocol`, `ip_protocol_fallback`, `source_ip_address`, `tls`, `tls_config`, `query_response` | `preferred_ip_protocol`, `ip_protocol_fallback`, `source_ip_address`, `dont_fragment`, `payload_size` |
| Extra interaction capability | Can send/expect bytes or regex with `query_response`; can use TLS or STARTTLS | No application handshake; only network echo behavior |
| Docker-on-Mac practicality | Usually the **easier** and more reliable choice locally | Can be **trickier** because ICMP often needs raw-socket privileges/runtime support |
| Best mental model | “Can I open the door to that port?” | “Can I even reach that building?” |

---

## `tcp_connect` explained

### What it is

`tcp_connect` uses the `tcp` prober. It tries to open a TCP connection to a target endpoint, usually something like:

- `google.com:443`
- `localhost:22`
- `mydb.internal:5432`

If the exporter can open that socket within the timeout, the probe is considered successful.

### What success means

If you see:

```text
probe_success 1
```

it generally means:

- DNS resolution worked if a hostname was used
- routing/network path was good enough
- the target host accepted a TCP connection on that port

It does **not** automatically mean:

- the application is healthy in a business sense
- the HTTP response was 200
- the database accepted a login

### Why it is useful

This is one of the best synthetic checks for:

- HTTPS listener up on `:443`
- SSH reachable on `:22`
- Postgres reachable on `:5432`
- MySQL reachable on `:3306`
- Redis reachable on `:6379`

### Important `tcp` options

#### 1. `preferred_ip_protocol`
Choose whether the probe should prefer:

- `ip4`
- `ip6`

Example:

```yaml
preferred_ip_protocol: ip4
```

This is often useful on Docker Desktop for Mac when IPv6 is not what you want first.

#### 2. `ip_protocol_fallback`
If set to `true`, Blackbox can fall back to the other IP family if the preferred one fails.

Example:

```yaml
preferred_ip_protocol: ip4
ip_protocol_fallback: true
```

#### 3. `tls`
If `true`, the TCP probe initiates TLS from the start of the connection.

Example use case:
- probe an endpoint where you want a raw TLS connection over TCP

#### 4. `tls_config`
Lets you tune TLS behavior, such as CA files, hostname verification, or min/max TLS version.

#### 5. `query_response`
This is where `tcp_connect` becomes more powerful than a simple port-open test.

You can:
- send data
- expect a regex match
- expect exact bytes
- do `starttls`
- attach labels to expectations

That means you can check not only that a port is open, but also that the service speaks the protocol you expect.

### Example: basic TCP port check

```yaml
modules:
  tcp_connect:
    prober: tcp
    timeout: 5s
    tcp:
      preferred_ip_protocol: ip4
```

Probe it with:

```bash
curl "http://localhost:9115/probe?target=google.com:443&module=tcp_connect"
```

### Example: TCP + TLS

```yaml
modules:
  tcp_tls:
    prober: tcp
    timeout: 5s
    tcp:
      preferred_ip_protocol: ip4
      tls: true
```

### Example: banner / protocol expectation

```yaml
modules:
  smtp_banner:
    prober: tcp
    timeout: 5s
    tcp:
      preferred_ip_protocol: ip4
      query_response:
        - expect: "^220"
```

This kind of probe can confirm that the remote endpoint is not just listening, but replying with something that looks like an SMTP banner.

### When to use `tcp_connect`

Use it when you care about:

- a specific port being open
- whether a service is listening
- whether load balancer / firewall / SG rules permit access
- quick service-level reachability checks

---

## `icmp` explained

### What it is

`icmp` uses the ICMP prober to send ping-style packets to a target. This is closer to the classic idea of “Can I ping that host?”

Typical targets:

- `8.8.8.8`
- `1.1.1.1`
- `google.com`

### What success means

If you see:

```text
probe_success 1
```

it generally means:

- the host was reachable over the network path used
- ICMP echo request/reply worked

It does **not** mean:

- any application port is open
- HTTP is working
- the database is accepting traffic

A host can reply to ping while the application port is down.

And the reverse is also possible:
- the host may block ping
- but TCP/HTTPS still works perfectly

### Important `icmp` options

#### 1. `preferred_ip_protocol`
Like TCP, this lets you prefer `ip4` or `ip6`.

```yaml
preferred_ip_protocol: ip4
```

#### 2. `ip_protocol_fallback`
Allows fallback to the other IP family if the preferred one fails.

#### 3. `source_ip_address`
Lets you choose the source IP for the probe.

#### 4. `dont_fragment`
Controls the IP “Don’t Fragment” bit. The config docs note this is only for IPv4, works on Unix-like systems, and requires raw sockets/root or `CAP_NET_RAW` on Linux.

This is more advanced and can help with MTU/path-MTU style testing.

#### 5. `payload_size`
Lets you choose the ICMP payload size.

That can help simulate different packet sizes or test behavior around fragmentation and path MTU.

### Example: basic ICMP probe

```yaml
modules:
  icmp:
    prober: icmp
    timeout: 5s
    icmp:
      preferred_ip_protocol: ip4
```

Probe it with:

```bash
curl "http://localhost:9115/probe?target=8.8.8.8&module=icmp"
```

### Example: ICMP with payload size

```yaml
modules:
  icmp_large:
    prober: icmp
    timeout: 5s
    icmp:
      preferred_ip_protocol: ip4
      payload_size: 1200
```

### When to use `icmp`

Use it when you care about:

- basic host reachability
- rough network-path visibility
- whether something is alive at the network level
- simple ping-style status checks

---

## Side-by-side examples

### Example 1: A website host answers ping, but app port is down

- `icmp` may succeed
- `tcp_connect` to `host:443` may fail

Interpretation:
- machine/network path is reachable
- web service is not reachable on that port

### Example 2: Ping is blocked, but website still works

- `icmp` may fail
- `tcp_connect` to `host:443` may succeed

Interpretation:
- firewall/security blocks ping
- actual service is still available

This is very common on the public internet.

---

## Practical recommendation for your Mac

For local Docker-on-Mac testing:

1. Start with **`tcp_connect` first**
2. Then try **`icmp`**
3. If ICMP fails, do not assume Blackbox is broken
4. Treat ICMP as more environment-sensitive because it often depends on raw socket permissions/capabilities

If your real goal is **port monitoring**, `tcp_connect` is the better primary module anyway.

---

## Suggested starter config

```yaml
modules:
  tcp_connect:
    prober: tcp
    timeout: 5s
    tcp:
      preferred_ip_protocol: ip4
      ip_protocol_fallback: true

  icmp:
    prober: icmp
    timeout: 5s
    icmp:
      preferred_ip_protocol: ip4
      ip_protocol_fallback: true
```

---

## Useful test commands

### TCP

```bash
curl "http://localhost:9115/probe?target=google.com:443&module=tcp_connect"
curl "http://localhost:9115/probe?target=github.com:443&module=tcp_connect"
curl "http://localhost:9115/probe?target=localhost:22&module=tcp_connect"
```

### ICMP

```bash
curl "http://localhost:9115/probe?target=8.8.8.8&module=icmp"
curl "http://localhost:9115/probe?target=google.com&module=icmp"
```

### Debug

```bash
curl "http://localhost:9115/probe?target=8.8.8.8&module=icmp&debug=true"
```

---

## Metrics to watch

Some commonly useful probe metrics include:

- `probe_success`
- `probe_duration_seconds`

For your day-to-day tests, `probe_success` is the main one:

- `1` = success
- `0` = failure

---

## Rule of thumb

- Use **`icmp`** to ask: **“Can I reach the machine?”**
- Use **`tcp_connect`** to ask: **“Can I reach the service port?”**

For service monitoring, **`tcp_connect` is usually more actionable**.

---

## References

- Prometheus Blackbox Exporter README: https://github.com/prometheus/blackbox_exporter
- Prometheus Blackbox Exporter CONFIGURATION.md: https://github.com/prometheus/blackbox_exporter/blob/master/CONFIGURATION.md
