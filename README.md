# Proxy [![CI | pre-commit](https://github.com/ed-asriyan/xray-server/actions/workflows/CI-pre-commit.yml/badge.svg)](https://github.com/ed-asriyan/xray-server/actions/workflows/CI-pre-commit.yml) [![CD | Production](https://github.com/ed-asriyan/xray-server/actions/workflows/CD-production.yml/badge.svg)](https://github.com/ed-asriyan/xray-server/actions/workflows/CD-production.yml)
This is deployment for my personal server with [xray](https://xtls.github.io/en/) on board for me and my friends to bypass internet censorship.

## Vless clients that work with this setup
https://hiddify.com#app

# Architecture
![digram](./diagram.svg)

There are 4 componets: **Cloudflare Pages**, **[Supabase](https://supabase.com) instance**, **metrics** and **proxy**. GH Actions that must be configured for this repository.
Metrics should be deployed as a single instance (sharding is not allowed). Proxies could be deployed as many instances as needed,
each instance should have dedicated IP address and DNS record (if exists). All hosts should be Debian hosts with public IPs.

# [Supabase](https://supabase.com)
Stores list of user configs. Users may generate new user configs (my means of HTML website). Deploymemnt fetch records from there
to render configs for xrays and other components. _The repository with Supabas is not open-source at the moment. It will becore
open-source later_.

## Cloudflare Pages
Serves static content:
* static html pages with installation instructions which is being developed in a separate repository:
[xray-server-frontend](https://github.com/ed-asriyan/xray-server-frontend). The user is provided with a private instruction link
with a personal ShadowSocks configuration, which the user uses once to install the ShadowSocks configuration* personal dynamic
ShadowSocks configuration json files ([SIP008](https://shadowsocks.org/doc/sip008.html)) for each client, which is used by
ShadowSocks client each time before connecting to a ShadowSocks server
* personal vless [subscription files](https://hiddify.com/app/URL-Scheme) for each client, which is used by Hiddify to refresh
list of available servers

Playbook: [frontman.yml](./frontman.yml). It just renders files locally, the should be uploaded got Cloudflare Pages using
Actions.

## Metrics
It is a single linux host with the prometheus installed. Users do not access this host. Host may have no domain name.
* [prometheus](https://prometheus.io) (role: `prometheus`): monitoring to detect traffic abuse

Playbook: [metrics.yml](./metrics.yml)

## Proxy
As many proxy hosts as needed could be deployed but each one should have its own IP address and/or DNS record.
Proxy(ies) is/are linux host(s) with installed
* [xray-core](https://github.com/xtls/xray-core) (role: `xray`) that proxies traffic:
  * if it's valid vless connection, to the destination
  * otherwise, to `server.fallback_proxy_target`
* [node-exporter](https://github.com/prometheus/node_exporter) (role: `node-exporter`): Prometheus exporter for hardware and OS
metrics. Exports metrics to TCP port available from localhost only
* xray-exporter (role: `node-exporter`): custom script that exports xray metrics. Exports metrics to TCP port available from
localhost only
* [nginx](https://nginx.org) (role: `metrics-exporter`) that proxies https requests on
`config_servers[uuid].prometheus_metrics.port` TCP port to *node-exporter* and *xray-exporter*:

Playbook: [proxies.yml](./proxies.yml)

# Development
This part requires [Ansible](https://www.ansible.com) knowledge. The deployment is tested on and implemented for Debian
only.

## At the very beginning
1. Initialize pre-commit hook to prevent secrets from being leaked:
   1. Install [pre-commit](https://pre-commit.com/#install)
   2. Initialize pre-commit hook:
      ```commandline
      pre-commit install
      ```
3. If servers are not configured yet, skip this step and go to "New server setup" section. Otherwise if server is already configured, add SSH private key to `id_rsa` file in the root of the local repository. **Make sure that only you have
permissions to read/write it: `chmod 600 id_rsa`!**

## Initial setup
1. Go to [config](./config) and setup config files or GitHub Secrets.
2. Add yout public key (pair of one you created in root of the local repo) to all servers' root user
3. Run Deploy

## How to update list of SS users
1. Update config in [config](./config) or update GitHub Secrets.
2. Run Deploy

## How to add anew
1. Update config in [config](./config) or update GitHub Secrets.
2. Add yout public key (pair of one you created in root of the local repo) to the new server's root user
3. Run Deploy

## How to do smth else
Read code and find out

## How to apply changes to production
* If you changed deploy code: just push to master branch. GitHub Actions will automatically apply updates to the servers.
* If you changed list of users: manually trigger [CD | Production](https://github.com/ed-asriyan/xray-server/actions/workflows/CD-production.yml)

# Development
## CD
The following GitHub secrets are required for CD:
* `KNOWN_HOSTS`: list of known hosts as in `.ssh/known_hosts`
* `SSH_PRIVATE_KEY`: SSH private key to access servers
* secrets described in [config](./config)

## Local
### Deploy proxies & metrics on production
```commandline
make deploy_proxies deploy_metrics
```

### Generate UUID for a new user
```commandline
make generate_uuid
```
