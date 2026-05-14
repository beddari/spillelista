# Local Split-DNS and Internal PKI — Developer Setup

This document records the choices made for running a local split-DNS resolver
and an internal Certificate Authority in developer environments (Linux and
macOS), and gives a step-by-step setup guide.

It is aimed at developers spinning up a workstation to work against local
clusters that need:

- DNS lookups for some domains resolved against an internal/cluster resolver
  rather than the upstream resolver, and
- Internally trusted TLS certificates for cluster services, issued via ACME.

## TL;DR

- **Split-DNS resolver: CoreDNS** (Homebrew / Linuxbrew).
- **Internal CA: step-ca** with an ACME provisioner (Homebrew / Linuxbrew).
- **Cert challenge type: HTTP-01 (or TLS-ALPN-01)** for per-name, internally
  issued certificates. No wildcard certificates, no RFC 2136 dynamic updates,
  no second authoritative DNS server.
- **Cert automation in clusters: cert-manager** with an ACME issuer pointing
  at step-ca.
- **Tenant-facing / public-CA scenarios are out of scope here** — see
  "Future directions" below.

## Decisions and rationale

### Why split-DNS at all

We want some names (e.g. `*.cluster.local`, `svc.lab.example.com`) to resolve
against an internal resolver while everything else continues to resolve via
the developer's normal upstream DNS. A local forwarding resolver is the
cleanest way to do this without touching `/etc/hosts` or fighting with
`resolv.conf` per project.

### Why CoreDNS (not dnsmasq, not Unbound)

| Option   | Split-DNS forwarding | Notes |
|----------|----------------------|-------|
| dnsmasq  | Good                 | Simple but limited; awkward to extend. |
| Unbound  | Excellent            | Recursive resolver only; not authoritative. |
| CoreDNS  | Excellent            | Plugin-based config, same engine as Kubernetes' in-cluster DNS, easy to mirror cluster behavior locally. |

CoreDNS is chosen because:

- Developers are already familiar with its `Corefile` from Kubernetes.
- The `forward` plugin makes split-DNS a one-liner per zone.
- It is available on both Homebrew and Linuxbrew.
- Authoritative-zone limitations (no RFC 2136) do not matter for us — see
  the next section.

### Why no wildcard certificates / no RFC 2136

ACME wildcard issuance requires the DNS-01 challenge, which in self-hosted
setups means the DNS server must support **RFC 2136 (Dynamic DNS Update)**
with TSIG. Neither dnsmasq nor CoreDNS support RFC 2136 in a way that
mainstream ACME clients can target. Supporting it would mean adding a second
authoritative DNS server (Knot or BIND) just to host the
`_acme-challenge` records.

We avoid that complexity by issuing **per-name certificates with HTTP-01**:

- Blast radius is per service, not per parent domain.
- Audit trail in step-ca's log maps one issuance to one hostname.
- No shared wildcard key sitting in cluster secret stores.
- No dynamic-DNS infrastructure to maintain.

Per-name renewals are not an operational burden when automated by
cert-manager.

### Why step-ca (not a public CA)

For internal cluster services, certificates only need to be trusted by other
internal clients (developers, cluster components, CI). A self-hosted CA is
the right tool:

- step-ca exposes a standard ACME provisioner, so cert-manager and any ACME
  client just works.
- Supports HTTP-01 and TLS-ALPN-01 natively. Since step-ca runs inside the
  network, these challenge types resolve without DNS-01.
- Short-lived certs (hours/days) are the default, which is healthier than
  long-lived wildcards.
- The `step` CLI bootstraps trust on developer machines with one command.

### Future directions (not implemented here)

These come up but are explicitly *out of scope* for the local dev setup:

- **Public, tenant-facing certs.** Customers will not trust step-ca's root,
  so external hostnames need a public CA (Let's Encrypt, ZeroSSL). The
  best fit for "issue per-tenant on first request" is **Caddy with
  on-demand TLS** and a mandatory `ask` endpoint for authorization.
- **k8s Gateway API + on-demand TLS.** No mainstream Gateway API
  implementation does true on-demand-by-SNI today. The idiomatic path is
  cert-manager reconciling `Certificate` resources created by your control
  plane at tenant-signup time ("JIT-at-signup" rather than
  "JIT-on-first-request"). If on-first-request is non-negotiable, put
  Caddy on the edge in front of the Gateway.
- **Public wildcard via RFC 2136.** If we ever do need this, the cleanest
  pairing is CoreDNS for split-DNS plus a tiny Knot DNS instance
  authoritative only for `_acme-challenge.<zone>`, reached via a CNAME
  delegation from the real public zone. Keeps Knot's scope minimal.

## Setup guide

The setup is identical on macOS and Linux because both use Homebrew.

### 1. Install Homebrew (if not already installed)

macOS and Linux:

    /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

On Linux, follow the post-install instructions Homebrew prints to add
`brew shellenv` to your shell profile.

### 2. Install CoreDNS, step-ca, and the step CLI

    brew install coredns step step-ca

`step` is the client CLI used to bootstrap and interact with the CA;
`step-ca` is the CA server itself.

### 3. Bootstrap step-ca

Pick a directory for CA state, e.g. `~/.step-ca`:

    step ca init \
      --name "Dev Internal CA" \
      --dns "ca.lab.test" \
      --address ":8443" \
      --provisioner "admin@lab.test" \
      --deployment-type standalone

This generates a root + intermediate, a default JWK provisioner, and a
config at `$(step path)/config/ca.json`.

Add an ACME provisioner so cert-manager and other ACME clients can request
certificates:

    step ca provisioner add acme --type ACME

Start the CA:

    step-ca $(step path)/config/ca.json

For long-running dev use, wrap this in a launchd plist (macOS) or a
systemd user unit (Linux). Example systemd user unit:

    # ~/.config/systemd/user/step-ca.service
    [Unit]
    Description=step-ca local dev CA

    [Service]
    ExecStart=%h/.local/bin/step-ca %h/.step/config/ca.json
    Restart=on-failure

    [Install]
    WantedBy=default.target

Then:

    systemctl --user daemon-reload
    systemctl --user enable --now step-ca

### 4. Trust the CA root on the developer machine

    step certificate install $(step path)/certs/root_ca.crt

On Linux this updates the system trust store; on macOS it adds to the
login keychain. Browsers and most language runtimes will pick it up.

### 5. Configure CoreDNS for split-DNS

Create a `Corefile` at `~/.coredns/Corefile`:

    # Internal zones: forward to the cluster's DNS, or serve from a zone file.
    lab.test:53 {
        forward . 10.0.0.10
        log
        errors
        cache 30
    }

    cluster.local:53 {
        forward . 10.96.0.10
        log
        errors
        cache 30
    }

    # Everything else: forward to the host's upstream resolver(s).
    . :53 {
        forward . /etc/resolv.conf
        log
        errors
        cache 300
    }

Run it on a high port to avoid needing root, e.g. `1053`, and point your
system resolver at `127.0.0.1:1053`. Example:

    coredns -conf ~/.coredns/Corefile -dns.port 1053

Wrap it in a launchd/systemd unit the same way as step-ca.

### 6. Point the OS resolver at CoreDNS

**macOS** — create `/etc/resolver/lab.test` (one file per internal zone):

    nameserver 127.0.0.1
    port 1053

macOS will route queries for that zone through CoreDNS while leaving the
rest of the system alone. Repeat for `cluster.local` and any other
internal zone.

**Linux (systemd-resolved)** — add a drop-in to route specific domains to
the local CoreDNS:

    # /etc/systemd/resolved.conf.d/coredns.conf
    [Resolve]
    DNS=127.0.0.1:1053
    Domains=~lab.test ~cluster.local

    sudo systemctl restart systemd-resolved

The `~` prefix marks them as routing-only domains, so other queries are
unaffected.

### 7. Issue a cert from step-ca (smoke test)

    step ca certificate test.lab.test test.crt test.key

This uses the default JWK provisioner. To smoke-test the ACME path:

    export STEP_CA_URL=https://ca.lab.test:8443
    export STEP_CA_FINGERPRINT=$(step certificate fingerprint $(step path)/certs/root_ca.crt)
    # Any ACME client pointed at https://ca.lab.test:8443/acme/acme/directory
    # will work, e.g. lego, acme.sh, certbot, cert-manager.

### 8. Wire cert-manager in clusters

In each cluster, add an `Issuer` (or `ClusterIssuer`) pointing at step-ca:

    apiVersion: cert-manager.io/v1
    kind: ClusterIssuer
    metadata:
      name: step-ca
    spec:
      acme:
        server: https://ca.lab.test:8443/acme/acme/directory
        privateKeySecretRef:
          name: step-ca-account-key
        solvers:
          - http01:
              ingress:
                ingressClassName: nginx

The CA's root certificate must be mounted into cert-manager so it can
trust the ACME endpoint — either by adding it to the cert-manager Pod's
CA bundle or via the `caBundle` field on the Issuer.

## Operational notes

- step-ca defaults to short-lived certs. Make sure cert-manager renewals
  are running; the default `renewBefore` is fine.
- CoreDNS port 53 needs root on Linux/macOS. Using a high port plus the
  OS resolver mapping (steps 5–6) avoids that.
- If a developer's upstream DNS is itself `127.0.0.53` (systemd-resolved),
  the `forward . /etc/resolv.conf` line in the catch-all block will loop.
  Forward to an explicit upstream (`1.1.1.1`, `8.8.8.8`, or the office
  resolver) in that case.
- Keep step-ca's root + intermediate keys backed up out-of-band. Losing
  them means re-trusting on every developer machine.

## References

- CoreDNS: https://coredns.io
- step-ca: https://smallstep.com/docs/step-ca
- cert-manager ACME issuer: https://cert-manager.io/docs/configuration/acme/
- RFC 2136 (Dynamic DNS Update) — relevant if we ever revisit wildcards.
