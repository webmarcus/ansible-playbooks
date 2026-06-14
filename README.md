# ansible-playbooks — DEPRECATED

This repo is retired. The deploy flow moved into the infra repo:

→ **`webmarcus/infra` → `ansible/`**

Everything here was obsolete: Bitwarden Secrets Manager (now pondra/OpenBao),
Redis fact caching, Docker (now rootless Podman quadlets), the distributed Dagu
coordinator/worker setup (Dagu is a tsnet node now), and the `personal.yml` snap
apps. The only piece worth keeping — Tailscale dynamic inventory
(`freeformz.ansible.tailscale`) — was carried over to `infra/ansible/`, now
sourcing its API token from pondra instead of Bitwarden.

See `infra/ansible/README.md`.
