# Migration Guide: v0.1.x → v0.2.0

This guide covers migrating from the single-service chart layout (`v0.1.x`) to the
new multi-service layout (`v0.2.0`).

## Summary of changes

- **`service` is replaced by `services`** — a list of one or more services.
- **New `metrics` block** — enables a Prometheus `ServiceMonitor` CRD.
- **`ingress` gains `serviceName` / `servicePort`** — to target a specific service.
- Chart version bumped to `0.2.0`.

---

## 1. Services

### Before (v0.1.x)

```yaml
service:
  type: ClusterIP
  port: 80
  targetPort: 7011
```

### After (v0.2.0)

```yaml
services:
  - name: http
    type: ClusterIP
    port: 80
    targetPort: 7011
```

Notes:

- Each entry in `services` becomes its own Kubernetes `Service`.
- `name` is required. It is used in the Service name (`<release>-<chart>-<name>`)
  and as the port name.
- `type` is optional and defaults to `ClusterIP`.

### Adding a metrics service (example)

```yaml
services:
  - name: http
    type: ClusterIP
    port: 80
    targetPort: 7011
  - name: metrics
    type: ClusterIP
    port: 9091
    targetPort: 9091
```

---

## 2. Ingress

`ingress.serviceName` and `ingress.servicePort` now select which Service the
Ingress routes to.

### Before (v0.1.x)

```yaml
ingress:
  enabled: true
  hosts:
    - host: example.com
```

### After (v0.2.0)

```yaml
ingress:
  enabled: true
  serviceName: http   # must match a name in `services`
  servicePort: 80
  hosts:
    - host: example.com
```

`serviceName` defaults to `http` if omitted. `servicePort` must be set.

---

## 3. Metrics (ServiceMonitor)

New in `v0.2.0`. To scrape Prometheus metrics, enable `metrics` and point it at
a service port.

```yaml
metrics:
  enabled: true
  interval: 30s
  scrapeTimeout: 10s
  metric:
    port: metrics   # must match a `name` in `services`
    path: /metrics
```

The ServiceMonitor selects the Service whose
`app.kubernetes.io/service` label equals `metric.port`. So `metric.port` should
match a service `name` from the `services` list.

When `metrics.enabled` is `false` (default), no ServiceMonitor is rendered.

---

## 4. Configurable domain (v0.3.0)

New in `v0.3.0`. The base domain is now a variable. Set `ingress.domain`
(defaults to `srv.mdpd.ir`); hosts in `ingress.hosts[].host` and
`ingress.tls[].hosts` are treated as **subdomains** and rendered as
`<sub>.<domain>`.

### Before (v0.2.x)

```yaml
ingress:
  enabled: true
  serviceName: http
  servicePort: 80
  hosts:
    - host: report.srv.mdpd.ir
  tls:
    - secretName: report-tls
      hosts:
        - report.srv.mdpd.ir
```

### After (v0.3.0)

```yaml
ingress:
  enabled: true
  serviceName: http
  servicePort: 80
  domain: srv.mdpd.ir   # optional, defaults to srv.mdpd.ir
  hosts:
    - host: report       # subdomain only
  tls:
    - secretName: report-tls
      hosts:
        - report         # subdomain only
```

To change the domain across the cluster, update `ingress.domain` in a single
place instead of editing every host string.

---

## Rollback

To roll back to `v0.1.x`, restore the old `service` block and set
`metrics.enabled: false`:

```yaml
service:
  type: ClusterIP
  port: 80
  targetPort: 7011
```
