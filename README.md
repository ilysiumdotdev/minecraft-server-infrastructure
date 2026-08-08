# Kubernetes Minecraft Game Servers

A Kubernetes-native, GitOps-delivered Minecraft server network running on server instances fronted by Velocity. Designed for scale, automated instance management, and security.

![An image of a Minecraft server select menu demonstrating the ability to connect to different backend servers via the same Velocity proxy.](/docs/images/minecraft_server_list.png)

## Architecture

![An architecture diagram illustrating how a Minecraft client connects to a Minecraft server through the Velocity proxy.](/docs/images/minecraft_kubernetes_diagram.png)

### Velocity

Velocity is deployed as a Kubernetes `Deployment` and proxies incoming Minecraft connections to dynamically route players to instances based on the requested hostname (e.g., `survival.mc.example.com`, `creative.mc.example.com`). It uses native Kubernetes service names to direct traffic to running instances inside the cluster:

```toml
[servers]
athens = "athens-mc-minecraft.minecraft.svc:25565"
thebes = "thebes-mc-minecraft.minecraft.svc:25565"
try = ["athens"]

[forced-hosts]
"athens.mc.ilysium.io" = ["athens"]
"thebes.mc.ilysium.io" = ["thebes"]
```

Velocity is the only component of the Minecraft network exposed outside of Kubernetes via the cluster's Envoy Gateway, which routes raw TCP traffic to the velocity service via a TCP listener and **TCPRoute** (defined in `kubernetes/base/tcproute.yml`). In addition, each backend instance is configured with Velocity's **forwarding secret**, which means that backend servers will only accept connections coming from the Velocity proxy. The nuance with this approach is that because Velocity itself is behind a proxy, to extract real player IPs from connections, the parameter `haproxy-protocol` must be set to `true` under the `[advanced]` section of the `velocity.toml` config.

```toml
[advanced]
haproxy-protocl = true
```

### Servers

The backend server instances are deployed and managed with Jeff Bourne's [Minecraft Server Helm Charts](https://github.com/itzg/minecraft-server-charts). The repository structure makes use of layered Helm values to efficiently combine common server configurations with instance-specific configs. Some examples:

**Common Values (shared `ConfigMap`)**

- Server Image
- Server Type
- Kubernetes Services Config
- RCON Config
- Data Persistence

**Instance Values (defined inline in `HelmRelease`)**

- Gamemode
- Difficulty
- Game Version
- Max Players
- RCON Credentials

Each server is defined in its own directory under `servers/` in the environment oerlay. The server directory is managed as an atomic unit, with plugin configs, external secrets, and the server `HelmRelease` object itself all bundled and deployed with the server's own `kustomization.yml`. These kustomizations are then aggregated at the environment `kustomization.yml` by referencing the server directory:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../base
  - servers/athens
  - servers/thebes
```

### Autoscaling & Cost Savings

![An image showing two Minecraft servers that are each sleeping due to inactivity. Each server's MOTD reads 'The server is currently asleep. Join to wake it up.'](/docs/images/minecraft_server_list_sleep.png)

The [AutoStartStop](https://modrinth.com/plugin/autostartstop) plugin is used to automatically to automatically start and stop servers based on interaction, providing scale-to-zero functionality for resource savings on inactive instances. The plugin uses a rule-based system to define actions to take based on certain triggers:

```yaml
rules:
  auto_stop_on_empty:
    triggers:
      - empty_server:
          empty_time: 5m
    action:
      - log:
          level: INFO
          message: "Server ${empty_server.server.name} has been empty for ${empty_server.empty_time}. Shutting down server..."
      - stop:
          server: ${empty_server.server.name}
```

Although primarily used for start/stop behavior, these rules can also be used for ping management, player messaging, and scheduled actions.

### Secrets Management

The External Secrets Operator is used to fetch sensitive server and plugin configuration values from a secure secrets storage solution. These secrets projected into the cluster as Kubernetes `Secret` objects, where they can then be templated into configuration files, or mounted directly into running containers as environment variables.

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: velocity-forwarding-secret
  namespace: minecraft
spec:
  secretStoreRef:
    kind: SecretStore
    name: minecraft-secret-store
  refreshPolicy: Periodic
  refreshInterval: "1h0m0s"
  target:
    name: velocity-forwarding-secret
    creationPolicy: Owner
  data:
    - secretKey: forwarding_secret
      remoteRef:
        key: minecraft/velocity
        property: forwarding_secret
```

### External Networking

Because forwarding the default Minecraft port **TCP 25565** attracts attention from malicious bots and crawlers, `SRV` records are used on public DNS to direct the Minecraft client to a non-standard WAN port for client connections. The Minecraft client is capable of looking up `SRV` records that follow the format `_minecraft._tcp.<server-address>`, which means that no port needs to be specified when adding a server. This approach is used to provide a seamless user experience, all while masking the service from most automated, low-sophistication attacks.

***

## Operational Runbooks

### Adding a New Backend Server

To deploy a new backend server, follow these steps:

**1. Create the Server Directory**

Copy an existing server directory to the new directory (e.g., `kubernetes/production/servers/havana`)

**2. Update Overlays**

- Modify the server `HelmRelease` object with instance-specific configurations and overrides.
- Update `kustomization.yml` resource names and references to match the server name.
- Update `externalsecrets.yml` to point to the correct remote secret path for RCON and any other required secrets for the server/plugins.

**3. Register backend in Velocity Config**

Add the new backend server to `kubernetes/production/proxy/config/velocity.toml`:

```toml
[servers]
havana = "havana-mc-minecraft.minecraft.svc:25565"

[forced-hosts]
"havana.mc.ilysium.io" = ["havana"]
```

**4. Register in Root Kustomization**

Add the path to the server `kustomization.yml` to the root kustomization under `resources`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../base
  - proxy/
  - servers/athens
  - servers/thebes
  - servers/havana # <-- Add new server to 'resources'
  - externalsecrets.yml
```

**5. Push and Wait for Reconcile**

FluxCD will pick up the changes once they land on `main` and deploy the new instance to the cluster.
