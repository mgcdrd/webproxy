webproxy deployment
====================

Two-node active/passive nginx reverse proxy cluster with a floating VIP
(`mgcdrd.infrasvc.keepalived`, `webproxy` preset) and self-signed SSL via
`mgcdrd.infrabase.ssl_scripting` by default.

Built 2026-07-10 alongside `deployments/vsftp` and `deployments/haproxy-lb`
as part of a broader review/build pass — see that session's notes in memory
if picking this up cold.

---

## Status: infrastructure only, no site configured yet

This deployment stands up nginx + keepalived + SSL on the `webproxy` hosts
in `inventory-common`. It does **not** yet proxy any real backend —
`nginx_server_blocks` is empty. Add a real entry once there's something to
proxy to (see `inventory/group_vars/all/main.yml` for the commented
example, and the nginx role's own README for the full `server_block`
schema). Note the `server_block` template only manages the vhost shell —
actual `proxy_pass`/`location` directives come from a separate file you
supply at `proxy_files_path`, which this role does not manage.

---

## Prerequisites

- Two Rocky 9/10 or Debian 12/13 VMs, reachable via SSH with `become: true`
- Hosts added to the `webproxy` group in `../../inventory-common/hosts.yml`
  — currently `proxy1.example.com` / `proxy2.example.com` placeholders,
  replace with real FQDNs
- A real, free VIP address set in `keepalived_vrrp_instances[0].virt_ip` —
  currently a placeholder (`192.168.1.50/24`)
- `../../inventory-common` cloned as a sibling of `deployments/` (see that
  repo's README)
- Collections installed:
  `ansible-galaxy collection install -r collections/requirements.yml`

---

## Usage

```bash
ansible-playbook site.yml
```

No tags — both phases (nginx, keepalived) run every time. Both roles are
idempotent.

---

## Firewall

`inventory-common/group_vars/webproxy.yml` opens 22, 80, 443 for the
`webproxy` group — applied by whichever deployment runs
`mgcdrd.infrabase.firewall` against these hosts (normally `deployments/harden`,
run once before this deployment).

---

## SSL

`nginx_ssl_provider` picks which role `mgcdrd.infrasvc.nginx` calls to
obtain certs — `ssl_scripting` or `acme_sh`. Both blocks are already in
`inventory/group_vars/all/main.yml`; switching is commenting one out and
uncommenting the other.

- **`ssl_scripting`** (default) — self-signed, safe out of the box, no
  external dependency, but browsers will warn.
- **`acme_sh`** — real DNS-validated certs via the lab's PowerDNS
  (`dns_pdns` challenge; `dns_cf` also available). Needs `vault_acme_email`
  and `vault_pdns_api_key` — copy
  `inventory/group_vars/all/vault.yml.example` to `vault.yml` (gitignored).
  Set a real domain in the commented `acme_sh_certs` entry once the VIP has
  a DNS record. See `mgcdrd.infrabase.acme_sh`'s own `defaults/main.yml` for
  the full `acme_sh_*` variable set.

---

## Client/customer delivery

Portable as-is: point `ansible.cfg`'s inventory path at the customer's
`inventory-<client>` repo instead of `inventory-common`, add their real
`webproxy` hosts/VIP, and set real `nginx_server_blocks` for whatever
they're actually proxying.
