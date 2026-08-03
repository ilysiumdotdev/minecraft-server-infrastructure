# Kubernetes Minecraft Game Servers

A Kubernetes-native, GitOps-delivered Minecraft server network running on PaperMC server instances fronted by Velocity. Designed for scale, automated instance management, and security.

![An image of a Minecraft server select menu demonstrating the ability to connect to different backend servers via the same Velocity proxy.](/docs/minecraft_server_list.png)

## Architecture

```
[ Player ]
    │
    ▼ (TCP 25565)
[ Envoy Gateway ]
    │
    ▼ (Proxy Protocol)
[ Velocity Proxy ] ── (autostartstop) ──► [ Kubernetes API ] (Scale StatefulSet 0 ↔ 1)
    │
    ├──────── (Modern Forwarding) ────────► [ Athens (PaperMC StatefulSet) ]
    └──────── (Modern Forwarding) ────────► [ Thebes (PaperMC StatefulSet) ]
```

- **Velocity Proxy** — Handles incoming Minecraft client connections and dynamically routes players to backend server instances based on target host.

- **Auto-Scaling & Cost Savings** — Velocity uses the `AutoStartStop` plugin combined with native Kubernetes RBAC to scale backend `StatefulSet` instances to 0 when empty and automatically start them with a player connects.

- **Network & Ingress** — Envoy Gateway receives traffic and forwards raw TCP to Velocity (via its `TCPRoute`) with PROXY Protocol enabled to preserve real client IPs.

- **Secrets Management** — External Secrets Operator fetches sensitive values, such as Velocity's forwarding secret, RCON passwords, etc. from an external secret store to into native Kubernetes secrets.

## Operational Runbooks

### Adding a New Backend Server

To deploy a new backend server, follow these steps:

**1. Create the Server Directory**

Copy an existing server directory to the new directory (e.g., `kubernetes/production/servers/sparta`)

**2. Update Overlays**

- Modify `server.env` with the server name, version, and gameplay settings.
- Update `kustomization.yml` resource names and label selectors to match the server name.
- Update `externalsecrets.yml` to point to the correct remote secret path for RCON and any other required secrets for the server/plugins.

**3. Register backend in Velocity Config**

Add the new backend server to `kubernetes/production/proxy/config/velocity.toml`:

```toml
[servers]
sparta = "sparta-mc.minecraft.svc:25565"

[forced-hosts]
"sparta.mc.ilysium.io" = ["sparta"]
```

**4. Register in Root Kustomization**

Add the path to the server `kustomization.yml` to the root kustomization under `resources`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - proxy/
  - servers/athens
  - servers/thebes
  - servers/sparta # <-- Add new server to 'resources'
```

**5. Push and Wait for Reconcile**

FluxCD will pick up the changes once they land on `main` and deploy the new instance to the cluster.
