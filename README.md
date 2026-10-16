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
their content. Each site has two such files: `<name>.proxies` (the `location`
blocks, inside the server block) and `<name>.upstreams` (`upstream { }` blocks,
included above it, so nginx can keep backend connections open). Devops must
follow the header-inheritance rule (a location that sets its own `add_header` or
`proxy_set_header` has to include the matching snippet first); the role ships
`/usr/local/sbin/nginx-proxy-lint` to check this, and the sync script runs it on
every published file. The nginx role README, "Writing proxy and upstream files",
is the authoring guide to give the devops team. Requires `nginx_proxy_files_group` set to the group that
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

## Variable reference

Role variables not listed here keep the defaults in the role READMEs ([`nginx`](../../collections/infrasvc/roles/nginx/README.md), [`keepalived`](../../collections/infrasvc/roles/keepalived/README.md), [`acme_sh`](../../collections/infrabase/roles/acme_sh/README.md), [`ssl_scripting`](../../collections/infrabase/roles/ssl_scripting/README.md), [`sudoers`](../../collections/infrabase/roles/sudoers/README.md)).

### Set in `inventory/group_vars/all/main.yml`

Shared by every cluster.

| Variable | Shipped value | Required | Description |
|---|---|---|---|
| `nginx_ssl_enable` | `true` | No | Manage SSL certs |
| `nginx_ssl_provider` | `ssl_scripting` | No | Role that provides certs: `ssl_scripting` (self-signed, no dependencies) or `acme_sh` (real DNS-validated certs) |
| `nginx_ssl_scripting` | `{base_dir: /etc/nginx/ssl_scripting}` | No | Options passed to `ssl_scripting`; only used with that provider |
| `nginx_http_block` | `add_mime: true`, `force_80_redirect: true`, `hsts_max_age: 63072000`, `block_badagents: true`, `gzip_comp_lvl: 6`, `client_body_buffer_size: 64k`, `default_443: {enabled: true, reject_handshake: true}` | No | `http` block settings; `reject_handshake` refuses TLS for unknown or missing SNI (no cert served, nginx starts with no server block); `block_badagents` blocks scanners, scrapers and empty User-Agent for the public tier |
| `nginx_shared_configs` | `ssl`: TLS 1.2/1.3 and ECDHE-only ciphers (no cert paths, those are per server block); `logging`: access log (`local6`, tag `nginx_access`) and error log (`warn`, tag `nginx_error`) to syslog | No | Shared per-server-block snippets; takes effect once `nginx_server_blocks` is non-empty. Each server block also writes its own `/var/log/nginx/<name>.access.log` and `.error.log` |
| `nginx_geoip2_enabled` | `false` | No | GeoIP2 country lookup and geoblocking; needs the module package in the host's repos (see Hardening) |
| `keepalived_vip_for` | `[webproxy]` | No | keepalived preset to activate |

### Per instance (`inventory-common/instances/webproxy/<name>/group_vars/webproxy_<name>/`)

| Variable | Shipped value | Required | Description |
|---|---|---|---|
| `keepalived_vrrp_instances` | none in this repo | Yes | Master/backup nodes and VIP for the cluster |
| `nginx_server_blocks` | none | No | Sites to proxy (`name`, `server_names`, `exclude_proxy_files`, optional `proxy_files_path`, `upstream_files_path`, `cert_name`, `upstream_host`, `optional_snippets`, `allow_indexing`); empty means infrastructure only |
| `nginx_trusted_cidrs` | none | No | Trusted tier: rate limited, with high ceilings |
| `nginx_real_ip_from` | none | Only behind a load balancer/CDN | Addresses allowed to supply the real client address |
| `nginx_geoblock_countries` | none | No | Countries blocked for the public tier (with GeoIP2 on) |
| `acme_sh_issuer_host` | none | If `acme_sh_vault_kv_enabled` | Literal hostname that issues certs for the cluster |

### Optional, commented out in `main.yml`

| Variable | Example value | Required | Description |
|---|---|---|---|
| `acme_sh_email` | `{{ vault_acme_email }}` | If `nginx_ssl_provider: acme_sh` | ACME account email |
| `acme_sh_ca` | `letsencrypt` | No | Certificate authority |
| `acme_sh_pdns_url` | `http://dns.{{ domain }}:8081` | If `challenge: dns_pdns` | PowerDNS API URL for DNS-01 |
| `acme_sh_pdns_token` | `{{ vault_pdns_api_key }}` | If `challenge: dns_pdns` | PowerDNS API key |
| `acme_sh_flat_ssl_dir` | `/etc/nginx/ssl` | With `acme_sh` | Flat `<domain>.crt`/`.key` files; each server block's static `ssl_certificate` points here. Owner stays `root`, since the nginx master reads them |
| `acme_sh_certs` | one cert per FQDN (`domains`, `challenge: dns_pdns`, `reload_cmd`) | If `nginx_ssl_provider: acme_sh` | Certificates to issue; add an entry per site |
| `acme_sh_vault_kv_enabled` | `true` | No | Issue on one node and fan certs out to the rest through Vault, avoiding Let's Encrypt duplicate-cert limits |
| `acme_sh_vault_kv_mount` | `{{ vault_kv_infra_mount }}` | If Vault fan-out | KV mount |
| `acme_sh_vault_kv_path_prefix` | `{{ vault_kv_env }}/webproxy/certs` | If Vault fan-out | KV path prefix |
| `acme_sh_vault_deploy_reload_cmd` | `systemctl reload nginx` | If Vault fan-out | Reload command on non-issuer nodes |
| `nginx_proxy_files_group` | `svc-webproxy-devops` | If any server block has `exclude_proxy_files: false` | Group of the devops service account that edits proxy files |
| `nginx_proxy_sync_enabled` | `true` | No | Pull proxy files from S3 on a timer instead of editing them on-node; replaces `nginx_proxy_files_group` and the sudoers grant |
| `nginx_proxy_sync_interval` | `5min` | No | Pull interval |
| `nginx_proxy_sync_s3_endpoint` | `https://s3.example.com:9000` | If proxy sync | S3-compatible endpoint |
| `nginx_proxy_sync_s3_bucket` | `proxy` | If proxy sync | Bucket |
| `nginx_proxy_sync_s3_prefix` | `webproxy` | No | Key prefix |
| `nginx_proxy_sync_s3_ca_file` | `/etc/pki/ca-trust/source/anchors/lab-ca.pem` | No | CA bundle for a private-CA endpoint |
| `sudoers_rules_group` | `webproxy` | If `sudoers_rules` | Namespace for the drop-in files |
| `sudoers_rules` | `nginx-test-restart`: `nginx -t` and `nginx -s reload` for `svc-webproxy-devops` | No | Sudo grant for the devops account; the sudoers play is skipped unless this is set. Unneeded once proxy sync is on |

### Vault (`inventory/group_vars/all/vault.yml.example`)

Copy to `vault.yml` (gitignored). Needed only with `nginx_ssl_provider: acme_sh` or GeoIP2 database downloads; the default provider needs no secrets.

| Variable | Source | Required | Description |
|---|---|---|---|
| `vault_acme_email` | Vault KV `<vault_kv_env>/acme`, field `email` | If `acme_sh` | ACME account email |
| `vault_pdns_api_key` | Vault KV `<vault_kv_env>/pdns`, field `api_key` | If `dns_pdns` | PowerDNS API key |
| `vault_maxmind_account_id` | Vault KV `<vault_kv_env>/maxmind`, field `account_id` | If `nginx_geoip2_enabled` with `geoipupdate` | MaxMind account ID |
| `vault_maxmind_license_key` | Vault KV `<vault_kv_env>/maxmind`, field `license_key` | If `nginx_geoip2_enabled` with `geoipupdate` | MaxMind license key (written to `/etc/GeoIP.conf`, mode 0600) |

The AppRole `role_id`/`secret_id` files that the Vault fan-out and proxy sync use (`/etc/vault/...`) are pre-positioned by the identity build and have no variable here.

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

Certificates are **static per server block**: each block's
`ssl_certificate`/`ssl_certificate_key` point at
`/etc/nginx/ssl/<cert_name>.crt|.key` (`cert_name` defaults to the first
`server_names` entry). That makes `nginx -t` catch a missing or misnamed cert,
keeps the private keys readable by the root master only, and lets OCSP stapling
work. A cert with SANs is listed once, under its primary name; put every SAN in
`server_names`, and no per-SAN symlinks are needed. The trade-off is that nginx
re-reads a renewed cert only on reload, so the issuer's `reload_cmd` must reload
nginx (the shipped `acme_sh` entries do).

`ssl_scripting` only creates a dummy cert and dhparam, not per-site certs. With
that provider, put each site's cert files at the static path, or set
`ssl_cert_path`/`ssl_key_path` on the server block.

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

## Hardening

Everything below lives in the `mgcdrd.infrasvc.nginx` role; its README has the
full detail. This deployment turns on the safe defaults and leaves the
environment-specific ones to the instance's `group_vars/webproxy_<name>/`.

- **Client tiers and rate limits.** Internal (RFC1918) is never limited,
  geoblocked or user-agent blocked; `nginx_trusted_cidrs` get high limits;
  everyone else is the public tier with tight limits. Over-limit requests get a
  429 (logged at `warn` in the server block's error log).
- **GeoIP2 and geoblocking.** Off (`nginx_geoip2_enabled: false`) until the
  `nginx-mod-http-geoip2` package is available to the hosts. On Rocky that package is
  built from the `geoip2-rpm` project against the distro nginx and published through
  the Foreman content view; nothing is compiled on the proxy. Enable it, set
  `nginx_geoblock_countries`, and supply the MaxMind account ID and license key
  from Vault (`vault_maxmind_*`). The country is logged as `country=`, and the
  `grafana_alloy` regex accepts it.
- **User-agent blocking.** `block_badagents: true` blocks self-identifying
  scanners, SEO/AI scrapers and requests with no User-Agent, public tier only.
- **Behind a load balancer.** Set `nginx_real_ip_from`. Without it `$remote_addr`
  is the balancer and tiers, limits, geoblocking and fail2ban all act on the
  balancer instead of the client.
- **Banning abusive clients.** fail2ban is a separate role in `mgcdrd.infrabase`
  (`fail2ban`) and reads the per-server-block logs under `/var/log/nginx/`.
- **Behavior changes** from earlier versions of this deployment (CORS now opt-in,
  `expires off`, `X-Forwarded-For` from the connecting address, 10s connect timeout)
  are listed in the nginx role README under "Behavior changes".

Requirements: the role versions pinned in `collections/requirements.yml` must
include these changes; the pins were not bumped here.

---

## Client/customer delivery

Portable as-is: point `ansible.cfg`'s inventory paths at the customer's
`inventory-<client>` repo instead of `inventory-common`, add an
`instances/webproxy/<name>/` directory there with their real hosts/VIP, and
set real `nginx_server_blocks` for whatever they're actually proxying.
