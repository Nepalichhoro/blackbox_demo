# Blackbox Exporter Setup (Mac + Docker)

## Overview
This guide helps you run Prometheus Blackbox Exporter locally on macOS using Docker for:
- TCP port monitoring
- ICMP (ping) monitoring

---

## 1. Create Working Directory

```bash
mkdir -p ~/blackbox-demo
cd ~/blackbox-demo
```

---

## 2. Create Config File

Create `blackbox.yml`:

```yaml
modules:
  tcp_connect:
    prober: tcp
    timeout: 5s

  icmp:
    prober: icmp
    timeout: 5s
```

---

## 3. Run Blackbox Exporter

```bash
docker run -d   --name blackbox_exporter   -p 9115:9115   --cap-add=NET_RAW   -v "$(pwd)/blackbox.yml:/config/blackbox.yml:ro"   quay.io/prometheus/blackbox-exporter:latest   --config.file=/config/blackbox.yml
```

---

## 4. Verify Container

```bash
docker ps
curl http://localhost:9115/metrics | head
```

---

## 5. Test TCP Monitoring

```bash
curl "http://localhost:9115/probe?target=google.com:443&module=tcp_connect"
```

Expected:
```
probe_success 1
```

More examples:

```bash
curl "http://localhost:9115/probe?target=github.com:443&module=tcp_connect"
curl "http://localhost:9115/probe?target=1.1.1.1:53&module=tcp_connect"
```

---

## 6. Test ICMP (Ping)

```bash
curl "http://localhost:9115/probe?target=8.8.8.8&module=icmp"
```

Debug mode:

```bash
curl "http://localhost:9115/probe?target=8.8.8.8&module=icmp&debug=true"
```

---

## 7. Important Notes (Mac)

- TCP works reliably
- ICMP may fail due to Docker limitations
- Use TCP for most monitoring needs

---

## 8. Check Logs

```bash
docker logs blackbox_exporter
```

---

## 9. Restart Container

```bash
docker rm -f blackbox_exporter
```

---

## Key Metric

- `probe_success 1` → success
- `probe_success 0` → failure

---

## Summary

- Use TCP for ports (recommended)
- ICMP may require extra privileges
- Integrate later with Prometheus + Grafana

