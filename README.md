# libcertstore

Unified local certificate store for Linux. Tracks, renews and notifies services of certificate changes, regardless of CA or client tool.

Includes the `certctl` command-line tool. A network-capable daemon (`certstored`) is planned.

---

## Problem

Linux has no standard location or API for service certificates. `certbot` uses `/etc/letsencrypt/`, `acme.sh` uses `~/.acme.sh/` and every CA client does its own thing. Services must be manually pointed at cert paths, renewal hooks are per-tool and there is no way to ask the system "what certificates are installed and when do they expire?"

`libcertstore` fixes this.

---

## Goals

- Single certificate registry for Linux, compliant with the Linux Filesystem Hierarchy Standard
- CA-agnostic: works with Let's Encrypt, DigiCert, GlobalSign, Sectigo, ZeroSSL, Google Trust Services, AWS Private CA, internal CAs, self-signed, PKCS#12, anything with an ACME or REST API
- Tool-agnostic: import certs from certbot, acme.sh, openssl or manually
- Automatic renewal via pluggable CA backends
- Service notification on renewal (systemd, D-Bus, custom hooks)
- No daemon required for local use
- Future: `certstored` daemon for network delivery, multi-host environments and enterprise CA integration

---

## Architecture

```
libcertstore (Rust library)
    │
    ├── Store backend (SQLite index + filesystem)
    ├── CA plugin interface (ACME, PKCS#12, manual, ...)
    ├── Renewal engine (systemd timer integration)
    └── Notification hooks (systemd, D-Bus, shell)

certctl (CLI - included)
    │
    ├── certctl list [--ca] [--end-entity] [--expired]
    ├── certctl add
    ├── certctl remove
    ├── certctl revoke
    ├── certctl renew
    ├── certctl rotate
    ├── certctl status [--check-revocation]
    ├── certctl scan
    └── certctl notify
```

Planned:

```
certstored (daemon, optional - future)
    │
    ├── Certificate proxy: brokers requests to local or remote CA backends
    │       clients request certs from certstored; certstored handles CA selection
    │       clients do not need to know or care which CA is used
    ├── Signed delivery: certstored signs all cert deliveries with its own key
    │       clients verify the signature before accepting - prevents MITM substitution
    ├── Validation: validates all certs in the store on startup and on import
    │       catches expired or corrupt certs before they cause service failures
    ├── Real-time expiry tracking: marks certs invalid the moment they expire
    │       store state is always authoritative -- no polling required
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
| `/etc/pki/` | Preferred. Red Hat convention, widely adopted. `libcertstore` advocates for FHS standardization of this path. |
| `/etc/ssl/` | Supported. Debian, Ubuntu and derivatives. |

`/etc/pki/` is the preferred base. `/etc/ssl/` is supported but not preferred.

No other paths (e.g. `/etc/letsencrypt/`, `~/.acme.sh/`) are natively recognized. If you use a CA tool that writes elsewhere, configure `libcertstore` with the foreign path and it will bind-mount it into `/etc/pki/certstore/` automatically. Bind mounts are used instead of symlinks to prevent accidental deletion or redirection.

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

# List CA certificates only
certctl list --ca

# List end-entity certificates only
certctl list --end-entity

# List expired certificates (tombstoned -- visible for audit but not usable)
certctl list --expired

# Add an existing or pre-issued certificate to the store (including airgapped hosts via removable media)
certctl add --cert /path/to/cert.pem --key /path/to/key.pem --chain /path/to/chain.pem

# Show status and expiry of all certs
certctl status

# Check revocation status via OCSP or CRL
certctl status --check-revocation

# Manually trigger renewal for a certificate
certctl renew --cn ldap.example.com

# Register a post-renewal notification hook
certctl notify --cn ldap.example.com --exec "systemctl restart slapd"

# Remove a certificate from the local store only
certctl remove --cn ldap.example.com

# Purge all expired certificates from the store
certctl remove --purge-expired

# Rotate key pair and request a fresh certificate from the CA
# Old key and cert are tombstoned
certctl rotate --cn ldap.example.com

# Revoke a certificate with the issuing CA and remove it from the local store
certctl revoke --cn ldap.example.com
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

The timer fires once at the renewal date. If renewal fails, it retries with exponential backoff -- it does not attempt renewal every day for the entire window. The previous cert remains active until the new one is in place. Renewal takes under 10 minutes.

Renewal hooks fire after successful renewal and can restart services, reload configs or run arbitrary scripts.

> **Note:** Setting `--renew-before 1d` is supported but not recommended for production. If renewal fails for any reason, you have one day to fix it before the cert expires.

### Certificate Overlap with Internal CAs

Public CAs (Let's Encrypt, DigiCert, etc.) issue certificates with `notBefore` set to the time of issuance. Pre-dating is not supported.

Internal CAs support setting `notBefore` to a date in the past or future. When using an internal CA backend, `libcertstore` can request the new cert with `notBefore` set 30 days before the old cert expires -- providing a clean overlap window with no gap in validity. This eliminates any risk of a service seeing an expired cert during the switchover.

```bash
# Enable notBefore overlap for internal CA (ignored for public CAs)
certctl add --cn ldap.example.com --ca internal --overlap 30d
```

---

## Provisioning Integration

### Automatic Certificate Discovery

`certctl scan` walks the system and finds existing certificates regardless of where they were placed -- by certbot, acme.sh, manual installation or any other tool. Found certs are registered into the store without re-issuance. No assumption is made that a host needs a new cert; discovery comes first.

Certs are automatically categorized on import by reading the `basicConstraints: CA:TRUE` X.509 extension -- no manual classification needed. CA certs and end-entity certs are stored and tracked separately.

```bash
certctl scan
certctl scan --path /etc/letsencrypt/live/
```

### Preseed and Kickstart Integration

`libcertstore` provides hooks for Debian preseed and Red Hat Kickstart so a host certificate can be requested and installed during provisioning, before the system is fully up. The cert is requested from whatever CA backend is configured -- Let's Encrypt, an internal CA or `certstored` on the network.

Debian preseed example:

```
d-i preseed/late_command string certctl scan && certctl request --cn $(hostname --fqdn) --ca letsencrypt
```

Kickstart example:

```
%post
certctl scan
certctl request --cn $(hostname --fqdn) --ca letsencrypt
%end
```

This means a freshly provisioned host has a valid cert from first boot, with no manual steps.

---



Every package manager, every service and every distro currently ships its own certificate handling code. `dnf`, `apt`, language runtimes and individual services all reinvent the same wheel - fetching, installing and trusting certificates in subtly different ways.

`libcertstore` is the answer to that. Instead of writing cert handling code in your package or service, call `libcertstore`. One integration, one store, one place to audit.

If your service needs a certificate:

1. Link against `libcertstore` or query `certstored` over the socket API
2. Ask for the cert by CN
3. Get back the path or PEM - done

No custom install scripts. No hardcoded paths. No reinventing renewal logic. The system handles it.

---



`libcertstore` is designed to be consumed by other services, not just used from the command line.

### Rust Crate

`libcertstore` is a native Rust crate. Add it to your `Cargo.toml` and query the store directly:

```rust
use libcertstore::Store;

let store = Store::open()?;
let cert = store.get("ldap.example.com")?;
println!("{}", cert.path());
```

### C FFI

A C-compatible FFI layer is provided so any language with a C FFI can link against `libcertstore` -- Python, Go, C, C++ and others.

```c
#include <certstore.h>

certstore_t *store = certstore_open();
certstore_cert_t *cert = certstore_get(store, "ldap.example.com");
printf("%s\n", certstore_cert_path(cert));
```

### IPC / Socket API (with certstored)

When `certstored` is running, services can query it over a local Unix socket without linking against the library at all. The protocol is simple and language-agnostic.

```bash
# Query certstored for a cert path
certctl query --cn ldap.example.com
```

Bindings for common languages and service managers are planned.

---



Certificates are public - tampering with them is not the primary threat. Private keys are. `libcertstore` secures keys through permissions, MAC policy and audit logging.

### File Permissions

- `keys/` - `0700`, root only
- Individual key files - `0600`, root only
- `certs/` and `chains/` - `0755` / `0644`

### Audit Logging with auditd

`libcertstore` does not implement its own tamper detection. Instead, use `auditd` to watch the key directory at the kernel level:

```bash
auditctl -w /etc/pki/certstore/keys/ -p rwxa -k certstore_keys
```

This logs every read, write, execute and attribute change, tagged `certstore_keys`. Query with:

```bash
ausearch -k certstore_keys
```

You get: who, what process, what UID, what syscall and what time. Forward to syslog or a SIEM for alerting.

### SELinux / AppArmor

A reference SELinux policy is planned that restricts key access to `certctl` and explicitly allowlisted service executables (e.g. `slapd`, `httpd`). Everything else is denied and logged. On non-SELinux systems, an AppArmor profile will be provided.

### Hardware Security Module (HSM)

For `certstored` operating as a subordinate CA, the signing key should never exist in software. `libcertstore` supports PKCS#11 so the subordinate CA signing key is generated on and never leaves the HSM. Cert signing requests are passed to the HSM; the signed cert is returned. The private key is not exportable.

Enterprise CAs will require HSM key storage before issuing a subordinate CA certificate. This is the correct architecture regardless.

For regular host certificates, HSM is not required.

Tested HSM targets: SoftHSM2 (development), Nitrokey HSM 2, YubiHSM 2 and any PKCS#11-compliant device.

---



No cert store design survives a root compromise. The correct response is host recovery, not tamper detection. Audit logging exists to tell you *when* and *how* the compromise happened, not to prevent it.

---



## Syslog and Event Integration

`libcertstore` logs all operations to syslog with a dedicated identifier (`certstore`). Example message templates are shipped with the package so sysadmins can wire them into rsyslog, syslog-ng or a SIEM without reverse engineering log output.

Example messages:

```
certstore[1234]: INFO  cert=ldap.example.com action=renewed ca=letsencrypt expiry=2026-09-27
certstore[1234]: INFO  cert=ldap.example.com action=rotated old_fingerprint=aa:bb:cc new_fingerprint=dd:ee:ff
certstore[1234]: WARN  cert=ldap.example.com action=expiry_imminent days_remaining=7
certstore[1234]: ERROR cert=ldap.example.com action=renewal_failed ca=letsencrypt reason="DNS-01 challenge failed"
certstore[1234]: ERROR cert=ldap.example.com action=expired
certstore[1234]: INFO  cert=ldap.example.com action=revoked ca=letsencrypt
certstore[1234]: WARN  cert=ldap.example.com action=ocsp_revoked
```

rsyslog and syslog-ng filter examples are provided under `contrib/syslog/` in the repository. A Prometheus exporter exposing cert expiry, revocation status and store health metrics is provided under `contrib/prometheus/` for those who prefer metrics-based monitoring.

---



Requires Rust 1.75+.

```bash
git clone https://github.com/georgemarselis/libcertstore
cd libcertstore
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
| 0.6 | CI/CD pipeline: GitHub Actions producing `.rpm` (Fedora/RHEL) and `.deb` (Debian/Ubuntu) packages on every release |
| 0.7 | Package manager integration: `apt` post-invoke hook and `dnf` plugin that automatically register certs dropped by installed packages into `libcertstore` |
| 1.0 | Stable API, multi-CA plugin registry, enterprise-ready |
| 1.x | Subordinate CA support: `certstored` acts as an intermediate CA for a domain hierarchy, issuing certs for any host beneath it (e.g. `pgsql.db.example.com`) without contacting the upstream CA on every request. Requires a subordinate CA certificate from an enterprise CA (DigiCert, GlobalSign, Sectigo). Not supported by public CAs such as Let's Encrypt. |
| 2.x | TUI and GUI frontends. Last priority -- the library, CLI and daemon are the product. |

---

## Why Rust?

- Memory-safe handling of private key material
- Single statically-linked binary, no runtime dependencies
- Easy to package for any Linux distro
- C FFI for `libcertstore` if other languages need to link against it

---

## License

`libcertstore` is dual-licensed:

- **Apache-2.0** (default) - for commercial use, proprietary integration and open source projects that prefer liberal licensing. Includes explicit patent protection.
- **GPL-3.0** - for projects that require copyleft. Available on the `gpl` branch.

Choose whichever license fits your use case. Contributions are accepted under both and ported to both.

---

## Status

Pre-alpha. Design phase. Contributions and RFC-style discussion welcome.
