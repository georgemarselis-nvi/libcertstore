# libcertstore

Unified local certificate store for Linux. Tracks, renews, and notifies services of certificate changes, regardless of CA or client tool.

Includes the `certctl` command-line tool. A network-capable daemon (`certstored`) is planned.

---

## Problem

Linux has no standard location or API for service certificates. `certbot` uses `/etc/letsencrypt/`, `acme.sh` uses `~/.acme.sh/`, and every CA client does its own thing. Services must be manually pointed at cert paths, renewal hooks are per-tool, and there is no way to ask the system "what certificates are installed and when do they expire?"

`libcertstore` fixes this.

---

## Goals

- Single certificate registry for Linux, compliant with the Linux Filesystem Hierarchy Standard
- CA-agnostic: works with Let's Encrypt, internal CAs, self-signed, PKCS#12, anything
- Tool-agnostic: import certs from certbot, acme.sh, openssl, or manually
- Automatic renewal via pluggable CA backends
- Service notification on renewal (systemd, D-Bus, custom hooks)
- No daemon required for local use
- Future: `certstored` daemon for network delivery, multi-host environments, and enterprise CA integration

---

## Architecture

```
libcertstore (Rust library)
    │
    ├── Store backend (SQLite index + filesystem)
    ├── CA plugin interface (ACME, PKCS#12, manual, ...)
    ├── Renewal engine (systemd timer integration)
    └── Notification hooks (systemd, D-Bus, shell)

certctl (CLI — included)
    │
    ├── certctl list
    ├── certctl add
    ├── certctl remove
    ├── certctl renew
    ├── certctl status
    └── certctl notify
```

Planned:

```
certstored (daemon, optional — future)
    │
    ├── D-Bus API for local queries
    ├── Network certificate delivery (Kerberos, mTLS)
    ├── Pull model (default): clients request certs from the daemon
    ├── Push model (plugin): for dumb clients that cannot initiate requests
    │       e.g. embedded devices, managed switches, legacy appliances
    └── Multi-CA plugin registry
```

---

## Filesystem Policy

`libcertstore` recognizes exactly two certificate base paths, both defined by the Linux FHS:

| Path | Primary use |
|------|-------------|
| `/etc/pki/` | Preferred. Red Hat, Fedora, and FHS-compliant systems. |
| `/etc/ssl/` | Supported. Debian, Ubuntu, and derivatives. |

`/etc/pki/` is the preferred base. `/etc/ssl/` is supported but not preferred.

No other paths (e.g. `/etc/letsencrypt/`, `~/.acme.sh/`) are natively recognized. If you use a CA tool that writes elsewhere, you have two options:

1. Configure `libcertstore` to import from that path on renewal.
2. Bind-mount or symlink the foreign directory into `/etc/pki/certstore/` — `libcertstore` can manage this automatically if configured to do so.

Default store layout:

```
/etc/pki/certstore/
    certs/          # PEM certificates, named by CN or fingerprint
    keys/           # Private keys, mode 0600
    chains/         # Full chains
    meta/           # SQLite index (expiry, CA, renewal config, hooks)
```

The base path is configurable via `/etc/certstore/certstore.conf` or `$CERTSTORE_BASE`.

---

## CA Plugins (Planned)

| Plugin | Status |
|--------|--------|
| ACME (Let's Encrypt, DNS-01/HTTP-01) | Planned |
| Manual (import existing certs) | Planned |
| Self-signed | Planned |
| Internal CA (PKCS#12) | Planned |
| FreeIPA / Dogtag | Future |
| Active Directory CS | Future |
| HashiCorp Vault | Future |

---

## CLI Usage

```bash
# List all tracked certificates
certctl list

# Add an existing certificate to the store
certctl add --cert /path/to/cert.pem --key /path/to/key.pem --chain /path/to/chain.pem

# Show status and expiry of all certs
certctl status

# Manually trigger renewal for a certificate
certctl renew --cn ldap.example.com

# Register a post-renewal notification hook
certctl notify --cn ldap.example.com --exec "systemctl restart slapd"

# Remove a certificate from the store
certctl remove --cn ldap.example.com
```

---

## Renewal

`certctl` creates a dedicated systemd timer per certificate at registration time. Each cert renews independently with its own schedule.

Default renewal window: **30 days before expiry**. Configurable per cert.

```bash
# Register with default renewal window (30 days before expiry)
certctl add --cn ldap.example.com

# Renew 7 days before expiry
certctl add --cn ldap.example.com --renew-before 7d

# Live dangerously: renew 1 day before expiry
certctl add --cn ldap.example.com --renew-before 1d
```

Each registration creates:

- `/etc/systemd/system/certstore-renew-<cn>.timer`
- `/etc/systemd/system/certstore-renew-<cn>.service`

The timer fires daily and checks whether the cert is within the renewal window. If so, it renews. Renewal takes under 10 minutes. The previous cert remains active until the new one is in place.

Renewal hooks fire after successful renewal and can restart services, reload configs, or run arbitrary scripts.

> **Note:** Setting `--renew-before 1d` is supported but not recommended for production. If renewal fails for any reason, you have one day to fix it before the cert expires.

---

## Security

Certificates are public — tampering with them is not the primary threat. Private keys are. `libcertstore` secures keys through permissions, MAC policy, and audit logging.

### File Permissions

- `keys/` — `0700`, root only
- Individual key files — `0600`, root only
- `certs/` and `chains/` — `0755` / `0644`

### Audit Logging with auditd

`libcertstore` does not implement its own tamper detection. Instead, use `auditd` to watch the key directory at the kernel level:

```bash
auditctl -w /etc/pki/certstore/keys/ -p rwxa -k certstore_keys
```

This logs every read, write, execute, and attribute change, tagged `certstore_keys`. Query with:

```bash
ausearch -k certstore_keys
```

You get: who, what process, what UID, what syscall, what time. Forward to syslog or a SIEM for alerting.

### SELinux / AppArmor

A reference SELinux policy is planned that restricts key access to `certctl` and explicitly allowlisted service executables (e.g. `slapd`, `httpd`). Everything else is denied and logged. On non-SELinux systems, an AppArmor profile will be provided.

### If Someone Has Root

No cert store design survives a root compromise. The correct response is host recovery, not tamper detection. Audit logging exists to tell you *when* and *how* the compromise happened, not to prevent it.

---



Requires Rust 1.75+.

```bash
git clone https://github.com/yourname/libcertstore-certctl
cd libcertstore-certctl
cargo build --release
```

Install:

```bash
cargo install --path .
```

---

## Roadmap

| Version | Milestone |
|---------|-----------|
| 0.1 | Core store, `certctl list/add/remove/status`, filesystem backend |
| 0.2 | Renewal engine, systemd timer integration, post-renewal hooks |
| 0.3 | ACME plugin (DNS-01 + HTTP-01), Let's Encrypt support |
| 0.4 | D-Bus API, `certstored` daemon |
| 0.5 | Network certificate delivery, Kerberos 6 integration |
| 1.0 | Stable API, multi-CA plugin registry, enterprise-ready |

---

## Why Rust?

- Memory-safe handling of private key material
- Single statically-linked binary, no runtime dependencies
- Easy to package for any Linux distro
- C FFI for `libcertstore` if other languages need to link against it

---

## License

MIT or Apache-2.0, at your option.

---

## Status

Pre-alpha. Design phase. Contributions and RFC-style discussion welcome.
