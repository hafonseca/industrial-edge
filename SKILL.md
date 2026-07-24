---
name: industrial-edge
description: Build, package, deploy and operate apps on Siemens Industrial Edge (IE). Covers (a) authoring IE Docker-Compose stacks with their strict quirks that differ from normal Docker, and (b) the platform itself — IEHUB/IEM/IED architecture, IEAP publishing, onboarding, OT networking/DNS/firewall triage, IEM Kubernetes pod recovery, licensing, and homologation. Use whenever building, packaging, writing a docker-compose for, deploying, or troubleshooting anything on Industrial Edge — anything mentioning "industrial edge", "IE app", "edge device", IEM, IED, IEHUB, IEVD, IEAP, iectl, Flow Creator, or IE-specific compose/platform constraints.
user_invocable: true
---

# industrial-edge

You are packaging/deploying an application for **Siemens Industrial Edge (IE)**.
An IE app is *almost* a normal `docker-compose` stack, but with hard constraints
that will silently break the deploy (or destabilize the whole device) if ignored.

Treat the rules below as **normative**. When they conflict with "normal Docker"
habits, the IE rules win.

---

## The rules (hard constraints)

1. **Compose version must be `'2.4'`.** Put `version: '2.4'` at the very top of the
   compose file. Not v3.x — the App Publisher rejects 3.x outright. The v2.4 schema
   is what enables the memory keys IE requires (`mem_limit`, `mem_reservation`) at
   the service level. ⚠ Do **not** add a CPU limit — it freezes the Publisher UI
   (see rule 10).

2. **No bind mounts. Named Docker volumes only.**
   - No `./data:/data`, no host paths, no relative `./` volumes at all.
   - Every persistent path must be a **named docker volume** declared under
     top-level `volumes:`.

3. **Names are GLOBALLY unique across ALL apps on the device — volumes,
   `container_name`, and published host ports.**
   - None of these are namespaced per app the way you'd expect. Generic names
     collide with other installed apps.
   - **Volumes:** never use names like `timescale-data`, `grafana-data`,
     `db-data`, `redis-data`, `config`. Always prefix with the app's own name,
     e.g. `mkmqttwatcher-data`, `mkmqttwatcher-config`. Keep the prefix
     distinctive.
   - **`container_name`:** also global on the device's Docker — two apps (or
     two variants of the same app) with the same `container_name` won't start.
   - **Host ports:** one published port belongs to one app; record per-app
     choices (see the ports lesson below).
   - **Multiple variants/versions of the SAME app on one device** (e.g. a
     per-connector build family like `mk-data-bridge-oracle` + `mk-data-bridge-uns`):
     suffix EVERYTHING per variant — `container_name`, volume `name:`s, and the
     host port. A shared compose template should interpolate the variant into
     all three.
   - ⚠ **Migration trap:** renaming the volume of an already-installed app
     orphans its data. Installs that predate this rule keep their legacy names
     (grandfathered) until consciously migrated — and no new app/variant may
     take those legacy names.

4. **`mem_limit` and `mem_reservation` are MANDATORY on every service.**
   - `mem_reservation` = the *floor*: RAM guaranteed and effectively reserved
     100% of the time for that service. Set it to the realistic minimum the app
     needs to run.
   - `mem_limit` = the *ceiling*: the maximum RAM the service may consume.
   - **Device-wide safety behavior:** if the whole device ever crosses ~100% RAM
     usage, IE will reset and temporarily cap every app down to its
     `mem_reservation`. So `mem_reservation` must be large enough that the app
     still *functions* (degraded is OK) when capped there. Don't set it absurdly
     low just to look lean.
   - Keep the sum of all `mem_reservation` on the device well under physical RAM.

5. **Network is ALWAYS the external `proxy-redirect` network.**
   ```yaml
   networks:
     proxy-redirect:
       external: true
   ```
   Attach every service that needs to be reachable to `proxy-redirect`. Do not
   invent custom bridge networks for inter-app routing.

   ⚠ **Declare it as `external: true` only — NEVER define your own `proxy-redirect`
   with `driver: bridge`.** In `FromBoxReverseProxy` mode the IED rewrites the
   compose on-device to plug in its own nginx (`edge-iot-core`); if you declared the
   network yourself, that rewrite silently flips your **volumes** to `external: true`
   (they lose their `<proj>_` prefix), and since an external volume must pre-exist,
   the deploy dies with `external volume "<name>" not found` / `Unable to start
   application` — retrying forever. `external: true` on the network stops the rewrite
   and the volumes stay normal named volumes. (With `FromBoxSpecificPort` there is no
   proxy wiring, so the compose isn't rewritten and it starts either way — hence the
   classic "works with a direct port, fails with the reverse proxy" symptom.) Full
   mechanism in the exposure-modes section below.

   **Layer-2 / fieldbus access to automation devices (Profinet, DCP, LLDP, GigE
   Vision cameras, etc.):** do NOT hand-roll a `macvlan`. IE has an **official**
   mechanism (Siemens SIOS **109810456**, "Creating a Layer 2 network access"):
   reference a **platform-provided external network** whose key *and* `name:` are
   both exactly **`zzz_layer2_net1`**, attached *in addition to* `proxy-redirect`.
   The physical NIC + addressing are configured on the **device** (not in the app
   compose).
   ```yaml
   services:
     myapp:
       networks:
         proxy-redirect:      # eth0 (UI/management)
         zzz_layer2_net1:     # eth1 (field/L2 access)
   networks:
     proxy-redirect:
       external: true
     zzz_layer2_net1:
       external: true
       name: zzz_layer2_net1
   ```
   - The **`zzz_` prefix is required**: Docker attaches networks alphabetically and
     the first becomes `eth0`; `zzz_` forces the L2 net to sort last so
     `proxy-redirect` stays the primary interface. Any extra container-to-container
     bridge network must sort *before* `zzz_layer2_net1`.
   - The L2 network is **only** for talking to field devices — not for
     container-to-container traffic (use a separate bridge for that).
   - Add `cap_add:` **only** for raw-L2 protocols (Profinet/DCP/LLDP need
     `NET_RAW`/`NET_ADMIN`). Plain UDP/IP protocols (e.g. **GigE Vision**) need no
     caps.
   - **Nuance:** L2-only devices (Profinet/DCP) don't need an IP, but **L3-over-L2**
     devices (GigE Vision cameras) DO — verify on the device that the container
     actually gets an IP on the field subnet via `zzz_layer2_net1`.

6. **Prefer hard-coded env vars inside the compose.** Configuring environment
   variables through the IE UI is painful and error-prone. Put the values
   directly in the service's `environment:` block in the compose file rather than
   using `.env` files or expecting the operator to set them in IE.

7. **No `build:` — images must be prebuilt.** IE compose cannot build images. Build
   the image separately (a `build.sh` is the convention) and never use a `build:`
   key. In the local IEAP/`iectl` path the tool reads the image from the **local
   Docker** by the **exact tag in the compose** — it does **not** pull. So the image
   must already exist in `docker images` with that precise `nome:vX.Y.Z` tag or the
   publish fails (`image not found`). **`:latest` is rejected** — always pin an
   explicit `vX.Y.Z` tag. **Bump the version on every test** — the IED refuses to
   reimport an already-installed version number.

8. **Multiple containers per app are allowed — but do NOT share a volume between
   containers.** A single IE app's compose can define several services. However,
   mounting the *same named volume* into two containers works on some device
   firmware versions and fails on others. Design so each volume is used by exactly
   one container. If two containers must exchange data, use the network
   (`proxy-redirect`) or give each its own volume — never a shared mount.

9. **Cap the logs.** Give every service a bounded `json-file` driver so app logs
   can't fill the device disk:
   ```yaml
   logging:
     driver: json-file
     options:
       max-size: "10m"
       max-file: "2"
   ```

10. **No CPU limit — it freezes the App Publisher.** A `cpus:` (or any CPU-limit)
    key makes the Publisher render an unexpected **"Other"** tab in the service
    config and the UI **hangs** there, blocking the publish. The Publisher doesn't
    understand the v2.4 CPU directive. Use **only** `mem_limit`/`mem_reservation`.
    If you must cap CPU, do it outside the Publisher (not supported in the `.app`
    flow).

11. **Set `container_name` and `restart: unless-stopped`.** A fixed `container_name`
    makes the container findable on-device (`docker ps`) — remember it's globally
    unique (rule 3). `restart: unless-stopped` keeps the app alive across device
    reboots.

---

## Reference skeleton

```yaml
version: '2.4'

services:
  myapp:
    image: myregistry/myapp:1.0.0   # prebuilt, exact tag, exists in local Docker; NO `build:` key
    container_name: myapp           # fixed + globally-unique on the device
    restart: unless-stopped
    # ports:                        # ONLY for FromBoxSpecificPort mode — see exposure modes
    #   - "40580:5000"
    networks:
      - proxy-redirect
    environment:
      - SOME_SETTING=value        # hard-code, don't rely on IE env UI
    volumes:
      - myapp-data:/data          # named volume, app-prefixed name
    mem_reservation: 256m         # guaranteed floor (must still run when capped here)
    mem_limit: 1g                 # hard ceiling — do NOT add a CPU limit (rule 10)
    logging:                      # bound the logs (rule 9)
      driver: json-file
      options:
        max-size: "10m"
        max-file: "2"

volumes:
  myapp-data:                     # globally-unique, app-prefixed

networks:
  proxy-redirect:
    external: true
```

---

## Exposing the app UI: two redirect modes

An app is published with a **redirect type** (the iectl `-t` flag / Publisher
setting). It decides how the device tile opens the UI **and whether you need
`ports:` in the compose**. Pick one:

### A) `FromBoxReverseProxy` — behind the device's nginx (recommended for web UIs)
- **Compose: NO `ports:`.** The IE reverse-proxy exposes the app; mapping a port
  here just causes conflicts.
- Tile URL: `https://<IED>/<REDIRECT_URL>/`. Requires an **`nginxjson`** config
  (`-n` on iectl / reverse-proxy config in the Publisher).
- The app **must be prefix-aware**: relative asset paths, trust `X-Forwarded-*`
  (Flask → `ProxyFix`), and do **not** force internal HTTPS/HSTS. Otherwise you get
  a blank page or an HTTPS redirect loop.
- Upside: the IED's **TLS + portal login sit in front of the app for free**.
- ⚠ Do **not** map ports in the reserved **40560–40599** range (the Publisher owns
  it — collision).

### B) `FromBoxSpecificPort` — direct host port (no proxy)
- **Compose: REQUIRES `ports: "<HOST_PORT>:<SERVICE_PORT>"`** (e.g. `"40580:5000"`).
- Tile URL: `http(s)://<IED>:<HOST_PORT>/` — no path prefix, so the asset-path
  problem disappears. Uses no `nginxjson`; the `redirectUrl` is just the port.
- ⚠ **No IE portal login in front** — the app is reachable directly by anyone who
  can reach the device. The app must protect its own actions (e.g. require IIH
  credentials). Serves plain HTTP if the app does no TLS (`isAppSecure: false`).
- Pick the launch port **deliberately from 40560–40599** here.

### nginxjson `port`/`protocol` errors (reverse-proxy mode, after the container is up)
Even with the container running, the tile can still 503 — the nginx on the device
names the cause in its log:

| nginx log | Cause | Fix |
|---|---|---|
| `connect() failed (111: Connection refused) … upstream` | `port` is the **published host** port | set `port` to the container's **internal** port (e.g. `5000`) |
| `SSL_do_handshake() failed … wrong version number` | `protocol: HTTPS` but the app speaks plain **HTTP** | `protocol: HTTP` (or make the app serve TLS) |

The browser↔IED hop is always HTTPS/443 (IED cert); `protocol` above is only the
internal nginx→container hop, which may be HTTP without violating "HTTPS-only" policy.

> **Reverse-proxy won't start at all (`external volume … not found`)?** That's the
> own-network → compose-rewrite → volumes-flip-to-external trap — see rule 5. Fix is
> `proxy-redirect: external: true`, not anything in the nginxjson.

---

## Don't leak config/secrets into the packaged image

The `.app` bundles the **entire Docker image**. If the `Dockerfile` does
`COPY app/config/` or `COPY app/data/`, every IP, credential and plant datum in
those folders ships to the customer.

- Ship a **`.dockerignore`** that excludes the real `init.json`/local data and keeps
  only an `init.example.json` template — the customer configures via the UI on first
  boot.
- Bonus: the same `.dockerignore` drops `build/`, `node_modules/`, `.git/` and large
  files, so builds are much faster.
- Runtime secrets go through `environment:` (rule 6), never hard-coded into the image.

---

## Pre-deploy checklist

- [ ] `version: '2.4'` at top
- [ ] No `build:` key — prebuilt image, explicit `vX.Y.Z` tag (never `:latest`),
      already present in local `docker images` with that exact tag
- [ ] Version **bumped** since the last test/install (IED refuses a repeat number)
- [ ] Zero bind mounts / `./` volumes
- [ ] Every volume named `<appprefix>-*` and declared under top-level `volumes:` (no `external`)
- [ ] No volume mounted into more than one container
- [ ] Every service has both `mem_reservation` and `mem_limit`
- [ ] **No CPU limit** anywhere (freezes the Publisher — rule 10)
- [ ] App still runs (even if degraded) at `mem_reservation`
- [ ] `proxy-redirect` declared `external: true` (never your own `driver: bridge`) and attached where needed
- [ ] Bounded `logging` (json-file `max-size`/`max-file`) on every service
- [ ] `container_name` set + `restart: unless-stopped`
- [ ] If field/L2 access is needed: uses external `zzz_layer2_net1` (not macvlan),
      `zzz_`-prefixed so it lands on eth1; `cap_add` only for raw-L2 protocols
- [ ] Config/secrets hard-coded in compose, not in `.env` or IE env UI
- [ ] `.dockerignore` excludes real config/data so secrets aren't baked into the image
- [ ] `ports:` present **only** if publishing as `FromBoxSpecificPort`

---

## Platform architecture & operations (IEHUB / IEM / IED)

The compose rules above are about *authoring an app*. This section is about the
*platform it runs on* — needed to answer deployment, networking, licensing and
homologation questions, not just to write a compose file.

**Three layers (upward-polling only — lower tiers poll higher, never top-down;
this is a cybersecurity requirement):**
- **IEHUB** — Siemens cloud. Buy apps, allocate licenses, instantiate IEMs, push
  new app versions, download dev tools (**IEAP** = IE App Publisher) and the
  **IEVD** image. Tenant-based, hierarchical user permissions.
- **IEM** (Industrial Edge Management, on-prem) — two flavors: **IEM Virtual**
  (appliance) or **IEM Pro** (a **Kubernetes cluster** on a Linux host, e.g. RHEL/
  Ubuntu). Needs connectivity to specific Siemens endpoints to sync licenses and
  pull app versions. Manages onboarded IEDs.
- **IED** (Industrial Edge Device) — runs the Docker apps. Physical (e.g. Siemens
  **IPC227G**) or virtual (**IEVD** on VMware ESXi/vSphere or Hyper-V, max 8 vCPU /
  64 GB). Needs only **local** connectivity to its IEM — **no direct internet**.

**Two deploy paths:**
1. **Cloud/scaled:** version published to IEHUB → replicated to IEMs → rolled out
   to IEDs.
2. **Local (offline-friendly):** **IEAP** connects to a **Docker Engine API** + an
   IEM and publishes the app straight into that IEM, which then deploys to its
   IEDs. This is the usual path here — each app is its own `docker-compose.yml`
   packaged via IEAP with a locally-loaded image. (No internet needed once images
   are local.) `iectl` is the Siemens IE CLI — a standalone binary; `chmod +x` and
   drop into `~/.local/bin` or `/usr/local/bin`.

**Networking is the #1 source of pain (segregated OT networks):**
- **One IEM per subnet.** An IED can only onboard to an IEM reachable in its own
  network range. Different workshop = different subnet = you need a *separate IEM*
  in that range (people literally carry a laptop with per-workshop IEM VMs between
  areas). IED↔IEM binding is per-IEM.
- **IEM K8s pods crash on ANY network-interface change.** The `gateway` (Kong OSS
  API gateway — single entry point, auth, JWTs) and `twin service` pods reliably
  crash whenever the host's network changes (bridge↔NAT, moving subnet, factory
  transport). Fix: `kubectl delete pod -n iem <pod-name>` — K8s recreates them
  (works offline; images already pulled) and the IEM comes back healthy. Expect to
  do this on *every* interface change.
- **IEM needs internet only for setup** (pull containers/K8s images / IEHUB
  content like the Flow Creator). Provision it somewhere with open internet, then
  move it to the factory. Because **K8s requires a fixed IP**, give internet
  without changing the IP via an internal **NAT within the IEM's own IP range** —
  and delete the gateway/twin pods after each bridge↔NAT switch.
- **IED→IEM uses the IEM's FQDN over HTTPS/443.** Triage connectivity in order:
  DNS first, firewall second. `getaddrinfo EAI_AGAIN <host>` = **DNS resolution
  failed before any TCP** → that's the **networks/DNS team** (DNS server reachable?
  port 53? does the record exist? is the IED on corporate vs public DNS?), *not*
  firewall. Only after the name resolves does firewall/443 matter. Isolate by
  hitting the **IP directly** — resolves-by-IP-but-not-by-name proves it's DNS.
  Codes: `EAI_AGAIN`=DNS down/timeout, `ENOTFOUND`=no record, `ETIMEDOUT`=resolved
  but no route, `ECONNREFUSED`=host up, port refused.
- **Flow Creator** (Siemens' Node-RED app on the IED) is a **minimal image**: no
  `ping`, maybe no `curl`; you often can't install extra nodes. The `exec` node is
  *sometimes* enabled. Diagnose with what's there: `getent hosts <fqdn>`,
  `nslookup`/`dig`/`host`, `cat /etc/resolv.conf`, `cat /etc/hosts`, `nc -vz host
  443`, or an HTTP Request to the raw IP. Also: apps that stored an L2/field IP at
  install time may need **reinstalling** if the device's L2 range changes later.
- **IEHUB→IEM app copy can silently fail** (IEHUB reports success, nothing lands on
  the IEM) even when "All domains connected" and K8s is error-free — it's the sync
  pod in the `iem` namespace; grab `kubectl get all -n iem` + pod logs.

**Licensing (for proposals/slides — factory audiences dislike OPEX, so frame
subscription as strategic flexibility against tech obsolescence vs. sunk CAPEX):**
- **IEM license:** one per onboarded IED; **1 free** with each IEHUB Access. So
  1 IED→1 IEM is free; each additional IED = annual subscription.
- **IED:** buy an **IPC** (CAPEX) or an **IEVD** subscription (OPEX, no dedicated
  hardware; runs on existing ESXi/Hyper-V). Cloud variant = IECD. IEVD/IECD are
  licensed per IED *and* per IED→IEM. Volume discounts >24.
- **Apps:** self-developed = **free**; **Mendix** = yearly Device License (≤10 named
  users/app, RAM tier 4–64 GB); other Siemens apps range free (drivers) → paid
  subscriptions (**AI Inference Server**, **WinCC Unified**). **HighByte**:
  Professional (1-yr, 3 named users/site) or Enterprise (unlimited).
- **AI on the edge:** the **AI Inference Server** app runs trained models via an
  embedded Python interpreter — deploy models as app *content*, no per-model custom
  container.

**Homologation / "software dependencies?" questions:** all IE apps are
self-contained Docker containers → **no external host deps** (Java/.NET/Visual
C++/Acrobat). Only prerequisites are a supported OS + the K8s/Docker infra IE
itself needs (IEAP just needs a Docker Engine API). Attribute the claim to the
**official Siemens docs** rather than personally guaranteeing it. Docs:
`docs.eu1.edge.siemens.cloud` (+ `/develop_an_application/`).

**"CI/CD on IE" expectation-setting:** there is **no native GitOps/ArgoCD-style
continuous-deploy**. What exists is app packaging/versioning, publish-to-IEM, and
controlled/semi-automated rollout. If a client asks for "CI/CD", pin down whether
they mean build+version+publish automation (plausible) vs. cloud-style continuous
deployment / immutable infra / instant rollback (not how IE works) — surface the
mismatch early so scope doesn't drift into an imagined "industrial Azure DevOps".

---

## Enrich this skill (do this every time it loads)

This skill is a **shared, growing knowledge base** about Industrial Edge, fed by
lessons from every project that loads it.

**At the end of any session where this skill was used, ask the user:**

> "Did we learn anything new about Industrial Edge in this project that isn't
> already in the `industrial-edge` skill? (New constraint, firmware gotcha,
> resource tuning insight, a name collision that bit us, etc.) If so, I'll add it."

If the user offers a lesson:
1. Confirm it's genuinely IE-general (not app-specific).
2. Add it to the relevant section above, or to the **Lessons learned** log below
   with a short date + context so future sessions can trust it.
3. Keep entries terse and normative. Prefer editing the rules directly when a
   lesson is a firm rule; use the log when it's a nuance/observation.

### Lessons learned (append-only log)

<!-- Format: - [YYYY-MM] <lesson> — <project/context> -->
- [2026-07] Initial ruleset captured from the mk-mqtt-tracker project (Hugo,
  Mekatronik): v2.4 mandatory, named volumes only + globally-unique app-prefixed
  names, mandatory mem_limit/mem_reservation with device-wide reset-to-reservation
  behavior, always the external `proxy-redirect` network, hard-code env in compose,
  never share a volume across containers.
- [2026-07] The IE platform issues frequent TCP **health-check probes** from the
  internal overlay network (e.g. `10.201.45.x`) against every published port. In
  service logs these look like rapid connect-then-immediately-disconnect churn
  from several different IPs within the same millisecond ("New connection … /
  disconnected: connection closed by client"), with no application traffic. This
  is benign platform liveness checking — NOT a client bug or a failing app. A
  genuinely reconnecting app instead shows repeated connects from a *single* IP
  spaced by its backoff interval. Probes may also try TLS against a plaintext port
  (harmless protocol error). Don't chase this noise; quiet it via app log settings
  if it's distracting.
- [2026-07] If an app exposes a UI/API that must be reachable from outside the
  device, publish it with a `ports:` mapping. The external (host) port is a
  **per-app decision** — pick one that won't collide with other IE apps on the
  device and record it. (Don't hard-code a standard port in this skill; ask/confirm
  per project. Example: mk-mqtt-tracker uses host port 40562 -> container 8000.)
- [2026-07] **Official Layer-2 / fieldbus access** = the external `zzz_layer2_net1`
  network, NOT a hand-rolled `macvlan` (Siemens SIOS 109810456). Now captured as a
  firm rule under rule 5. Discovered on the basler-edge-monitor project (Hugo,
  Mekatronik) connecting Basler GigE Vision cameras via pylon/pypylon. Key nuances:
  `zzz_` prefix forces it onto eth1 (proxy-redirect stays eth0); the NIC/addressing
  is device-side config not compose; `cap_add` only for raw-L2 (Profinet/DCP/LLDP);
  GigE Vision is UDP/IP so needs an IP on the field subnet — verify the container
  actually gets one via the L2 network.
- [2026-07] **Same-app variants collide too:** when one codebase ships multiple
  per-connector builds installed side-by-side on one IED (mk-data-bridge: oracle,
  uns, …), `container_name`, volume `name:`s AND host ports must be suffixed per
  variant (`mk-data-bridge-uns`, `mk-data-bridge-uns-data`, oracle=40561 /
  uns=40566). Now a firm part of rule 3. Also captured the migration trap:
  renaming volumes of an installed app orphans its config/secrets — the first
  installed variant keeps the legacy generic names until migrated deliberately.
  (mk-data-bridge, Hugo, Mekatronik.)
- [2026-07] A built-in **in-app diagnostics/console panel** is very valuable for
  debugging IE networking (which is opaque from outside the device): expose the
  container's `ip addr`/`route`/`neigh`, interface rx/tx drop counters, relevant
  `net.core.*` socket-buffer sysctls, ping, and (gated) a raw-command runner. On IE
  you often can't easily `docker exec`, so shipping these as HTTP endpoints behind
  the app UI gives live L2/field-network diagnosis. Gate any command-exec behind an
  env flag. (basler-edge-monitor, Mekatronik.)
- [2026-07] **Platform ops section added** from Hugo's Stellantis/factory history
  (mined from ChatGPT logs 2024–2026): 3-layer IEHUB/IEM/IED architecture, IEAP
  local-deploy path, one-IEM-per-subnet, the recurring **gateway/twin-service pod
  crash on every network-interface change** (fix = `kubectl delete pod -n iem …`),
  NAT-in-same-range trick for giving a fixed-IP IEM temporary internet, DNS-first
  (`EAI_AGAIN`) vs firewall triage for IED→IEM, minimal Flow Creator tooling,
  silent IEHUB→IEM copy failures, licensing model, homologation "no external deps"
  answer, and the "CI/CD doesn't map to IE" framing. See the new **Platform
  architecture & operations** section.
- [2024-10] **IED `.env` semantics differ from vanilla compose.** In a normal
  compose, `MX_Module_Variable=${TEST}` reads `TEST=...` from `.env`. In the IE/IED
  `.env` you instead set the **final referenced name directly** —
  `MX_Module_Variable='value'` — not the `${TEST}` intermediate. (Also: a leading
  dot on the host path mattered in that deploy.) Reinforces rule 6 — prefer
  hard-coding env in the compose to avoid this entirely.
- [2026-07] **Whole `iem` namespace in `ImagePullBackOff` is a DIFFERENT failure
  from the gateway/twin pod-crash-on-network-change** — do NOT reflexively
  `kubectl delete pod`. It means the IEM app images (`cr.eu1.edge.siemens.cloud/
  portal/*`, `/auth/*`) are **gone from the local containerd store** and the node
  can't re-pull them. Root cause is almost always **kubelet image garbage
  collection under disk pressure**: k3s evicts unused images once the root FS
  crosses the image-GC **high threshold (~85%)**. The tiny kube-system images
  (coredns, metrics-server) survive, so those pods stay Running — a misleading
  signal that "k8s is fine". Because pods are `imagePullPolicy: IfNotPresent`,
  present-locally = starts, absent = must pull. Diagnosis (all image/containerd
  commands need **root**; `kubectl` works as the normal user via kubeconfig):
  `kubectl describe pod -n iem <p>` (see the pull error + registry host);
  `sudo k3s ctr -n k8s.io images ls | grep -c portal` (0 = store wiped);
  `df -h /` (>85% = GC was the culprit); `uptime` (a recent reboot re-exposes it).
  (iem-stellantis, Hugo, Mekatronik.)
- [2026-07] **Registry-pull error decoding on the IEM node:** `401 Unauthorized`
  from `cr.eu1.edge.siemens.cloud` = reachable but creds stale — the `regcred`
  imagePullSecret in ns `iem` is refreshed by the **`regsecgen` CronJob** (every
  6h); let it run / check it Completed. `dial tcp …:443: connect: no route to
  host` = the node has **no route to the registry** (network problem, not auth).
  A bare `curl https://cr.eu1.edge.siemens.cloud/v2/` returning **HTTP 401 = the
  registry is reachable** (auth-gated), which is the healthy "network OK" result.
  (iem-stellantis.)
- [2026-07] **Stale default route after moving an IEM VM between subnets** silently
  kills registry access. Symptom: two `default` routes on one NIC, e.g. a
  `proto static onlink` route to the OLD subnet's gateway (metric 0, wins) shadowing
  the correct `proto dhcp` gateway (metric 100) → all off-subnet traffic
  black-holed → pulls fail `no route to host`, `ping 8.8.8.8` 100% loss. Source was
  a leftover `routes: {to: default, via: <old-gw>}` block in
  `/etc/netplan/01-static-ip.yaml` (with the old static IP still present, just
  commented). Fix: remove that block (DHCP already provides the right default), or
  for a fixed-IP offline move set a clean static netplan with **no** default route
  at all (the IED is on the same /23; none is needed). `ip route del default via
  <old-gw> dev <nic>` is a safe runtime test — it won't drop an SSH session that
  rides the local-subnet route. (iem-stellantis.)
- [2026-07] **Making an IEM survive offline / airgap — protect its images 3 ways.**
  The IEM's app images live only in the node's local containerd store; offline, a
  GC'd or lost image = permanent `ImagePullBackOff` (no registry to re-pull). While
  still online: (1) **remove disk pressure** — k3s image-GC fires at ~85% root FS;
  if the disk is LVM with free VG space (`vgs` → `VFree`), grow it live:
  `lvextend -l +100%FREE /dev/<vg>/<lv> && resize2fs /dev/<vg>/<lv>`. (2) **Pin the
  images** so GC can never evict them, without disabling GC globally:
  `sudo k3s ctr -n k8s.io images label <ref> io.cri-containerd.pinned=pinned`
  (managed images show label `io.cri-containerd.image=managed`; pinned show
  `pinned=pinned`). (3) **Local auto-restore backup** — export every app image into
  k3s's airgap auto-import dir **`/var/lib/rancher/k3s/agent/images/`** (create it;
  default k3s data-dir); k3s re-imports any `*.tar` there on every start, no
  internet. ⚠ **Export gotcha:** `k3s ctr images export <one.tar> <many refs…>` in
  one shot hits a containerd transfer-service bug ("error copying stream: file
  already closed", rc still 0) that **silently truncates** the tar — always
  `tar tf` verify. Export **one image per tar in a loop** (robust), and run it
  **detached** (`setsid nohup …`) so an SSH/pty close can't SIGHUP it mid-write.
  (iem-stellantis, Hugo, Mekatronik.)
- [2026-07] **Moving a running k3s-based IEM to a new fixed IP.** k3s here had no
  explicit `--node-ip`/`--tls-san` (auto-detects the default-route IP), so it picks
  up the new IP on restart — but pin it to avoid surprises via
  `/etc/rancher/k3s/config.yaml`: `node-ip: <new-ip>` + `tls-san: [<new-ip>,
  <hostname>]` (adds the API-server serving-cert SAN). Sequence: write static
  netplan + k3s config → reboot → `kubectl get nodes -o wide` shows the new
  InternalIP → **delete the gateway (Kong) and twin-service pods** (they crash on
  the IP change; recover from local images) → wait Ready. Caveat: if the IEM base
  URL / IED-onboarding was pinned to the OLD address, the UI/onboarding may still
  point there even once pods run — onboard the IED against the new IP, or make its
  FQDN resolve to the new IP on the offline net. (iem-stellantis.)
- [2026-07] **Two UI-exposure redirect modes** (`FromBoxReverseProxy` vs
  `FromBoxSpecificPort`) added as a section. Reverse-proxy = NO `ports:`, needs
  `nginxjson`, app must be prefix-aware, gets IED TLS+login for free; direct-port =
  REQUIRES `ports:` from the reserved **40560–40599** range, no portal login in
  front. Reverse-proxy 503-after-up debugging: nginxjson `port` must be the
  container-**internal** port (not the published host port; `Connection refused`),
  and `protocol` must match the app (HTTP vs HTTPS; `wrong version number`).
  (compose cheatsheet digest, MekaSync/mk-autotag, Hugo, Mekatronik.)
- [2026-07] **Own `proxy-redirect` network breaks the reverse-proxy deploy.** Real
  case (device `mrcindxievd003`, mk-autotag): the same image published as
  `FromBoxSpecificPort` came up fine but as `FromBoxReverseProxy` looped on
  `external volume "config-data" not found` / `Unable to start application`. Root
  cause: declaring the network yourself (`driver: bridge`) makes the IED rewrite the
  compose to wire in its nginx (`edge-iot-core`), and that rewrite strips the
  `<proj>_` prefix off the volumes and marks them `external: true` — which must
  pre-exist, so they don't, so it dies. The network "already exists"/"endpoint
  already exists" errors in the device log are **tolerated**; the volume error is
  the fatal one. Fix = `proxy-redirect: external: true` (now emphasized in rule 5).
  Direct-port has no proxy wiring → no rewrite → volumes stay normal → it starts.
- [2026-07] **A CPU limit in the compose freezes the App Publisher** — it opens an
  unexpected "Other" tab in the service config and hangs (the Publisher can't parse
  the v2.4 CPU directive). Removed the old `cpus:` recommendation; now rule 10 =
  memory limits only, cap CPU outside the Publisher if ever truly needed. (compose
  cheatsheet digest.)
- [2026-07] **iectl reads the image from LOCAL Docker by exact tag — it does not
  pull.** In the local IEAP path the `nome:vX.Y.Z` in the compose must already be in
  `docker images` or publish fails; `:latest` is rejected outright; and the IED
  refuses to reimport an already-installed version number, so **bump the version
  every test**. Folded into rule 7. (compose cheatsheet digest.)
- [2026-07] **Bound the logs + don't leak secrets in the image.** Give every service
  a `json-file` driver with `max-size`/`max-file` so app logs can't fill the device
  disk (rule 9). And the `.app` bundles the whole image, so a `Dockerfile` that
  `COPY`s `app/config`/`app/data` ships real IPs/credentials to the customer — use a
  `.dockerignore` that keeps only an `init.example.json` template (also speeds builds
  by dropping `build/`/`node_modules/`/`.git/`). (compose cheatsheet digest.)
