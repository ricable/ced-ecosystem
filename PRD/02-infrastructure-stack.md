# Infrastructure Stack — Edge Cluster, Platform Layer & Network Topology

> Source: consolidated from `infra-PRD.md`. Part of the ced-ecosystem PRD set — see `/Users/cedric/work/dev/ced-ecosystem/PRD/README.md` for the full document map.

**Project:** RANO Swarm (Radio Access Network Optimization Swarm)
**Author:** Cédric — Senior RAN Optimization Engineer, Orange France, Lille
**Revision:** ULTIMATE v1.0 — 2026-06-28
**Status:** CANONICAL REFERENCE — supersedes prior PRD versions
**Governed by:** the infra ADR skeletons in this document (local numbering ADR-001..012 + ADR-087; distinct from the canonical repo sequence in `/adr/` — cross-check by title, not number)
**Domain model:** DDD

---

## Table of Contents

0. [Relationship to the Ecosystem PRD](#relationship-to-the-ecosystem-prd)
1. [Vision, Goals & Hard Constraints](#vision-goals--hard-constraints)
2. [Hardware Inventory & Network Topology](#hardware-inventory--network-topology)
3. [Platform Layer: Kairos Factory, AuroraBoot & Netboot Pipeline](#platform-layer-kairos-factory-auroraboot--netboot-pipeline)
4. [Kubernetes Cluster: k3s HA Control Plane & Worker Fleet](#kubernetes-cluster-k3s-ha-control-plane--worker-fleet)
5. [Networking: NetBird Mesh, Reverse Proxy, DNS & TLS](#networking-netbird-mesh-reverse-proxy-dns--tls)
6. [Inference Stack: LocalAI, LiteLLM & MacBook MLX](#inference-stack-localai-litellm--macbook-mlx)
7. [RANO Swarm Agent Harness](#rano-swarm-agent-harness)
8. [RuVix Cognition Kernel Integration](#ruvix-cognition-kernel-integration)
9. [Observability: Prometheus, Grafana & Alerts](#observability-prometheus-grafana--alerts)
10. [Security Model: Secrets, RBAC & SafetyGate](#security-model-secrets-rbac--safetygate)
11. [Phase Plan](#phase-plan)
12. [Deep Research Prompts](#deep-research-prompts)
13. [ADR Skeletons](#adr-skeletons)
14. [DDD Model](#ddd-model)
15. [File Tree & Justfile](#file-tree--justfile)
16. [Glossary](#glossary)

---

## Relationship to the Ecosystem PRD

`01-ecosystem-architecture.md` (Aegis Harness) is the canonical top-level document; this PRD supplies the physical substrate it runs on. Boundary decisions an implementing swarm must respect:

- **Two runtimes, one policy.** Aegis coding/orchestration agents run in Herdr panes on the dev MacBook (01 §3.2, PRD 04); RANO production agents run as Rust k8s pods on the NUC fleet (this doc). Neither replaces the other. The MacBook is an inference peer of the cluster, never a k3s node.
- **Routing.** The LiteLLM tier table here is the in-cluster implementation of 01 §11 ("local first, compressed always, frontier only on escalation"). The dev-host equivalent is the ControllerLoop cost-budget router (`src/routing/`); keep tier names aligned when either changes.
- **Memory (open reconciliation).** This doc's agent memory is sqlite-vec inside pods (local ADR-011); 01 §9–10 mandates AgentDB + RuVector + RVF as the strategic memory substrate. Working assumption until an ADR closes it: sqlite-vec stays as pod-local working memory; evidence-bearing episodes/reflections/RAN cases are exported to RVF containers (`ruvector rvf create/ingest`) so the ecosystem memory layer can retrieve them.
- **ADR namespace.** ADR numbers in this doc (001..012, 087) are the source document's own sequence, not `/adr/`'s. Cross-check by title.

---

## Vision, Goals & Hard Constraints

### Vision

Deliver a fully autonomous, self-healing, closed-loop RAN optimization harness with immutable audit trails, running entirely on a sovereign edge cluster. The system observes RAN performance, reasons over it with local inference, proposes parameter changes, gates every mutation behind a cryptographic proof quorum, applies changes to the live network, and reverts automatically on KPI degradation — across nine acceptance phases (Phase 0–8).

### Strategic Goals

| ID | Goal | Success metric |
|---|---|---|
| G-01 | Zero-touch hardware provisioning | Power on any node → joins cluster without human input |
| G-02 | Sovereign inference | >80% of agent inference calls served locally (NUC + RPi4 + MacBook) |
| G-03 | Proof-gated RAN mutations | 100% of APPLY actions carry a valid RuVix proof chain |
| G-04 | Immutable audit trail | Every agent action witnessable and cryptographically linked |
| G-05 | No CUDA dependency | All local inference via GGUF CPU (LocalAI) or Metal (MLX) |
| G-06 | Data sovereignty | No PM counter data, RAN config, or agent memory leaves the mesh |
| G-07 | Self-healing cluster | Any single node failure → workloads reschedule within 60s |
| G-08 | OTA upgrades | OS upgrade to node via `kairos-agent upgrade` without manual SSH |

### Hard Constraints

| ID | Constraint |
|---|---|
| CONSTRAINT-01 | No Python in agent runtime (Rust/WASM pods) |
| CONSTRAINT-02 | No CUDA (Apple Silicon Metal + CPU only) |
| CONSTRAINT-03 | No public internet exposure of the cluster |
| CONSTRAINT-04 | All inter-node traffic E2E encrypted |
| CONSTRAINT-05 | API keys never stored in cloud-config or pod env plaintext |
| CONSTRAINT-06 | Kairos immutable — no runtime root writes |
| CONSTRAINT-07 | k3s token pre-shared before first boot |
| CONSTRAINT-08 | RPi4 RAM cap: LocalAI GGUF model ≤ 2GB per node |
| CONSTRAINT-09 | LiteLLM pinned to v1.72.6 (avoid 1.82.7/1.82.8) |
| CONSTRAINT-10 | RuVix proof quorum = 2-of-N agents before APPLY |

---

## Hardware Inventory & Network Topology

### Node Inventory

| Node | Model | Count | Arch | RAM | Storage | Role |
|---|---|---|---|---|---|---|
| nuc-0 | Intel NUC8i5BEH | 1 | x86_64 | 16–32GB | 256GB SSD | k3s server (`--cluster-init`) + AuroraBoot host + NetBird self-host |
| nuc-1..5 | Intel NUC8i5BEH | 5 | x86_64 | 16–32GB | 256GB SSD | k3s server (HA) + LocalAI |
| pi-1..10 | Raspberry Pi 4 | 10 | ARM64 | 4GB | 32GB microSD | k3s worker + LocalAI |
| MBP | MacBook M3 Max | 1 | ARM64 | 128GB | 2TB NVMe | inference peer only |

**Total cluster nodes:** 16 (6 NUC + 10 RPi4). The MacBook is an inference-only peer with no k3s role.

### Network Topology

```
LAN:              192.168.1.0/24
NetBird overlay:  100.64.0.0/10
k3s pod CIDR:     10.42.0.0/16
k3s svc CIDR:     10.43.0.0/16
```

### Static IP Assignments

```
nuc-0     192.168.1.10   / 100.64.0.10
nuc-1     192.168.1.11   / 100.64.0.11
nuc-2     192.168.1.12   / 100.64.0.12
nuc-3     192.168.1.13   / 100.64.0.13
nuc-4     192.168.1.14   / 100.64.0.14
nuc-5     192.168.1.15   / 100.64.0.15
pi-1..10  192.168.1.20..29 / 100.64.0.20..29
MacBook   192.168.1.5    / 100.64.0.5
```

---

## Platform Layer: Kairos Factory, AuroraBoot & Netboot Pipeline

### Kairos Factory Image Build

The NUC image is built with `kairos-init` (`--model generic`, `--provider k3s`) and layered with Docker, NetBird, and a k3s-ready stage hook.

**Image A: `rano/kairos-nuc:1.0.0` (amd64, k3s server)**

```dockerfile
# kairos/Dockerfile.nuc
FROM quay.io/kairos/kairos-init:v0.14.6 AS kairos-init
FROM ubuntu:24.04

ARG VERSION=1.0.0
ARG K3S_VERSION=v1.31.5+k3s1

RUN --mount=type=bind,from=kairos-init,src=/kairos-init,dst=/kairos-init \
    /kairos-init \
      --version "${VERSION}" \
      --model generic \
      --provider k3s \
      --provider-k3s-version "${K3S_VERSION}"

# Docker (for AuroraBoot + local registry on nuc-0)
RUN curl -fsSL https://get.docker.io | sh && systemctl enable docker

# NetBird
RUN curl -fsSL https://pkgs.netbird.io/install.sh | sh

# k3s-ready stage hook
RUN cat > /etc/systemd/system/k3s-ready.service <<'EOF'
[Unit]
Description=RANO k3s booted stage runner
After=k3s.service
[Service]
Type=oneshot
ExecStart=kairos-agent run-stage provider-kairos.bootstrap.after.k3s-ready
TimeoutSec=120
RemainAfterExit=yes
[Install]
WantedBy=k3s.service
EOF
RUN systemctl enable k3s-ready.service

RUN --mount=type=bind,from=kairos-init,src=/kairos-init,dst=/kairos-init \
    /kairos-init validate
```

```bash
docker build --platform linux/amd64 \
  --build-arg VERSION=1.0.0 \
  --build-arg K3S_VERSION=v1.31.5+k3s1 \
  -t 192.168.1.10:5000/rano/kairos-nuc:1.0.0 \
  -f kairos/Dockerfile.nuc kairos/
docker push 192.168.1.10:5000/rano/kairos-nuc:1.0.0
```

**Image B: `rano/kairos-rpi4:1.0.0` (arm64, k3s worker)**

```dockerfile
# kairos/Dockerfile.rpi4
FROM quay.io/kairos/kairos-init:v0.14.6 AS kairos-init
FROM ubuntu:24.04

ARG VERSION=1.0.0
ARG K3S_VERSION=v1.31.5+k3s1

RUN --mount=type=bind,from=kairos-init,src=/kairos-init,dst=/kairos-init \
    /kairos-init \
      --version "${VERSION}" \
      --model rpi4 \
      --provider k3s \
      --provider-k3s-version "${K3S_VERSION}"

RUN curl -fsSL https://pkgs.netbird.io/install.sh | sh

RUN cat > /etc/systemd/system/k3s-ready.service <<'EOF'
[Unit]
Description=RANO k3s booted stage runner
After=k3s-agent.service
[Service]
Type=oneshot
ExecStart=kairos-agent run-stage provider-kairos.bootstrap.after.k3s-ready
TimeoutSec=120
RemainAfterExit=yes
[Install]
WantedBy=k3s-agent.service
EOF
RUN systemctl enable k3s-ready.service

RUN --mount=type=bind,from=kairos-init,src=/kairos-init,dst=/kairos-init \
    /kairos-init validate
```

```bash
docker buildx build --platform linux/arm64 \
  --build-arg VERSION=1.0.0 \
  --build-arg K3S_VERSION=v1.31.5+k3s1 \
  -t 192.168.1.10:5000/rano/kairos-rpi4:1.0.0 \
  -f kairos/Dockerfile.rpi4 --load kairos/
docker push 192.168.1.10:5000/rano/kairos-rpi4:1.0.0
```

### Local OCI Registry (nuc-0, bootstrapped before provisioning)

```bash
# Run once before any AuroraBoot provisioning
docker run -d --name registry --restart always \
  -p 5000:5000 \
  -v /opt/rano/registry:/var/lib/registry \
  registry:2

# Allow insecure registry on nuc-0
cat > /etc/docker/daemon.json <<'EOF'
{ "insecure-registries": ["192.168.1.10:5000"] }
EOF
systemctl restart docker
```

### AuroraBoot Netboot Pipeline

Two images, two AuroraBoot runs. Port 67 (ProxyDHCP) can only be bound by one process at a time, so the NUC image and the RPi4 (UEFI) image are served sequentially.

**Phase 2a — nuc-0 install (USB ISO, one-time):**

```bash
# Build nuc-0 ISO on any Linux/Mac machine with Docker
docker run --rm \
  -v "$PWD/cloud-configs/nuc-0.yaml":/c.yaml \
  -v "$PWD/build":/tmp/ab \
  -v /var/run/docker.sock:/var/run/docker.sock \
  quay.io/kairos/auroraboot \
  --set container_image=docker://192.168.1.10:5000/rano/kairos-nuc:1.0.0 \
  --set disable_http_server=true \
  --set disable_netboot=true \
  --set state_dir=/tmp/ab \
  --cloud-config /c.yaml
# Flash build/kairos.iso to USB → boot nuc-0 from USB → one-time install
```

**Phase 2b — nuc-1..5 via PXE from nuc-0:**

```bash
# Run on nuc-0 (Linux, --net host works natively)
docker run --rm -ti --net host \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /opt/rano/configs:/configs:ro \
  quay.io/kairos/auroraboot \
  --set container_image=docker://192.168.1.10:5000/rano/kairos-nuc:1.0.0 \
  --cloud-config /configs/nuc-server.yaml
# Power on nuc-1..5 → PXE boot → auto-install → k3s HA join
# Wait for: kubectl get nodes shows 6 NUC nodes Ready
# Then Ctrl+C AuroraBoot
```

**Phase 2c — RPi4s via PXE from nuc-0:**

```bash
# RPi4 EEPROM must be updated to network-boot-first (one-time per Pi)
# Use Raspberry Pi Imager → Bootloader → Network Boot
docker run --rm -ti --net host \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /opt/rano/configs:/configs:ro \
  quay.io/kairos/auroraboot \
  --set container_image=docker://192.168.1.10:5000/rano/kairos-rpi4:1.0.0 \
  --cloud-config /configs/rpi4-worker.yaml
# Power on all 10 RPi4s → PXE boot → auto-install → k3s worker join
```

### kairos-init Flags Reference

| Flag | NUC value | RPi4 value | Effect |
|---|---|---|---|
| `--version` | `1.0.0` | `1.0.0` | Embedded in `/etc/kairos-release`, required for OTA upgrades |
| `--model` | `generic` | `rpi4` | RPi4: installs U-Boot + model-specific firmware |
| `--provider` | `k3s` | `k3s` | Installs k3s provider plugin |
| `--provider-k3s-version` | `v1.31.5+k3s1` | `v1.31.5+k3s1` | Must be exact k3s release tag |
| `--trusted-boot` | `false` | `false` | UKI Trusted Boot off (not supported on RPi4 UEFI) |

---

## Kubernetes Cluster: k3s HA Control Plane & Worker Fleet

### HA Control Plane (6 NUCs, embedded etcd)

The k3s embedded etcd quorum spans 6 server nodes: it tolerates 2 simultaneous server failures (quorum = 4).

### cloud-configs/nuc-0.yaml

```yaml
#cloud-config
hostname: rano-nuc-0
install:
  device: /dev/sda
  auto: true
  reboot: true
users:
  - name: kairos
    passwd: kairos
    groups: [admin]
    ssh_authorized_keys:
      - github:<GH_HANDLE>
k3s:
  enabled: true
  env:
    K3S_TOKEN: "<SHARED_TOKEN>"
  args:
    - --cluster-init
    - --disable=traefik,servicelb
    - --tls-san=192.168.1.10
    - --tls-san=100.64.0.10
    - --write-kubeconfig-mode=0644
    - --node-label=rano.io/role=control-plane
    - --node-label=rano.io/hw=nuc
write_files:
  - path: /etc/systemd/system/auroraboot-nuc.service
    permissions: "0644"
    content: |
      [Unit]
      Description=AuroraBoot NUC netboot server
      After=docker.service network-online.target
      [Service]
      Type=simple
      Restart=on-failure
      ExecStart=docker run --rm --net host \
        -v /var/run/docker.sock:/var/run/docker.sock \
        -v /opt/rano/configs:/configs:ro \
        quay.io/kairos/auroraboot \
        --set container_image=docker://192.168.1.10:5000/rano/kairos-nuc:1.0.0 \
        --cloud-config /configs/nuc-server.yaml
      [Install]
      WantedBy=multi-user.target
  - path: /oem/netbird.env
    permissions: "0600"
    content: |
      NB_SETUP_KEY=<KAIROS_K8S_NODES_SETUP_KEY>
      NB_MANAGEMENT_URL=https://192.168.1.10:443
stages:
  provider-kairos.bootstrap.after.k3s-ready:
    - name: "Start NetBird self-hosted"
      commands:
        - cd /opt/netbird && docker compose up -d
        - sleep 30
        - bash /opt/rano/scripts/netbird-bootstrap.sh
    - name: "Enable AuroraBoot for NUCs"
      commands:
        - systemctl enable --now auroraboot-nuc.service
    - name: "NetBird connect"
      commands:
        - source /oem/netbird.env
        - netbird up --setup-key "$NB_SETUP_KEY" --management-url "$NB_MANAGEMENT_URL" --hostname rano-nuc-0
```

### cloud-configs/nuc-server.yaml

nuc-1..5 HA servers (same token, no `--cluster-init`):

```yaml
#cloud-config
hostname: rano-nuc-{{ trunc 4 .MachineID }}
install:
  device: /dev/sda
  auto: true
  reboot: true
users:
  - name: kairos
    passwd: kairos
    groups: [admin]
    ssh_authorized_keys:
      - github:<GH_HANDLE>
k3s:
  enabled: true
  env:
    K3S_TOKEN: "<SHARED_TOKEN>"
  args:
    - --server https://192.168.1.10:6443
    - --disable=traefik,servicelb
    - --node-label=rano.io/role=control-plane
    - --node-label=rano.io/hw=nuc
    - --node-label=rano.io/inference=localai
write_files:
  - path: /oem/netbird.env
    permissions: "0600"
    content: |
      NB_SETUP_KEY=<KAIROS_K8S_NODES_SETUP_KEY>
      NB_MANAGEMENT_URL=https://192.168.1.10:443
stages:
  network:
    - name: "NetBird connect"
      commands:
        - source /oem/netbird.env
        - netbird up --setup-key "$NB_SETUP_KEY" --management-url "$NB_MANAGEMENT_URL" --hostname "$(hostname)"
```

### cloud-configs/rpi4-worker.yaml

```yaml
#cloud-config
hostname: rano-pi-{{ trunc 4 .MachineID }}
install:
  device: auto
  auto: true
  reboot: true
users:
  - name: kairos
    passwd: kairos
    groups: [admin]
    ssh_authorized_keys:
      - github:<GH_HANDLE>
k3s-agent:
  enabled: true
  env:
    K3S_TOKEN: "<SHARED_TOKEN>"
    K3S_URL: "https://192.168.1.10:6443"
  args:
    - --node-label=rano.io/role=worker
    - --node-label=rano.io/hw=rpi4
    - --node-label=rano.io/inference=localai
write_files:
  - path: /oem/netbird.env
    permissions: "0600"
    content: |
      NB_SETUP_KEY=<KAIROS_K8S_NODES_SETUP_KEY>
      NB_MANAGEMENT_URL=https://192.168.1.10:443
stages:
  network:
    - name: "NetBird connect"
      commands:
        - source /oem/netbird.env
        - netbird up --setup-key "$NB_SETUP_KEY" --management-url "$NB_MANAGEMENT_URL" --hostname "$(hostname)"
```

### k3s Token Pre-Generation (before any boot)

```bash
# Generate once, embed in ALL cloud-configs before provisioning
export K3S_TOKEN="$(openssl rand -hex 16)"
echo "$K3S_TOKEN" > .cluster-token   # gitignored
# Substitute into all templates:
sed -i "s/<SHARED_TOKEN>/$K3S_TOKEN/g" cloud-configs/*.yaml
```

---

## Networking: NetBird Mesh, Reverse Proxy, DNS & TLS

### NetBird Self-Hosted Stack (nuc-0)

NetBird v0.74+ combined container: management + signal + relay in one image. NetBird self-hosted runs on nuc-0 with management URL `https://192.168.1.10:443`. Traefik v3.1 fronts the Management and Signal gRPC services (`h2c://` backend), with WebSocket passthrough for the Relay service; Coturn provides the TURN/STUN relay path.

```yaml
# /opt/netbird/docker-compose.yml
version: "3.8"
services:
  traefik:
    image: traefik:v3.1
    ports:
      - "80:80"
      - "443:443"
    command:
      - --providers.docker=true
      - --entrypoints.web.address=:80
      - --entrypoints.websecure.address=:443
      - --certificatesresolvers.le.acme.tlschallenge=true
      - --certificatesresolvers.le.acme.email=ced@orange.local
      - --certificatesresolvers.le.acme.storage=/letsencrypt/acme.json
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - letsencrypt:/letsencrypt

  netbird-server:
    image: netbirdio/netbird-server:latest
    restart: unless-stopped
    environment:
      - NETBIRD_DOMAIN=netbird.rano.local
      - NETBIRD_LETSENCRYPT_DOMAIN=none
      - NB_SETUP_PAT_ENABLED=true
      - NB_RELAY_ADDRESS=netbird.rano.local:443
      - TURN_USER=netbird
      - TURN_PASSWORD=${TURN_PASSWORD}
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.nb-dash.rule=Host(`netbird.rano.local`)"
      - "traefik.http.routers.nb-dash.tls.certresolver=le"
      - "traefik.http.services.nb-dash.loadbalancer.server.port=8080"
    volumes:
      - netbird-data:/var/lib/netbird
    expose: ["8080","8081","10000","8084"]

  coturn:
    image: coturn/coturn:latest
    restart: unless-stopped
    ports:
      - "3478:3478/udp"
    command:
      - --listening-port=3478
      - --lt-cred-mech
      - --user=netbird:${TURN_PASSWORD}
      - --realm=netbird.rano.local
      - --no-tls
      - --no-dtls

volumes:
  letsencrypt:
  netbird-data:
```

### NetBird Automated Bootstrap Script

```bash
#!/usr/bin/env bash
# /opt/rano/scripts/netbird-bootstrap.sh
set -euo pipefail
NB="https://netbird.rano.local"

# First-owner + PAT creation
NB_PAT=$(curl -fsS -X POST "$NB/api/setup" \
  -H "Content-Type: application/json" \
  -d "{\"email\":\"admin@rano.local\",\"name\":\"RANO Admin\",
       \"password\":\"$(openssl rand -hex 16)\",
       \"create_pat\":true,\"pat_expire_in\":365}" | jq -r '.personal_access_token')

echo "$NB_PAT" > /opt/rano/secrets/netbird-pat
chmod 600 /opt/rano/secrets/netbird-pat

HDR=(-H "Authorization: Token $NB_PAT" -H "Content-Type: application/json")

grp() { curl -fsS -X POST "$NB/api/groups" "${HDR[@]}" -d "{\"name\":\"$1\"}" | jq -r '.id'; }
GRP_SERVERS=$(grp k8s-servers)
GRP_WORKERS=$(grp k8s-workers)
GRP_ROUTERS=$(grp kubernetes-routers)
GRP_MLX=$(grp mlx-inference)
GRP_DEVS=$(grp rano-developers)

key() {
  curl -fsS -X POST "$NB/api/setup-keys" "${HDR[@]}" \
    -d "{\"name\":\"$1\",\"type\":\"reusable\",\"ephemeral\":$2,\"auto_groups\":$3}" \
    | jq -r '.key'
}
KEY_K8S=$(key kairos-k8s-nodes false "[\"$GRP_SERVERS\",\"$GRP_WORKERS\"]")
KEY_ROUTER=$(key k8s-netbird-router true "[\"$GRP_ROUTERS\"]")
KEY_MLX=$(key macbook-mlx false "[\"$GRP_MLX\"]")
KEY_DEV=$(key rano-dev false "[\"$GRP_DEVS\"]")

pol() { curl -fsS -X POST "$NB/api/policies" "${HDR[@]}" -d "$1" > /dev/null; }
pol "{\"name\":\"rano-mesh\",\"enabled\":true,\"rules\":[
  {\"name\":\"k8s-internal\",\"enabled\":true,
   \"sources\":[\"$GRP_SERVERS\"],\"destinations\":[\"$GRP_WORKERS\"],
   \"action\":\"accept\",\"bidirectional\":true}]}"
pol "{\"name\":\"mlx-access\",\"enabled\":true,\"rules\":[
  {\"name\":\"cluster-to-mlx\",\"enabled\":true,
   \"sources\":[\"$GRP_SERVERS\",\"$GRP_WORKERS\"],\"destinations\":[\"$GRP_MLX\"],
   \"action\":\"accept\",\"ports\":[\"8080\",\"8081\"]}]}"
pol "{\"name\":\"dev-full\",\"enabled\":true,\"rules\":[
  {\"name\":\"devs\",\"enabled\":true,
   \"sources\":[\"$GRP_DEVS\"],
   \"destinations\":[\"$GRP_SERVERS\",\"$GRP_WORKERS\",\"$GRP_MLX\"],
   \"action\":\"accept\",\"bidirectional\":true}]}"

ACCT=$(curl -fsS "$NB/api/accounts" "${HDR[@]}" | jq -r '.[0].id')
curl -fsS -X POST "$NB/api/routes" "${HDR[@]}" -d \
  "{\"description\":\"k3s pods\",\"network\":\"10.42.0.0/16\",
    \"network_id\":\"rano-pods\",
    \"peer_groups\":[\"$GRP_ROUTERS\"],
    \"groups\":[\"$GRP_MLX\",\"$GRP_DEVS\"],
    \"enabled\":true,\"metric\":9999}"

cat > /opt/rano/secrets/setup-keys.env <<EOF
KEY_K8S=$KEY_K8S
KEY_ROUTER=$KEY_ROUTER
KEY_MLX=$KEY_MLX
KEY_DEV=$KEY_DEV
EOF
chmod 600 /opt/rano/secrets/setup-keys.env
echo "NetBird bootstrap complete. Keys in /opt/rano/secrets/setup-keys.env"
```

### NetBird Agent Network — LLM Providers

NetBird Agent Network (v0.74+) is self-hosted and open-source. It acts as a keyless, identity-aware LLM gateway: a per-account endpoint reachable only from inside the WireGuard overlay. Register providers via dashboard → Agent Network → Providers:

```
Provider 1: rano-localai-cpu
  Type: Custom / OpenAI-compatible
  URL:  http://localai.rano.svc.cluster.local:8080/v1
  Auth: none (mesh-internal)

Provider 2: rano-mlx-primary
  Type: Custom / OpenAI-compatible
  URL:  http://100.64.0.5:8080/v1   (MacBook NetBird IP)
  Auth: none (mesh-internal)

Provider 3: openrouter-frontier
  Type: OpenAI-compatible
  URL:  https://openrouter.ai/api/v1
  API Key: <stored server-side, never in pods>

Provider 4: anthropic-direct
  Type: Anthropic
  API Key: <stored server-side>
```

**Policy:** `rano-swarm-pods` service user → all providers, token cap 2M/day.

---

## Inference Stack: LocalAI, LiteLLM & MacBook MLX

### LocalAI DaemonSet

```yaml
# manifests/inference/localai-daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: localai
  namespace: rano
spec:
  selector:
    matchLabels: { app: localai }
  template:
    metadata:
      labels: { app: localai }
    spec:
      nodeSelector:
        rano.io/inference: localai
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
      initContainers:
        - name: model-puller
          image: alpine:3.19
          command: [sh, -c]
          args:
            - |
              MODEL_DIR=/models
              MODEL_SRV="http://192.168.1.10:9000/models"
              mkdir -p $MODEL_DIR
              # Phi-3-mini (NUC + RPi4, 2.4GB Q4)
              [ -f "$MODEL_DIR/phi3-mini.gguf" ] || \
                wget -q "$MODEL_SRV/Phi-3-mini-4k-instruct-Q4_K_M.gguf" -O "$MODEL_DIR/phi3-mini.gguf"
              # TinyLlama (RPi4 only, 638MB)
              NODE_HW=$(cat /etc/node-hw 2>/dev/null || echo rpi4)
              if [ "$NODE_HW" = "rpi4" ]; then
                [ -f "$MODEL_DIR/tinyllama.gguf" ] || \
                  wget -q "$MODEL_SRV/tinyllama-1.1b-chat-v1.0.Q4_K_M.gguf" -O "$MODEL_DIR/tinyllama.gguf"
              fi
              # Mistral-7B (NUC 16GB+ only, 4.1GB Q4)
              if [ "$NODE_HW" = "nuc" ]; then
                [ -f "$MODEL_DIR/mistral-7b.gguf" ] || \
                  wget -q "$MODEL_SRV/Mistral-7B-Instruct-v0.3-Q4_K_M.gguf" -O "$MODEL_DIR/mistral-7b.gguf"
              fi
          volumeMounts:
            - { name: models, mountPath: /models }
      containers:
        - name: localai
          image: localai/localai:latest-cpu   # multi-arch: amd64 on NUC, arm64 on Pi
          args: [run]
          env:
            - { name: LOCALAI_MODELS_PATH, value: /models }
            - { name: LOCALAI_THREADS, value: "4" }
            - { name: LOCALAI_CONTEXT_SIZE, value: "2048" }
            - { name: LOCALAI_LOG_LEVEL, value: info }
          ports:
            - { containerPort: 8080, name: http }
          resources:
            requests: { cpu: "1", memory: "1Gi" }
            limits: { cpu: "3", memory: "8Gi" }
          volumeMounts:
            - { name: models, mountPath: /models }
          livenessProbe:
            httpGet: { path: /readyz, port: 8080 }
            initialDelaySeconds: 120
            periodSeconds: 30
      volumes:
        - name: models
          hostPath:
            path: /var/lib/localai/models
            type: DirectoryOrCreate
---
apiVersion: v1
kind: Service
metadata: { name: localai, namespace: rano }
spec:
  selector: { app: localai }
  clusterIP: None   # headless — DNS returns all pod IPs
  ports:
    - { port: 8080, name: http }
```

### LiteLLM Gateway (tiered routing)

```yaml
# manifests/inference/litellm-config.yaml
apiVersion: v1
kind: ConfigMap
metadata: { name: litellm-config, namespace: rano }
data:
  config.yaml: |
    model_list:
      # Tier 1 — RPi4 local (lightweight)
      - model_name: tinyllama
        litellm_params:
          model: openai/tinyllama
          api_base: http://localai.rano.svc.cluster.local:8080/v1
          api_key: "none"

      # Tier 2 — NUC local (medium)
      - model_name: phi3-mini
        litellm_params:
          model: openai/phi3-mini
          api_base: http://localai.rano.svc.cluster.local:8080/v1
          api_key: "none"

      - model_name: mistral-7b
        litellm_params:
          model: openai/mistral-7b
          api_base: http://localai.rano.svc.cluster.local:8080/v1
          api_key: "none"

      # Tier 3 — MacBook MLX (heavy, 128GB)
      - model_name: qwen3-35b
        litellm_params:
          model: openai/Qwen3-35B-A3B-4bit
          api_base: os.environ/MLX_BASE_URL
          api_key: "none"

      # Tier 4 — Embedding (MacBook MLX)
      - model_name: rano-embed
        litellm_params:
          model: openai/qwen3-embedding-rano
          api_base: os.environ/MLX_EMBED_URL
          api_key: "none"

      # Tier 5 — OpenRouter cloud frontier
      - model_name: claude-sonnet
        litellm_params:
          model: openrouter/anthropic/claude-sonnet-4
          api_key: os.environ/OPENROUTER_API_KEY

      - model_name: gpt-4o
        litellm_params:
          model: openrouter/openai/gpt-4o
          api_key: os.environ/OPENROUTER_API_KEY

    router_settings:
      context_window_fallbacks:
        - tinyllama: [phi3-mini, mistral-7b]
        - phi3-mini: [mistral-7b, qwen3-35b]
        - mistral-7b: [qwen3-35b, claude-sonnet]
      num_retries: 2
      request_timeout: 60

    general_settings:
      telemetry: false
```

LiteLLM is pinned to v1.72.6. LocalAI serves local GGUF only and does NOT proxy cloud APIs — all cloud routing (OpenRouter, Anthropic) flows through LiteLLM with keys held server-side.

### MacBook MLX launchd Services

```xml
<!-- ~/Library/LaunchAgents/ai.rano.mlx-llm.plist -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0"><dict>
  <key>Label</key><string>ai.rano.mlx-llm</string>
  <key>ProgramArguments</key>
  <array>
    <string>/opt/homebrew/bin/mlx_lm.server</string>
    <string>--host</string><string>0.0.0.0</string>
    <string>--port</string><string>8080</string>
    <string>--model</string><string>mlx-community/Qwen3-35B-A3B-4bit</string>
    <string>--max-tokens</string><string>8192</string>
  </array>
  <key>RunAtLoad</key><true/>
  <key>KeepAlive</key><true/>
  <key>ProcessType</key><string>Background</string>
  <key>StandardOutPath</key><string>/tmp/mlx-llm.log</string>
  <key>StandardErrorPath</key><string>/tmp/mlx-llm.err</string>
</dict></plist>
```

```xml
<!-- ~/Library/LaunchAgents/ai.rano.mlx-embed.plist -->
<!-- Same pattern, port 8081, model: rano-qwen3-embedding-0.6b-mlx -->
```

---

## RANO Swarm Agent Harness

### Architecture

RANO Swarm is a multi-agent closed-loop optimization harness. Agents are Rust/WASM binaries compiled to native via a Rust host runtime. No Python. No interpreted runtime.

Agent lifecycle:

```
OBSERVE → PM counter pull (Ericsson ENM/NETCONF)
REASON  → LLM call via LiteLLM
PLAN    → propose RAN parameter change (MO class, value, rationale)
GATE    → RuVix proof-gated: 2-of-N agents must co-sign
APPLY   → write to ENM via NETCONF/RESTCONF
WITNESS → record in RuVix witness chain (immutable)
REVERT  → if KPI degrades post-apply, REVERT and record as negative signal
```

### RANO Swarm Deployment

```yaml
# manifests/rano-swarm/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: rano-swarm, namespace: rano }
spec:
  replicas: 5
  selector: { matchLabels: { app: rano-swarm } }
  template:
    metadata: { labels: { app: rano-swarm } }
    spec:
      nodeSelector: { rano.io/hw: nuc }    # agents on NUC (x86, more RAM)
      serviceAccountName: rano-agent
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                topologyKey: kubernetes.io/hostname
                labelSelector:
                  matchLabels: { app: rano-swarm }
      containers:
        - name: agent
          image: 192.168.1.10:5000/rano/rano-swarm:latest
          env:
            - { name: INFERENCE_BASE_URL,
                value: http://litellm.rano.svc.cluster.local:4000/v1 }
            - { name: AGENT_NET_ENDPOINT,
                value: https://netbird.rano.local/agent-network/v1 }
            - { name: AGENT_MODEL_LOCAL,   value: phi3-mini }
            - { name: AGENT_MODEL_MEDIUM,  value: mistral-7b }
            - { name: AGENT_MODEL_HEAVY,   value: qwen3-35b }
            - { name: AGENT_MODEL_CLOUD,   value: claude-sonnet }
            - { name: RUVIX_ENDPOINT,
                value: unix:///run/ruvix/nucleus.sock }
            - { name: RUVIX_PROOF_QUORUM,  value: "2" }
            - { name: RUST_LOG,            value: info }
          envFrom:
            - secretRef: { name: rano-swarm-secrets }
          resources:
            requests: { cpu: "500m", memory: "512Mi" }
            limits:   { cpu: "2",    memory: "2Gi" }
          ports:
            - { containerPort: 9090, name: grpc }
            - { containerPort: 9091, name: metrics }
          volumeMounts:
            - { name: config, mountPath: /app/config }
            - { name: memory, mountPath: /data }
            - { name: ruvix-sock, mountPath: /run/ruvix }
      volumes:
        - { name: config, configMap: { name: rano-swarm-config } }
        - { name: memory, persistentVolumeClaim: { claimName: rano-memory-pvc } }
        - { name: ruvix-sock, hostPath: { path: /run/ruvix } }
```

### RANO Research CronJob (nightly ML loop)

```yaml
# manifests/rano-swarm/cronjob-research.yaml
apiVersion: batch/v1
kind: CronJob
metadata: { name: rano-research, namespace: rano }
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: rano-agent
          nodeSelector: { rano.io/hw: nuc }   # job runs on NUC; heavy inference reaches the MacBook via LiteLLM (qwen3-35b), never via k8s scheduling — the MacBook is not a k3s node
          containers:
            - name: research
              image: 192.168.1.10:5000/rano/rano-research:latest
              env:
                - { name: INFERENCE_BASE_URL,
                    value: http://litellm.rano.svc.cluster.local:4000/v1 }
                - { name: RESEARCH_MODEL, value: qwen3-35b }
                - { name: EMBED_MODEL,    value: rano-embed }
              volumeMounts:
                - { name: memory, mountPath: /data }
          restartPolicy: OnFailure
          volumes:
            - { name: memory, persistentVolumeClaim: { claimName: rano-memory-pvc } }
```

---

## RuVix Cognition Kernel Integration

### RuVix Overview (ADR-087)

RuVix is a Rust-first, `no_std` cognition kernel (ruvnet ecosystem) with:

- **6 primitives:** task, capability, region, queue, timer, proof
- **12 syscalls:** task_spawn, cap_grant, region_map, queue_send, queue_recv, timer_wait, rvf_mount, attest_emit, vector_get, vector_put_proved, graph_apply_proved, sensor_subscribe
- **Proof-gated mutation:** every state change requires a cryptographic proof chain before the kernel commits it

**Crates (pre-release, use git deps):**

```toml
[dependencies]
ruvix-rpi-boot = { git = "https://github.com/ruvnet/ruvector", package = "ruvix-rpi-boot" }
ruvix-nucleus   = { git = "https://github.com/ruvnet/ruvector", package = "ruvix-nucleus" }
ruvix-proof     = { git = "https://github.com/ruvnet/ruvector", package = "ruvix-proof" }
```

### Proof-Gated Mutation Flow

```
RANO Swarm pod
  │
  ├─ OBSERVE: pulls PM counters (no proof needed — read-only)
  │
  ├─ PLAN: LLM call via LiteLLM (no proof needed — planning)
  │
  ├─ GATE ──► ruvix-proof::ProofBuilder
  │             agent_alice.sign(proposed_action)
  │             agent_bob.sign(proposed_action)   ← 2-of-N minimum
  │             Proof::build() → fails if < 2 sigs
  │
  ├─ APPLY ──► ruvix-nucleus::vector_put_proved(rvf_object, proof)
  │             kernel validates proof chain before committing
  │             if invalid → rejected at kernel level (not app level)
  │
  └─ WITNESS → proof appended to immutable witness chain
               ruvix-cli witness tail → auditable history
```

### RuVix systemd Unit in Kairos NUC Image

Add to `kairos/Dockerfile.nuc`:

```dockerfile
# RuVix partition binary
FROM rust:1.85 AS ruvix-builder
RUN rustup target add x86_64-unknown-none
WORKDIR /build
COPY ruvix-partition/ .
RUN cargo build --target x86_64-unknown-none --release

# --- final image stage ---
COPY --from=ruvix-builder /build/target/x86_64-unknown-none/release/rano-cognition.bin \
     /usr/lib/rano/ruvix-partition.bin

RUN cat > /etc/systemd/system/ruvix-partition.service <<'EOF'
[Unit]
Description=RuVix RANO Cognition Kernel Partition
After=network-online.target
Before=k3s.service
[Service]
Type=simple
ExecStart=/usr/bin/ruvix-nucleus-loader /usr/lib/rano/ruvix-partition.bin
Restart=on-failure
RestartSec=5
[Install]
WantedBy=multi-user.target
EOF
RUN systemctl enable ruvix-partition.service
```

### ruvix-cli Operator Commands

```bash
# Inspect running tasks on any node via NetBird mesh
ruvix-cli --host 100.64.0.10 tasks list

# Audit all APPLY actions from last 24h
ruvix-cli --host 100.64.0.10 witness tail --since 24h

# Emergency stop: revoke all RAN write capabilities
ruvix-cli --host 100.64.0.10 cap revoke-all

# Grant a scoped capability to an agent
ruvix-cli --host 100.64.0.10 cap grant \
  --agent rano-swarm-pod-3 \
  --resource "NRCellDU::maxUlMcs::enb-lille-001" \
  --rights WRITE \
  --ttl 300
```

---

## Observability: Prometheus, Grafana & Alerts

### Prometheus + Grafana Stack on nuc-0

```yaml
# /opt/netbird/docker-compose.override.yml additions
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./observability/prometheus.yml:/etc/prometheus/prometheus.yml:ro
    ports: ["9090:9090"]

  grafana:
    image: grafana/grafana:latest
    environment:
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASS}
    volumes:
      - grafana-data:/var/lib/grafana
      - ./observability/dashboards:/var/lib/grafana/dashboards:ro
    ports: ["3000:3000"]
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.grafana.rule=Host(`grafana.rano.local`)"
      - "traefik.http.routers.grafana.tls.certresolver=le"
```

### Prometheus Configuration

```yaml
# observability/prometheus.yml
global:
  scrape_interval: 15s
  external_labels: { cluster: rano-swarm, environment: prod }

scrape_configs:
  - job_name: netbird-management
    static_configs: [{ targets: ['netbird-server:9090'] }]
  - job_name: netbird-signal
    static_configs: [{ targets: ['netbird-server:9091'] }]
  - job_name: netbird-relay
    static_configs: [{ targets: ['netbird-server:9092'] }]
  - job_name: rano-swarm
    kubernetes_sd_configs:
      - role: pod
        namespaces: { names: [rano] }
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        action: keep
        regex: rano-swarm
  - job_name: localai
    kubernetes_sd_configs:
      - role: pod
        namespaces: { names: [rano] }
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        action: keep
        regex: localai
```

### Alert Rules

```yaml
# observability/alerts.yml
groups:
  - name: rano
    rules:
      - alert: ClusterNodeDown
        expr: up{job="rano-swarm"} == 0
        for: 2m
        labels: { severity: critical }
      - alert: NetBirdPeerLow
        expr: netbird_management_connected_peers < 14
        for: 5m
        labels: { severity: warning }
      - alert: MLXInferenceUnreachable
        expr: probe_success{target="http://100.64.0.5:8080/v1/models"} == 0
        for: 2m
        labels: { severity: critical }
      - alert: LocalAIModelMissing
        expr: count(kube_pod_status_ready{namespace="rano",pod=~"localai-.*"} == 1) < 14
        for: 5m
        labels: { severity: warning }
```

---

## Security Model: Secrets, RBAC & SafetyGate

### Secrets Hierarchy

```
NEVER in cloud-config plaintext:
  • Cloud API keys (OpenRouter, Anthropic, OpenAI)
  • NetBird admin PAT
  • LiteLLM master key

IN cloud-config (rotatable secrets, OK for bootstrap):
  • K3S_TOKEN (pre-shared cluster join token)
  • NB_SETUP_KEY (NetBird node enrollment key)

IN Kubernetes Secrets:
  • inference-keys (OPENROUTER_API_KEY, ANTHROPIC_API_KEY, MLX_BASE_URL)
  • rano-swarm-secrets (RUVIX_HMAC_SECRET, NB_AGENT_PAT)
  • netbird-setup-key (router pod ephemeral key)

IN macOS Keychain (MacBook):
  • netbird-admin-pat
  • mlx-api-keys (if any)
```

### RBAC (rano-agent ClusterRole binding)

```yaml
# manifests/rano-swarm/rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata: { name: rano-agent, namespace: rano }
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata: { name: rano-agent }
rules:
  - apiGroups: [""]
    resources: [configmaps, secrets]
    verbs: [get, list, watch]
  - apiGroups: [apps]
    resources: [deployments]
    verbs: [get, list, watch, patch]
  - apiGroups: [""]
    resources: [nodes]
    verbs: [get, list, watch]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata: { name: rano-agent }
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: rano-agent
subjects:
  - kind: ServiceAccount
    name: rano-agent
    namespace: rano
```

### SafetyGate (application layer, before RuVix proof)

The SafetyGate is an application-layer quorum check that runs before kernel-level proof enforcement:

```toml
# RANO Swarm config.toml [safety] section
[safety]
hmac_secret_env        = "RUVIX_HMAC_SECRET"
safety_gate_quorum     = 2          # 2-of-N agents must co-sign
revert_window_seconds  = 300        # 5 min post-apply monitoring
kpi_degradation_thresh = 0.05       # 5% KPI drop triggers REVERT
negative_few_shot_db   = "/data/reverted-trials.db"  # fed back to LLM
```

- HMAC quorum via `hmac_secret_env = "RUVIX_HMAC_SECRET"`, 2-of-N signatures before APPLY.
- On REVERT, the trial is stored in a negative few-shot DB (`negative_few_shot_db = "/data/reverted-trials.db"`) and injected as a negative example into the next LLM prompt.

Secrets are never committed: the `31-secrets.yaml` manifest is gitignored and generated by script. API keys live server-side in the NetBird Agent Network, never in pod env.

---

## Phase Plan

Execution contract for the implementing agent swarm (ecosystem-level roadmap and task schema: `01-ecosystem-architecture.md` §23):

- **Task schema** — every task decomposes into atomic subtasks with a machine-verifiable acceptance command. When importing into the controller task graph, map columns onto `Task{id, title, role, acceptanceCriteria[], dependencies[]}`.
- **Dependency DAG** — `P0 → P1 → P2 → P3 → P4 → P5 → P6 → P7`. P8 depends only on P0 (MacBook + NetBird Agent Network) and may run in parallel with P1–P7. P5-T0 (Rust workspace) has no cluster dependency and may start any time after P0.
- **Executor** — tasks marked `HUMAN` need a physical action (USB flash, power-on, EEPROM); everything else is agent-executable from the dev host (kubectl/docker/ssh over the NetBird mesh) once Phase 0 completes. Human tasks: P0-T1(b), P1-T2, P2-T1, P2-T3, P7-T6.

**Corrections vs. source `infra-PRD.md` (re-verify at implementation):**

1. `mlx-community/Qwen3-35B-A3B-4bit` does not exist publicly; the intended model is `mlx-community/Qwen3-30B-A3B-4bit` (Qwen3 MoE, 30B total / 3B active, ~17 GB at 4-bit). The `qwen3-35b` alias is kept in configs until renamed fleet-wide.
2. `mlx_lm.server` is not shipped at `/opt/homebrew/bin` — install `mlx-lm` first (P0-T0) and resolve the real path with `which mlx_lm.server` before writing the launchd plist.
3. The research CronJob schedules on NUC (`rano.io/hw: nuc`); the MacBook is an inference peer only and never a k3s node — heavy inference reaches it through LiteLLM.
4. nuc-0's cloud-config does not set `rano.io/inference=localai`; P1-T6 labels it post-join so the LocalAI DaemonSet reaches 16/16 desired (nuc-1..5 and all RPi4s are labeled by their cloud-configs).
5. Header says "8 acceptance phases"; the plan is nine phases, Phase 0–8.

**Dev-host baseline (verified 2026-07-02 — MacBook Pro, Apple M3 Max, 128 GB unified memory):** installed: `docker`, `netbird`, `ollama 0.31.1` (only `llama3.2` 2 GB pulled), `rtk 0.42.3`, `ruflo 3.16.3` (claude-flow v3 CLI), `ruvector 0.2.33`, `agentdb 3.0.0-alpha.16`, `herdr`. Not installed: `kubectl`, `k3s`, `mlx_lm`, `litellm`, `localai` — P0-T0 installs the dev-host subset; cluster-side components ship via manifests.

### Phase 0 — Infrastructure Bootstrap

**Goal:** nuc-0 running, local registry live, NetBird self-hosted, bootstrap complete, MacBook MLX registered.
**Duration:** 2–3 hours
**Owner:** Cédric (manual steps) → automated thereafter

| ID | Task | Atomic subtasks | Acceptance |
|---|---|---|---|
| P0-T0 | Prepare dev host (MacBook) | (a) `brew install kubectl jq`; (b) `uv tool install mlx-lm` (or `pipx install mlx-lm`); (c) resolve server path: `which mlx_lm.server`; (d) prefetch weights: `huggingface-cli download mlx-community/Qwen3-30B-A3B-4bit` | `kubectl version --client` and `mlx_lm.server --help` both exit 0; model snapshot present in HF cache |
| P0-T1 | Provision nuc-0 initial OS | (a) build nuc-0 ISO (AuroraBoot Phase 2a command); (b) `HUMAN` flash USB, boot nuc-0, one-time install; (c) wait for k3s Ready | `kubectl get nodes rano-nuc-0` = Ready |
| P0-T2 | Start local OCI registry | (a) `docker run -d --name registry --restart always -p 5000:5000 -v /opt/rano/registry:/var/lib/registry registry:2`; (b) write `/etc/docker/daemon.json` insecure-registries; (c) `systemctl restart docker` | `curl http://192.168.1.10:5000/v2/_catalog` returns `{}` |
| P0-T3 | Build + push Kairos images | (a) `docker build` Dockerfile.nuc (amd64); (b) `docker buildx build` Dockerfile.rpi4 (arm64); (c) push both | Both tags listed in `curl http://192.168.1.10:5000/v2/_catalog` |
| P0-T4 | Deploy NetBird stack | (a) write `/opt/netbird/docker-compose.yml`; (b) `docker compose up -d`; (c) wait for management API | `curl https://netbird.rano.local/api/setup` returns 200 |
| P0-T5 | Run bootstrap script | (a) `bash /opt/rano/scripts/netbird-bootstrap.sh`; (b) verify groups/keys/policies/routes created | `/opt/rano/secrets/setup-keys.env` populated with 4 keys |
| P0-T6 | Start model server | (a) `docker run` nginx serving `/models` on :9000; (b) download Phi-3-mini, TinyLlama, Mistral-7B GGUFs into it | `curl http://192.168.1.10:9000/models/` lists all three files |
| P0-T7 | Register MacBook NetBird | `netbird up --setup-key $KEY_MLX --management-url https://192.168.1.10:443` on MacBook | MacBook peer in dashboard, group `mlx-inference` |
| P0-T8 | Start MLX servers | (a) write both launchd plists (LLM :8080, embed :8081) with the path from P0-T0(c); (b) `launchctl load` both | `curl http://localhost:8080/v1/models` returns the Qwen3 model id |

### Phase 1 — NUC Fleet Provisioning

**Goal:** All 6 NUCs running Kairos, joined as k3s HA servers.
**Duration:** 1 hour (mostly automated)

| ID | Task | Subtasks | Acceptance |
|---|---|---|---|
| P1-T1 | Enable AuroraBoot on nuc-0 | Verify `auroraboot-nuc.service` or run manually | `systemctl status auroraboot-nuc` = active |
| P1-T2 | Power on nuc-1..5 | Physical: connect ethernet, power on | Each NUC PXE-boots and displays installer output |
| P1-T3 | Wait for HA cluster formation | Watch `kubectl get nodes` | 6 NUC nodes Ready, all `control-plane` tainted |
| P1-T4 | Verify etcd health | `etcdctl endpoint health` on any NUC | All 6 endpoints healthy |
| P1-T5 | Stop NUC AuroraBoot | `systemctl stop auroraboot-nuc` | No PXE offers on LAN |
| P1-T6 | Complete inference labels | (a) `kubectl label node rano-nuc-0 rano.io/inference=localai` (nuc-1..5 are labeled by cloud-config); (b) verify fleet labels | `kubectl get nodes -l rano.io/inference=localai --no-headers | wc -l` = 6 (16 after Phase 2) |

### Phase 2 — RPi4 Fleet Provisioning

**Goal:** All 10 RPi4s running Kairos, joined as k3s workers.
**Duration:** 30 min (automated)

| ID | Task | Subtasks | Acceptance |
|---|---|---|---|
| P2-T1 | Update RPi4 EEPROMs | Raspberry Pi Imager → Bootloader → Network Boot; one-time per Pi | Pi boots from network (no SD required) |
| P2-T2 | Enable RPi4 AuroraBoot | `docker run ... rano/kairos-rpi4:1.0.0 --cloud-config rpi4-worker.yaml` | AuroraBoot listening, ProxyDHCP active |
| P2-T3 | Power on all 10 RPi4s | Physical power-on | Each Pi PXE-boots, installs, reboots |
| P2-T4 | Verify cluster | `kubectl get nodes` | 16 total nodes (6 NUC + 10 RPi4) = Ready |
| P2-T5 | Stop RPi4 AuroraBoot | Ctrl+C or `systemctl stop auroraboot-rpi4` | Done |

### Phase 3 — MetalLB + NetBird Router

**Goal:** LoadBalancer IPs working; pod CIDR routed to MacBook via NetBird.
**Duration:** 15 min

| ID | Task | Subtasks | Acceptance |
|---|---|---|---|
| P3-T1 | Deploy MetalLB | `kubectl apply -f metallb-native.yaml`; wait for pods | MetalLB system pods Running |
| P3-T2 | Apply IP pool | `kubectl apply -f metallb-pool.yaml` | IPAddressPool `rano-pool` created |
| P3-T3 | Deploy NetBird router | `kubectl apply -f netbird-router.yaml` | Pod Running; `netbird status` shows connected |
| P3-T4 | Verify pod CIDR route | In NetBird dashboard → Routes | Route `10.42.0.0/16` visible, distributed to `mlx-inference` group |
| P3-T5 | Test from MacBook | `curl http://10.42.x.x:8080` (any pod IP) | Response received (NetBird mesh routing works) |

### Phase 4 — Inference Stack

**Goal:** LocalAI on all 16 nodes, LiteLLM routing, MLX MacBook reachable.
**Duration:** 2 hours (model download dominant)

| ID | Task | Subtasks | Acceptance |
|---|---|---|---|
| P4-T1 | Apply LocalAI DaemonSet | `kubectl apply -f localai-daemonset.yaml` | DaemonSet shows 16 desired, 0 ready (init downloading) |
| P4-T2 | Wait for model downloads | Watch init container logs | All 16 pods transition to Running |
| P4-T3 | Smoke test LocalAI | `curl http://localai.rano.svc/v1/models` | Returns model list per node |
| P4-T4 | Apply LiteLLM ConfigMap + Deployment | `kubectl apply -f litellm-*.yaml` | LiteLLM pod Running |
| P4-T5 | Test tier routing | Request `model: phi3-mini`; `model: qwen3-35b`; `model: claude-sonnet` | Each routes to correct backend |
| P4-T6 | Register NetBird Agent Network providers | Dashboard → Agent Network → Providers | 4 providers active |

### Phase 5 — RANO Swarm Agent Harness

**Goal:** RANO Swarm pods running, calling LiteLLM, metric endpoints live.
**Duration:** 1 hour

| ID | Task | Subtasks | Acceptance |
|---|---|---|---|
| P5-T0 | Implement `rano-agent` Rust workspace | (a) scaffold cargo workspace per File Tree (`src/rano-agent/{agent,observe,reason,gate,apply,witness}` crates); (b) implement aggregates from the DDD Model section (AgentSession, RanAction, ProofRecord, KpiSnapshot, RevertedTrial); (c) implement OBSERVE→REASON→PLAN→GATE→APPLY→WITNESS→REVERT lifecycle against mocked ENM/LiteLLM (London-School TDD, mock-first); (d) HTTP surface `:9090 /v1/agent/invoke` + `:9091 /metrics`; (e) read `config.toml` `[safety]` section | `cargo test --workspace` passes; `cargo build --release` produces static binary; no cluster required |
| P5-T1 | Build + push rano-swarm image | (a) multi-stage Dockerfile from P5-T0 binary; (b) `docker build` + `docker push` to `192.168.1.10:5000/rano/rano-swarm` | Image tag in registry catalog |
| P5-T2 | Apply RBAC + Secrets + ConfigMap | `kubectl apply -f rbac.yaml secrets.yaml configmap.yaml` | Resources created |
| P5-T3 | Apply rano-swarm Deployment | `kubectl apply -f deployment.yaml` | 5 pods Running |
| P5-T4 | Test agent inference call | `curl http://rano-swarm.rano.svc:9090/v1/agent/invoke` | Response with model used, latency |
| P5-T5 | Verify metrics | `curl http://rano-swarm.rano.svc:9091/metrics` | Prometheus metrics visible |
| P5-T6 | Deploy Research CronJob | `kubectl apply -f cronjob-research.yaml` | CronJob created, dry-run succeeds |

### Phase 6 — RuVix Cognition Kernel

**Goal:** RuVix partition running on NUC nodes; proof chain active for APPLY actions.
**Duration:** 3–5 days (Rust development)

| ID | Task | Subtasks | Acceptance |
|---|---|---|---|
| P6-T1 | Add ruvix deps to Cargo.toml | Add git deps for ruvix-nucleus, ruvix-proof | `cargo build` succeeds |
| P6-T2 | Implement ProofBuilder in RANO agent | Wrap APPLY action with 2-sig proof construction | Unit tests pass |
| P6-T3 | Integrate ruvix-nucleus syscalls | Replace HMAC SafetyGate with kernel-level `vector_put_proved` | Integration test: reject < 2 sigs |
| P6-T4 | Build ruvix-partition binary | Compile to `x86_64-unknown-none` | Binary < 1MB, no std |
| P6-T5 | Add ruvix-partition to Dockerfile.nuc | COPY binary + systemd unit; enable | `systemctl status ruvix-partition` = active |
| P6-T6 | Test end-to-end proof gate | APPLY action with 1 sig → rejected; 2 sigs → accepted | Kernel error vs success |
| P6-T7 | Verify witness chain | `ruvix-cli witness tail --since 1h` | All APPLY events visible, linked |

### Phase 7 — Observability & Hardening

**Goal:** Grafana dashboards live; alerts firing; secrets rotated; OTA upgrade tested.
**Duration:** 1 day

| ID | Task | Subtasks | Acceptance |
|---|---|---|---|
| P7-T0 | Apply observability manifests | (a) write `/opt/netbird/docker-compose.override.yml` (prometheus + grafana); (b) write `observability/prometheus.yml` + `alerts.yml`; (c) `docker compose up -d` on nuc-0 | `curl -s http://192.168.1.10:9090/-/ready` = 200; `curl -s http://192.168.1.10:3000/api/health` reports ok |
| P7-T1 | Import NetBird Grafana dashboards | Download management/signal/relay JSON; import | Dashboards show live data |
| P7-T2 | Import RANO Swarm dashboard | Custom dashboard: agent calls, model usage, KPI trends | Dashboard visible |
| P7-T3 | Configure alert receivers | Prometheus → alertmanager → email/Slack | Test alert fires and delivers |
| P7-T4 | Rotate K3S_TOKEN | Re-provision one worker with new token | Worker rejoins cleanly |
| P7-T5 | Test OTA upgrade | `kairos-agent upgrade --image 192.168.1.10:5000/rano/kairos-rpi4:1.1.0` on one Pi | Node upgrades, rejoins, no data loss |
| P7-T6 | Test node failure recovery | Power off nuc-2 | Pods reschedule within 60s |
| P7-T7 | Validate data sovereignty | `tcpdump` on LAN: no PM data leaving mesh | Zero external PM data leakage |

### Phase 8 — Claude Code + RANO Development Workflow

**Goal:** Claude Code routes through NetBird Agent Network; RANO development uses local MLX.
**Duration:** 2 hours

| ID | Task | Subtasks | Acceptance |
|---|---|---|---|
| P8-T1 | Configure Claude Code settings | Write `~/.claude/settings.json` with `ANTHROPIC_BASE_URL` | `claude` CLI uses NetBird AN endpoint |
| P8-T2 | Verify keyless routing | `claude "what is 2+2"` → check NetBird AN logs | Request attributed to MacBook identity |
| P8-T3 | Create `rano-claude` shell function | zshrc alias using local MLX via AN | `rano-claude "list cluster nodes"` works |
| P8-T4 | Add Claude Code service user in NetBird | Service user `claude-code-dev`; PAT; policy | Claude Code requests in AN usage logs |

---

## Deep Research Prompts

These prompts are designed for execution by a deep research agent (Claude Research, GPT-4o with search, etc.) to refine and improve each phase before implementation.

### RP-0: Infrastructure Bootstrap Research

```
You are a senior platform engineer preparing to build the RANO Swarm
infrastructure bootstrap layer.

CONTEXT:
- 6× Intel NUC8i5BEH (x86_64, 16-32GB RAM, 256GB SSD)
- Running Kairos OS (immutable Linux, kairos-init v0.14.6)
- k3s v1.31.5+k3s1 embedded in the Kairos image
- NetBird self-hosted v0.74+ on nuc-0
- Local OCI registry at 192.168.1.10:5000

RESEARCH TASKS:
1. Verify the exact kairos-init flag syntax for `--model generic` + `--provider k3s`
   in kairos-init v0.14.6. Confirm the `--provider-k3s-version` accepts
   `v1.31.5+k3s1` format. Source: kairos.io/docs/reference/kairos-factory

2. Research the current NetBird v0.74+ combined container API for:
   - POST /api/setup endpoint (automated first-owner creation)
   - Group management: POST /api/groups
   - Setup key creation: POST /api/setup-keys with ephemeral:true
   - Network route creation: POST /api/routes with peer_groups array
   Source: docs.netbird.io/api

3. Research Traefik v3.1 configuration for NetBird reverse proxy:
   - h2c:// backend scheme for gRPC Management and Signal services
   - TLS-ALPN-01 challenge for wildcard certs (needed for netbird expose)
   - WebSocket passthrough for Relay service (/relay/* path)
   Source: docs.netbird.io/selfhosted/external-reverse-proxy

4. Verify Docker registry:2 configuration for insecure-registries with NUC
   fleet. What is the correct daemon.json format? Does Kairos's immutable
   rootfs require any special handling for /etc/docker/daemon.json?

5. Research AuroraBoot ProxyDHCP port conflict resolution when running two
   consecutive AuroraBoot instances on the same Linux host. Is there a
   `--dhcp-port` flag? Can we use the `start-pixie` subcommand to separate
   ProxyDHCP from the artifact HTTP server?

OUTPUT: Produce corrected, verified configurations for all 5 areas above.
Flag any discrepancies from the current PRD v3.2 and provide exact fixes.
```

### RP-1: Kairos Fleet Netboot Research

```
You are a Kairos OS expert preparing zero-touch netboot provisioning for
6 Intel NUC8i5BEH (amd64) and 10 Raspberry Pi 4 (arm64) nodes.

CONTEXT:
- Kairos v4.1.x + kairos-init v0.14.6
- AuroraBoot latest (quay.io/kairos/auroraboot)
- k3s v1.31.5+k3s1
- nuc-0 is the AuroraBoot server (Linux, --net host Docker)
- RPi4s need EEPROM updated for network boot priority

RESEARCH TASKS:
1. Confirm the exact AuroraBoot CLI syntax for:
   - Netboot serving mode (--net host, --cloud-config, --set container_image)
   - The difference between serving from a local Docker image
     (docker://registry.local/image:tag) vs a pulled OCI image
   Source: kairos.io/docs/reference/auroraboot

2. Research the Kairos P2P provider (network_token) vs explicit
   k3s:/k3s-agent: cloud-config approach. For a 6-NUC HA control plane,
   does the P2P provider support designating specific nodes as servers
   vs workers? Or must we use explicit k3s: blocks?
   Source: kairos.io/docs/installation/p2p

3. Research the `stages.provider-kairos.bootstrap.after.k3s-ready` stage
   hook mechanism. What systemd unit file is required in the Kairos image?
   Does it work for both k3s server AND k3s-agent nodes? What happens if
   the stage times out?
   Source: kairos.io/docs/examples/k3s-stages

4. For RPi4 with kairos-init --model rpi4: what specific firmware, U-Boot
   configuration, and kernel cmdline parameters does -m rpi4 install
   compared to -m generic? Is cgroup memory enable still required in
   cmdline, or does kairos-init handle it?

5. Research Kairos persistent paths for /var/lib/localai/models across
   reboots and upgrades. What is the correct `extra-dirs-rootfs` syntax?
   Does it persist across `kairos-agent upgrade` operations?
   Source: kairos.io/docs/reference/configuration

OUTPUT: Produce verified cloud-config snippets and Dockerfile additions
for all 5 areas. Flag any current PRD assumptions that are incorrect.
```

### RP-2: LocalAI + LiteLLM Inference Stack Research

```
You are an AI infrastructure engineer designing a multi-tier inference
stack for a 16-node Kubernetes cluster (6× x86_64 NUC + 10× ARM64 RPi4)
with a MacBook M3 Max as an external inference peer.

CONTEXT:
- LocalAI latest-cpu (multi-arch: amd64 + arm64 from same image tag)
- LiteLLM v1.72.6 (pinned, avoid 1.82.7/1.82.8)
- mlx-lm.server on MacBook M3 Max (OpenAI-compatible, port 8080)
- OpenRouter as cloud frontier gateway
- NetBird Agent Network v0.74+ as keyless LLM proxy

RESEARCH TASKS:
1. Confirm LocalAI multi-arch support: does `localai/localai:latest-cpu`
   manifest list include both linux/amd64 and linux/arm64? What is the
   correct image tag for production (pinned digest recommended)?
   Source: hub.docker.com/r/localai/localai

2. Research LocalAI model config YAML format for llama-cpp backend:
   - Required fields: name, backend, parameters.model
   - Optional: context_size, threads, mmap
   - How to pre-configure models without downloading at runtime
     (using PRELOAD_MODELS env or model config files in /models)
   Source: localai.io/docs/getting-started/customizing-the-model

3. Research LiteLLM router_settings for:
   - context_window_fallbacks syntax (exact YAML format)
   - model health checking (is there a health_check interval?)
   - routing to a headless k8s Service (returns multiple pod IPs via DNS)
   Does LiteLLM do client-side load balancing across headless service IPs?
   Source: docs.litellm.ai/docs/routing

4. Research mlx_lm.server API completeness:
   - Does it support /v1/embeddings? Or only /v1/chat/completions?
   - Is there a --max-tokens server-level default, or only per-request?
   - Does it support concurrent requests, or is it single-threaded?
   Source: github.com/ml-explore/mlx-lm (SERVER.md)

5. Research NetBird Agent Network provider configuration for self-hosted:
   - Is the Agent Network feature available in the open-source self-hosted
     NetBird v0.74+, or only in NetBird Cloud?
   - If self-hosted: where is the Agent Network config in management.json
     or config.yaml? What is the per-account endpoint URL structure?
   Source: docs.netbird.io/agent-network, github.com/netbirdio/netbird

OUTPUT: Produce corrected inference stack manifests addressing all 5 areas.
Include exact image digests for LocalAI and LiteLLM.
```

### RP-3: RuVix Integration Research

```
You are a Rust systems engineer integrating the RuVix Cognition Kernel
(ADR-087, ruvnet ecosystem) into the RANO Swarm agent harness.

CONTEXT:
- RuVix crates: ruvix-nucleus, ruvix-proof, ruvix-rpi-boot, ruvix-cli
- Crates are pre-release (git dependencies from ruvnet/RuVector)
- Target: x86_64-unknown-none (NUC bare partition) + integration via socket
- RANO Swarm agents are Rust/WASM running as k8s pods on NUC nodes

RESEARCH TASKS:
1. Research ruvix-nucleus API from ADR-087 source:
   - Exact CapHandle, CapRights types
   - cap_grant syscall signature
   - vector_put_proved syscall: what proof type does it accept?
   Source: github.com/ruvnet/ruvector (ADR-087 file + nucleus crate source)

2. Research ruvix-proof ProofBuilder API:
   - How are witness signatures added (.witness(sig) pattern)?
   - What signing key format is used (Ed25519 assumed)?
   - How does prev_chain linkage work?
   - What is the proof serialization format (RVF? custom bytes?)?
   Source: github.com/ruvnet/ruvector (proof crate source)

3. Research the ruvix-partition boot model for x86_64:
   - ruvix-rpi-boot is for ARM64/RPi. What is the equivalent for x86_64?
   - Is there a ruvix-x86-boot crate or does the partition run differently?
   - Can the cognition kernel partition run as a Linux userspace process
     (via a loader) rather than bare metal? This would simplify NUC integration.
   Source: github.com/ruvnet/rvm + ADR-087

4. Research the IPC mechanism between RANO Swarm k8s pods and the RuVix
   partition running on the same node:
   - Is gRPC over Unix socket the documented interface?
   - What protobuf service definition does ruvix-nucleus expose?
   - Is there a ruvix-client crate for pod-side integration?
   Source: github.com/ruvnet/ruvector

5. Research ruvix-cli binary availability:
   - Build instructions from source (cargo install path)
   - Connection protocol (TCP? Unix socket? gRPC?)
   - Authentication model (any credential needed to connect to a node?)
   Source: crates.io/crates/ruvix-cli + github.com/ruvnet/ruvector

OUTPUT: Produce corrected RuVix integration code and Dockerfile additions.
Flag any assumptions in the RuVix integration section that need correction
based on actual RuVix API surface found in source code.
```

### RP-4: RANO Swarm Agent Architecture Research

```
You are a distributed systems architect designing the RANO Swarm agent
runtime for closed-loop RAN optimization at Orange France.

CONTEXT:
- 5 RANO Swarm agent pods (Rust/WASM, k3s, NUC nodes)
- Inference via LiteLLM (LocalAI / MLX / OpenRouter)
- RAN data: Ericsson ENM NETCONF/RESTCONF, PM counters (15-min granularity)
- Managed Objects: EUtranCellFDD, NRCellDU, GNBDUFunction (Ericsson)
- Memory: sqlite-vec HNSW index + SQLite for agent state
- Safety: RuVix proof-gated mutations, REVERTED trial negative few-shot

RESEARCH TASKS:
1. Research Ericsson ENM NETCONF/RESTCONF API for parameter write:
   - What RPC is used to set a ManagedObject attribute?
   - Authentication: certificate-based or username/password?
   - Is there a Rust NETCONF client crate (netconf-client)?
   - How to batch parameter changes (avoid N individual RPCs)?

2. Research the ruv-swarm / ruflo agent coordination model:
   - How do multiple RANO Swarm pods coordinate without a central broker?
   - Does ruflo support a 2-of-N consensus pattern for APPLY gating?
   - What is the ruv-swarm agent messaging protocol?
   Source: github.com/ruvnet/ruflo + github.com/ruvnet/ruv-swarm

3. Research sqlite-vec for the RANO vector memory:
   - What embedding dimension does Qwen3-Embedding-0.6B produce?
   - How to configure HNSW index parameters (M, ef_construction) for
     a corpus of 6164+ Ericsson parameter descriptions?
   - Is there a Rust sqlite-vec binding?
   Source: github.com/asg017/sqlite-vec

4. Research REVERTED trial negative few-shot injection:
   - How to store a REVERTED trial (action, pre-KPI, post-KPI, reason)
     in SQLite and inject it as a negative example in the next LLM prompt?
   - Reflexion pattern (Shinn et al. 2023) vs Voyager pattern — which
     fits better for iterative RAN optimization?

5. Research DSPy/MIPROv2 for RANO agent prompt optimization:
   - Can DSPy optimize prompts offline (on MacBook MLX) and deploy
     optimized prompt templates to RANO Swarm pods via ConfigMap update?
   - What is the DSPy Teleprompter API for this pattern?
   Source: github.com/stanfordnlp/dspy

OUTPUT: Produce a detailed RANO agent state machine diagram (ASCII) and
Rust struct definitions for AgentState, RanAction, ProofRecord, KpiSnapshot.
```

---

## ADR Skeletons

The following ADR stubs define every significant architectural decision. Each should be expanded into a full ADR document.

```
ADR-001: Kairos as immutable OS for all edge nodes
  Status: Accepted
  Decision: Use Kairos (kairos-init) for all NUC and RPi4 nodes
  Consequences: No runtime SSH root writes; OTA via kairos-agent upgrade

ADR-002: k3s as Kubernetes distribution
  Status: Accepted
  Decision: k3s with embedded etcd, 6-server HA
  Consequences: No external etcd; quorum tolerance = 2 server failures

ADR-003: AuroraBoot sequential netboot strategy
  Status: Accepted
  Decision: Two sequential AuroraBoot runs (NUC image then RPi4 image)
  Consequences: Single ProxyDHCP host avoids port 67 conflict; sequential

ADR-004: NetBird WireGuard mesh as zero-trust network
  Status: Accepted
  Decision: Self-hosted NetBird on nuc-0; all traffic via WireGuard overlay
  Consequences: No public exposure; all API keys server-side in NetBird AN

ADR-005: LocalAI DaemonSet for on-cluster inference
  Status: Accepted
  Decision: DaemonSet with nodeSelector rano.io/inference=localai
  Consequences: 16 LocalAI instances; multi-arch from single image tag

ADR-006: LiteLLM as unified inference gateway (not LocalAI)
  Status: Accepted
  Decision: LiteLLM routes to LocalAI + MLX + OpenRouter; LocalAI is NOT a proxy
  Consequences: Cloud routing requires LiteLLM; LocalAI handles local GGUF only

ADR-007: MacBook M3 Max as inference-only peer
  Status: Accepted
  Decision: MacBook has no k3s role; purely MLX inference + NetBird peer
  Consequences: No cluster state on MacBook; launchd manages MLX lifecycle

ADR-008: RuVix proof-gated mutations for RAN parameters
  Status: Proposed
  Decision: Every APPLY action requires ruvix-proof chain with 2-of-N signatures
  Consequences: Kernel-level enforcement; no APPLY possible without quorum

ADR-009: Pre-shared K3S_TOKEN embedded in cloud-config
  Status: Accepted
  Decision: Generate token before first boot; embed in all cloud-configs
  Consequences: Token in OEM partition (rotatable); no manual copy step

ADR-010: OpenRouter as cloud frontier model gateway
  Status: Accepted
  Decision: OpenRouter via LiteLLM for cloud models; key server-side in NetBird AN
  Consequences: No API key in pods; per-request model routing

ADR-011: sqlite-vec HNSW as agent vector memory
  Status: Accepted
  Decision: sqlite-vec with Qwen3-Embedding-0.6B-RANO domain-adapted embeddings
  Consequences: Zero external vector DB; local-first; Rust binding available

ADR-012: No Python in agent runtime
  Status: Accepted
  Decision: RANO Swarm containers are Rust/WASM binaries via Rust host runtime
  Consequences: Eliminates Python supply-chain risk; WASM sandbox for agents
```

---

## DDD Model

### Bounded Contexts

```
┌──────────────────────────────────────────────────────────────────┐
│  RANO Swarm Domain Map                                            │
│                                                                   │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────┐  │
│  │  Infrastructure  │    │  Inference       │    │  RAN Domain │  │
│  │  Context         │    │  Context         │    │  Context    │  │
│  │                  │    │                  │    │             │  │
│  │  KairosNode      │    │  InferenceTask   │    │  RanCell    │  │
│  │  ClusterPeer     │    │  ModelTier       │    │  MoParam    │  │
│  │  NetBirdMesh     │    │  InferenceResult │    │  PmCounter  │  │
│  │  OciRegistry     │    │  EmbeddingVector │    │  KpiMetric  │  │
│  └────────┬─────────┘    └────────┬─────────┘    └──────┬──────┘  │
│           │                       │                      │         │
│  ┌────────▼───────────────────────▼──────────────────────▼──────┐ │
│  │                    Agent Context                               │ │
│  │                                                               │ │
│  │  AgentSwarm     ← orchestrates N agents                      │ │
│  │  AgentSession   ← single agent lifecycle                     │ │
│  │  RanAction      ← proposed parameter change                  │ │
│  │  ProofRecord    ← RuVix cryptographic proof                  │ │
│  │  WitnessChain   ← immutable audit log                        │ │
│  │  SafetyGate     ← quorum-based approval                      │ │
│  │  KpiSnapshot    ← pre/post apply KPI values                  │ │
│  │  RevertedTrial  ← negative few-shot signal                   │ │
│  └───────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────┘
```

### Core Aggregates

```rust
// Agent Context — core aggregates

pub struct AgentSession {
    pub id: AgentId,
    pub state: AgentState,
    pub memory: AgentMemory,           // sqlite-vec HNSW index
    pub current_action: Option<RanAction>,
    pub proof: Option<ProofRecord>,
}

pub enum AgentState {
    Idle,
    Observing,
    Reasoning { llm_call_id: Uuid },
    Planning  { draft: RanAction },
    Gating    { sigs_collected: u8, sigs_required: u8 },
    Applying  { action: RanAction, proof: ProofRecord },
    Monitoring { baseline_kpi: KpiSnapshot, apply_ts: DateTime<Utc> },
    Reverting { reason: RevertReason },
    Witnessing,
}

pub struct RanAction {
    pub id: ActionId,
    pub mo_class: MoClass,           // NRCellDU, EUtranCellFDD, ...
    pub mo_fdn: String,              // Full Distinguished Name
    pub parameter: String,           // e.g. maxUlMcs
    pub current_value: ParamValue,
    pub proposed_value: ParamValue,
    pub rationale: String,
    pub confidence: f32,
    pub model_used: ModelId,
}

pub struct ProofRecord {
    pub action_hash: [u8; 32],
    pub signatures: Vec<AgentSignature>,   // Ed25519
    pub prev_witness_hash: [u8; 32],
    pub timestamp: DateTime<Utc>,
    pub kernel_accepted: bool,
}

pub struct KpiSnapshot {
    pub cell_fdn: String,
    pub timestamp: DateTime<Utc>,
    pub values: HashMap<String, f64>,   // pmRrcConnEstabSucc, pmErabDrop, ...
}

pub struct RevertedTrial {
    pub action: RanAction,
    pub proof: ProofRecord,
    pub pre_kpi: KpiSnapshot,
    pub post_kpi: KpiSnapshot,
    pub delta: f64,                     // KPI change %
    pub revert_reason: RevertReason,
    pub negative_signal_injected: bool, // used as few-shot in next LLM call
}
```

### Domain Events

```rust
pub enum RanoDomainEvent {
    // Infrastructure
    NodeJoinedCluster     { node_id: NodeId, hw_type: HwType, timestamp: DateTime<Utc> },
    NodeLeftCluster       { node_id: NodeId, reason: String },
    InferenceNodeReady    { node_id: NodeId, models: Vec<ModelId> },

    // Inference
    LlmCallStarted        { session_id: AgentId, model: ModelId, prompt_tokens: u32 },
    LlmCallCompleted      { session_id: AgentId, completion_tokens: u32, latency_ms: u64 },
    InferenceFallback     { from_model: ModelId, to_model: ModelId, reason: String },

    // Agent
    ObservationCycleStarted  { session_id: AgentId },
    ActionProposed           { session_id: AgentId, action: RanAction },
    ProofQuorumReached       { action_id: ActionId, signers: Vec<AgentId> },
    ProofQuorumFailed        { action_id: ActionId, sigs: u8, required: u8 },
    ActionApplied            { action_id: ActionId, proof: ProofRecord },
    ActionReverted           { action_id: ActionId, reason: RevertReason },
    WitnessChainExtended     { action_id: ActionId, chain_tip: [u8; 32] },
    NegativeSignalInjected   { trial_id: ActionId, next_session_id: AgentId },
}
```

---

## File Tree & Justfile

### Complete File Tree

```
rano-swarm/
├── kairos/
│   ├── Dockerfile.nuc                  # amd64, --model generic, k3s server
│   ├── Dockerfile.rpi4                 # arm64, --model rpi4, k3s worker
│   ├── Dockerfile.nuc.ruvix            # nuc image + RuVix partition binary
│   ├── systemd/
│   │   └── k3s-ready.service           # fires k3s-ready stage hook
│   └── ruvix-partition/                # Rust project: x86_64-unknown-none
│       ├── Cargo.toml                  # deps: ruvix-nucleus, ruvix-proof
│       └── src/main.rs
│
├── cloud-configs/
│   ├── nuc-0.yaml                      # cluster-init + AuroraBoot + NetBird host
│   ├── nuc-server.yaml                 # nuc-1..5 HA servers
│   └── rpi4-worker.yaml                # all 10 RPi4 workers
│
├── netbird/
│   ├── docker-compose.yml              # NetBird + Traefik + Coturn + Prometheus + Grafana
│   ├── docker-compose.override.yml     # dev overrides
│   └── observability/
│       ├── prometheus.yml
│       ├── alerts.yml
│       └── grafana/dashboards/
│           ├── management.json         # from netbirdio/netbird repo
│           ├── signal.json
│           ├── relay.json
│           └── rano-swarm.json         # custom RANO dashboard
│
├── scripts/
│   ├── netbird-bootstrap.sh            # full API automation
│   ├── gen-token.sh                    # K3S_TOKEN generator
│   └── model-server-setup.sh           # download GGUFs to nuc-0
│
├── manifests/
│   ├── 00-namespace.yaml
│   ├── 05-metallb-native.yaml          # downloaded
│   ├── 06-metallb-pool.yaml
│   ├── 10-netbird-router.yaml
│   ├── inference/
│   │   ├── 20-localai-daemonset.yaml
│   │   ├── 21-litellm-config.yaml
│   │   ├── 22-litellm-deployment.yaml
│   │   └── 23-mlx-externalname.yaml
│   ├── rano-swarm/
│   │   ├── 30-rbac.yaml
│   │   ├── 31-secrets.yaml             # gitignored, generated by script
│   │   ├── 32-configmap.yaml
│   │   ├── 33-pvc.yaml
│   │   ├── 34-deployment.yaml
│   │   ├── 35-service.yaml
│   │   └── 36-cronjob-research.yaml
│   └── ruvix/
│       └── 40-ruvix-socket-hostpath.yaml
│
├── macbook/
│   ├── ai.rano.mlx-llm.plist           # launchd MLX LLM server
│   ├── ai.rano.mlx-embed.plist         # launchd MLX embedding server
│   ├── claude-settings.json            # Claude Code → NetBird AN
│   └── zshrc-snippets.sh               # rano-claude / cloud-claude
│
├── src/                                # RANO Swarm Rust workspace
│   ├── Cargo.toml                      # workspace
│   ├── rano-agent/                     # core agent binary (WASM-compilable)
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── main.rs
│   │       ├── agent.rs                # AgentSession state machine
│   │       ├── inference.rs            # LiteLLM client
│   │       ├── ran_action.rs           # RanAction + NETCONF writer
│   │       ├── proof.rs                # ruvix-proof integration
│   │       ├── memory.rs               # sqlite-vec HNSW
│   │       └── safety_gate.rs          # SafetyGate + RevertedTrial
│   ├── rano-research/                  # nightly research binary
│   └── rano-cli/                       # operator CLI
│
├── docker/
│   ├── rano-swarm.Dockerfile           # multi-stage: build Rust → minimal runtime
│   └── rano-research.Dockerfile
│
├── adr/                                # generated from ADR skeletons
│   ├── ADR-001-kairos-immutable-os.md
│   ├── ADR-002-k3s-ha.md
│   └── ...
│
├── justfile                            # full pipeline
└── README.md
```

### Complete Justfile

```makefile
set shell := ["bash", "-uc"]

VERSION     := "1.0.0"
K3S_VERSION := "v1.31.5+k3s1"
NUC0_IP     := "192.168.1.10"
NUC0_USER   := "kairos"
REGISTRY    := "192.168.1.10:5000"
KUBECONFIG  := "./kubeconfig"

# ── Token management ──────────────────────────────────────────────
gen-token:
    @echo "$(openssl rand -hex 16)" > .cluster-token
    @echo "Token saved. Inject into cloud-configs before building."

inject-token:
    @TOKEN=$(cat .cluster-token); \
    sed -i "s/<SHARED_TOKEN>/$TOKEN/g" cloud-configs/*.yaml; \
    echo "Token injected into all cloud-configs"

# ── Image builds ──────────────────────────────────────────────────
build-nuc:
    docker build --platform linux/amd64 \
      --build-arg VERSION={{VERSION}} \
      --build-arg K3S_VERSION={{K3S_VERSION}} \
      -t {{REGISTRY}}/rano/kairos-nuc:{{VERSION}} \
      -f kairos/Dockerfile.nuc kairos/
    docker push {{REGISTRY}}/rano/kairos-nuc:{{VERSION}}

build-rpi4:
    docker buildx build --platform linux/arm64 \
      --build-arg VERSION={{VERSION}} \
      --build-arg K3S_VERSION={{K3S_VERSION}} \
      -t {{REGISTRY}}/rano/kairos-rpi4:{{VERSION}} \
      -f kairos/Dockerfile.rpi4 --load kairos/
    docker push {{REGISTRY}}/rano/kairos-rpi4:{{VERSION}}

build-swarm:
    docker build \
      -t {{REGISTRY}}/rano/rano-swarm:latest \
      -f docker/rano-swarm.Dockerfile src/
    docker push {{REGISTRY}}/rano/rano-swarm:latest

build-all: build-nuc build-rpi4 build-swarm

# ── nuc-0 bootstrap (USB ISO, one-time) ──────────────────────────
build-nuc0-iso:
    docker run --rm \
      -v "$PWD/cloud-configs/nuc-0.yaml":/c.yaml \
      -v "$PWD/build":/tmp/ab \
      -v /var/run/docker.sock:/var/run/docker.sock \
      quay.io/kairos/auroraboot \
      --set container_image=docker://{{REGISTRY}}/rano/kairos-nuc:{{VERSION}} \
      --set disable_http_server=true \
      --set disable_netboot=true \
      --set state_dir=/tmp/ab \
      --cloud-config /c.yaml
    @echo "Flash build/kairos.iso to USB → boot nuc-0 once"

# ── NetBird bootstrap ─────────────────────────────────────────────
start-netbird:
    ssh {{NUC0_USER}}@{{NUC0_IP}} \
      "cd /opt/netbird && docker compose up -d"

bootstrap-netbird:
    ssh {{NUC0_USER}}@{{NUC0_IP}} \
      "bash /opt/rano/scripts/netbird-bootstrap.sh"

# ── Fleet provisioning ────────────────────────────────────────────
provision-nucs:
    @echo "Power on nuc-1..5 AFTER this command starts"
    ssh {{NUC0_USER}}@{{NUC0_IP}} \
      "docker run --rm -ti --net host \
       -v /var/run/docker.sock:/var/run/docker.sock \
       -v /opt/rano/configs:/configs:ro \
       quay.io/kairos/auroraboot \
       --set container_image=docker://{{REGISTRY}}/rano/kairos-nuc:{{VERSION}} \
       --cloud-config /configs/nuc-server.yaml"

provision-rpis:
    @echo "Power on all 10 RPi4s AFTER this command starts"
    ssh {{NUC0_USER}}@{{NUC0_IP}} \
      "docker run --rm -ti --net host \
       -v /var/run/docker.sock:/var/run/docker.sock \
       -v /opt/rano/configs:/configs:ro \
       quay.io/kairos/auroraboot \
       --set container_image=docker://{{REGISTRY}}/rano/kairos-rpi4:{{VERSION}} \
       --cloud-config /configs/rpi4-worker.yaml"

# ── kubeconfig ────────────────────────────────────────────────────
kubeconfig:
    ssh {{NUC0_USER}}@{{NUC0_IP}} \
      "sudo cat /etc/rancher/k3s/k3s.yaml" \
      | sed "s/127.0.0.1/{{NUC0_IP}}/g" > kubeconfig
    @echo "export KUBECONFIG=$PWD/kubeconfig"

# ── Cluster operations ────────────────────────────────────────────
deploy:
    KUBECONFIG={{KUBECONFIG}} kubectl apply -k manifests/

status:
    KUBECONFIG={{KUBECONFIG}} kubectl get nodes -o wide
    KUBECONFIG={{KUBECONFIG}} kubectl get pods -n rano -o wide

localai-check:
    #!/usr/bin/env bash
    set -uo pipefail
    for ip in $(seq -f "192.168.1.1%g" 0 5) $(seq -f "192.168.1.2%g" 0 9); do
      echo -n "LocalAI $ip: "
      curl -s --max-time 3 "http://$ip:8080/v1/models" \
        | jq -r '.data[].id' 2>/dev/null | tr '\n' ',' || echo "UNREACHABLE"
    done

# ── MacBook setup ─────────────────────────────────────────────────
macbook-netbird KEY:
    sudo netbird up \
      --setup-key {{KEY}} \
      --management-url https://{{NUC0_IP}}:443 \
      --hostname rano-macbook-mlx

macbook-mlx-start:
    launchctl load ~/Library/LaunchAgents/ai.rano.mlx-llm.plist
    launchctl load ~/Library/LaunchAgents/ai.rano.mlx-embed.plist

macbook-mlx-test:
    curl http://localhost:8080/v1/models | jq
    curl http://localhost:8081/v1/models | jq

# ── Full pipeline ─────────────────────────────────────────────────
all:
    @echo "=== RANO Swarm Full Pipeline ==="
    @echo "Step 1: just gen-token && just inject-token"
    @echo "Step 2: just build-all"
    @echo "Step 3: just build-nuc0-iso → flash USB → boot nuc-0"
    @echo "Step 4: just start-netbird && just bootstrap-netbird"
    @echo "Step 5: just provision-nucs  (power on nuc-1..5)"
    @echo "Step 6: just provision-rpis  (power on all RPi4s)"
    @echo "Step 7: just kubeconfig && just deploy"
    @echo "Step 8: just macbook-netbird KEY && just macbook-mlx-start"
    @echo "Step 9: just status && just localai-check"
```

---

## Glossary

| Term | Definition |
|---|---|
| AuroraBoot | Kairos tool: pulls OCI image, serves PXE netboot (kernel + initrd + squashfs) + ProxyDHCP |
| kairos-init | CLI tool that "Kairosifies" any base OS image: installs k3s, kernel, immutability layer |
| ProxyDHCP | DHCP extension that co-exists with existing DHCP; adds PXE boot offer without taking over IP assignment |
| RuVix | Rust-first cognition kernel (ruvnet ADR-087): 6 primitives, 12 syscalls, proof-gated mutations |
| RVF | RuVector Format: signed package = complete cognitive unit (RuVix's boot object) |
| SafetyGate | Application-layer quorum check (pre-RuVix): 2-of-N HMAC signatures before APPLY |
| GGUF | GGML Unified Format: quantized model weights for CPU inference via llama.cpp |
| LocalAI | OpenAI-compatible local inference server (CPU-only on cluster); does NOT proxy cloud APIs |
| LiteLLM | Multi-provider OpenAI-compatible proxy: routes to LocalAI, MLX, OpenRouter, Anthropic |
| MLX | Apple Metal-accelerated ML framework; `mlx-lm` serves OpenAI-compatible inference |
| NetBird AN | NetBird Agent Network: per-account LLM gateway, keyless, identity-aware, policy-gated |
| OpenRouter | Cloud LLM aggregator: `provider/model` slugs, single API key, routes to 100+ models |
| MetalLB | Bare-metal LoadBalancer implementation for k3s: assigns LAN IPs to Services |
| MO | Managed Object (Ericsson ENM): NRCellDU, EUtranCellFDD, GNBDUFunction, etc. |
| FDN | Full Distinguished Name: unique Ericsson RAN object identifier |
| PM counter | Performance Management counter: 15-min granularity KPI data from Ericsson ENM |
| REVERTED trial | An APPLY action that was undone due to KPI degradation; stored as negative few-shot signal |
| WitnessChain | Immutable linked chain of cryptographically signed events (RuVix proof records) |
| K3S_TOKEN | Pre-shared secret for k3s cluster join; embedded in all cloud-configs before first boot |
| OEM partition | Kairos immutable partition containing cloud-config and OEM customizations |
