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

## Infra/devops split (proxy files)

Deliberate boundary: this deployment (infra) owns the server block shell,
certs, and hardening; a devops service account owns each site's actual
`proxy_pass`/`location` content. `mgcdrd.infrasvc.nginx` only ensures
`nginx_proxy_files_dir` (`/etc/nginx/conf.d/proxies` by default — a
dedicated directory, not the rest of `/etc/nginx/conf.d/`) and an empty
placeholder file exist per server block; it never manages or overwrites
their content. Requires `nginx_proxy_files_group` set to the group that
account already belongs to (this deployment doesn't provision the account
itself). See `mgcdrd.infrasvc.nginx`'s README, "Proxy files", for the full
ownership/permission model.

The sudo rights that account needs to test/restart nginx after editing a
proxy file are wired too (`mgcdrd.infrabase.sudoers`, skipped unless
`sudoers_rules` is set — see `inventory/group_vars/all/main.yml`'s
commented example, same account as `nginx_proxy_files_group` above).

Proxy sync is an opt-in way to propagate edits to every node. Instead of
editing on one node, the devops team's CI publishes each site's proxy file to
a local S3-compatible bucket. A systemd timer on every node (every 5 minutes
by default, set with `nginx_proxy_sync_interval`) pulls the files, runs
`nginx -t`, and reloads, rolling a file back if it fails the test. Nobody
edits on the node, so the sudoers grant and `nginx_proxy_files_group` above
aren't needed. The commented example in `inventory/group_vars/all/main.yml`
shows the variables. The `mgcdrd.infrasvc.nginx` README, under "Proxy sync",
covers the bucket contract, the prerequisites (a bucket, a read-only
credential in Vault, and AppRole files on each node) and the limitations.
With sync off, a devops edit on one node only takes effect on that node.

---

## Prerequisites

- Two Rocky 9/10 or Debian 12/13 VMs, reachable via SSH with `become: true`
- An instance directory at `../../inventory-common/instances/webproxy/<name>/`
  (see Inventory below) — currently `proxy1.example.com` /
  `proxy2.example.com` placeholders, replace with real FQDNs
- A real, free VIP address set in the instance's
  `keepalived_vrrp_instances[0].virt_ip` — currently a placeholder
  (`192.168.1.50/24`)
- `../../inventory-common` cloned as a sibling of `deployments/` (see that
  repo's README)
- Collections installed:
  `ansible-galaxy collection install -r collections/requirements.yml`

---

## Inventory

Hosts and cluster-specific vars live in
`../../inventory-common/instances/webproxy/<name>/`, one directory per proxy
pair, not in this deployment (see that repo's README, "Multiple instances").
Pick the pair with `DEPLOY_INSTANCE`; `ansible.cfg` builds the inventory path
from it. An unset or misspelled name fails the first play (`instance_guard`)
instead of silently matching no hosts. Load one instance per run — never two.

An instance's `hosts.yml` lists each node in `webproxy_<name>` and again in
`webproxy`, the role group the plays target:

```yaml
all:
  children:
    webproxy_example:
      hosts:
        proxy1.example.com:
        proxy2.example.com:
    webproxy:
      hosts:
        proxy1.example.com:
        proxy2.example.com:
```

Cluster-specific vars (`keepalived_vrrp_instances`, `nginx_server_blocks`,
`acme_sh_issuer_host`) live in that instance's `group_vars/webproxy_<name>/`.
Everything the same for every cluster (SSL provider, HTTP block, logging)
stays in this repo's `inventory/group_vars/all/main.yml`. Requires
`mgcdrd.infrabase` with the `instance_guard` role (v0.21.0 or later).

---

## Usage

```bash
DEPLOY_INSTANCE=lab ansible-playbook site.yml
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

### Certificate renewal architecture (with `acme_sh`)

With 2+ nodes each independently issuing the same domain set, every
renewal cycle burns against Let's Encrypt's 5-duplicate-certs/week limit
once per node instead of once total. The commented `acme_sh` block also
sets `acme_sh_issuer_host` (pinned to one literal proxy hostname) and
`acme_sh_vault_kv_enabled: true` so only that one node issues; every other
node pulls the issued cert from Vault instead — see
`mgcdrd.infrabase.acme_sh`'s README, "Issuer + Vault fan-out", for the full
mechanism. This decouples issuance from serving entirely (DNS-01 has no
dependency on which node holds the VIP), so the keepalived VIP moving
between nodes never affects which node renews.

This alone does not make the Vault-pull side self-healing on its own — an
AWX schedule running `site.yml` periodically against `webproxy` is what
turns "pulls on the next run" into actual convergence, and isn't set up by
this deployment.

---

## Client/customer delivery

Portable as-is: point `ansible.cfg`'s inventory paths at the customer's
`inventory-<client>` repo instead of `inventory-common`, add an
`instances/webproxy/<name>/` directory there with their real hosts/VIP, and
set real `nginx_server_blocks` for whatever they're actually proxying.
