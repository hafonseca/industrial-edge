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

### Recovering the IEM Pro initial admin password (`iem_user`) after deploy

`ieprovision install` prints the initial admin credentials **once**, to STDOUT, at
the end of the install. Losing that output does **not** mean redeploying the IEM —
the password lives in a Kubernetes Secret in the `iem` namespace and can be read
back at any time:

```bash
kubectl get secret keycloak-secret -n iem \
  -o jsonpath="{.data.INITIALUSER_PASSWORD}" | base64 -d; echo
```

Equivalent with go-template (use this when the key has characters `jsonpath`
chokes on, e.g. dots or dashes inside the key name):

```bash
kubectl get secret keycloak-secret -n iem \
  -o go-template='{{index .data "INITIALUSER_PASSWORD" | base64decode}}{{"\n"}}'
```

**`jsonpath` syntax traps** (both fail *loudly* but also dump the whole Secret):
- `{data.INITIALUSER_PASSWORD}` (no leading dot) → `unrecognized identifier data`.
  The leading `.` is mandatory — without it the parser reads `data` as a bare
  identifier, not a field of the root object.
- `{data[INITIALUSER_PASSWORD]}` → `invalid array index`. Brackets mean *array
  index* in kubectl's jsonpath; they do **not** work as a map-key accessor with a
  bare word. (Quoted form `{.data['INITIALUSER_PASSWORD']}` does work, but prefer
  the dot form or go-template.)

⚠ **Both malformed variants print the entire Secret object to the terminal** —
every Keycloak client secret plus `CUSTOMER_ADMIN_PASSWORD` and
`INITIALADMIN_PASSWORD`. Base64 is *encoding, not encryption*: anything on that
screen is plaintext to anyone who reads it. If such output was photographed,
screenshotted, pasted into a chat/ticket, or otherwise left the environment,
treat it as a credential disclosure: **rotate** the affected passwords/client
secrets, then clear the local trail (`history -c`, wipe the terminal scrollback,
and delete any saved log). Same reasoning applies to storing the password in a
plaintext file next to the deployment artifacts — keep it out of git.

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

### Automating publish and rollout with `iectl` (verified on iectl 2.19.8, 09/2026)

Everything below is scriptable; the house implementation lives in
`mk-data-bridge/deploy/` (`release.sh`, `upload-iem.sh`, `upload-iehub.sh`) with
`docs/devops-iectl.md`. Source: Siemens manual "Industrial Edge Platform Operation:
APIs & References" v26.09 (tutorials *Application Pipeline*, *Manage IEM Jobs*,
*Upload App on the IE Hub*), flags re-checked against the binary's `--help`.

**Pin the application id, or every machine publishes a different app.** The `.app`
carries a 32-hex `applicationId` that the publisher generates per workspace, and the
workspace is local/gitignored. Same name + different id = a *new* app in the IEM,
which the IED installs from scratch (new volumes, orphaned config). Keep a versioned
registry of ids per app/variant and pass it on create:
`iectl publisher standalone-app create --appId <32 hex>` (capital **I**: the
publisher plugin is Node/commander and case-sensitive; `--appid` is silently not
that flag). Generate new ids with `openssl rand -hex 16`; recover an existing one
from the IEM with `iectl iem device-apps app-details --app-name <name>`
(`.applicationId`). Never change a pinned id of an app that is already on an IEM.
**An IEM's ids are not always hex:** apps that came through the IEHub (and some
older locally-published ones) carry 32-character **base62** ids such as
`XGZulVRuU5A7TnOvFySJO2yTV7lFhTha`, and `versionId`s are base62 too. Don't validate
ids with a hex regex, and don't assume `--appId` accepts a base62 id until tested
(untested as of 2026-09).

**Two ways to hand the image to the publisher.** Default is the Docker Engine API
on `tcp://127.0.0.1:2375` (`config add publisher --dockerurl`), which is root
without auth. The alternative needs no TCP at all: `docker save img:tag -o x.tar`
and `version create ... --imagetarjson '{"<service>":"x.tar"}'`; then
`config add publisher` takes only `--name --workspace`. That is the CI-runner path.
`--changelogs` wants literal `\n` for line breaks. Re-running `version create` on an
existing version fails; delete it first in the workspace (`version delete`) to make
a script idempotent. The exported file is named `<appId>_<version>.app`.

**Level 1, straight into an IEM** (`iectl config add iem --name --url --user
--password-stdin`; `IE_SKIP_CERTIFICATE=true` / `EDGE_SKIP_TLS=1` for self-signed):

| step | command | trap |
|---|---|---|
| upload | `iectl iem device-apps upload --app-file-path X --follow` | needs IEM helm >= 1.15.5; older IEMs: `iectl iem catalog import-application --app-file-path X --follow` |
| ids | `iectl iem device-apps app-details --app-name N` | `.applicationId`, `.versions[].versionId` |
| devices | `iectl iem device list --size 1000` | **default page size is 5**; without `--size` the device is simply missing |
| installed? | `iectl iem device list-apps --deviceid ID` | decides `installApplication` vs `updateApplication` |
| rollout | `iectl iem job batch-create --appid A --versionId V --operation installApplication --infoMap '{"devices":["id1","id2"]}'` | one batch per app, N devices; returns the batch id in `.data` |
| wait | `job batch-status --batchId B` until `PROCESSED`, `job list --id B` (**`--id`, not `--batchId`**) for `installedJobId`s, `job device-job-wait --id J --timeout S` | the wait returns at timeout even if the job is not done: read the output |
| recon (read-only) | `device get-details --id`, `device get-statistics --id`, `device-logs list --id`, `job device-job-list --pagesize 1000`, `job import-job-list`, `iem-extensions list`, `device-types` | `get-statistics` returns JSON **inside a string**, keyed by epoch-ms; it carries `SystemInfo` (CPU, RAM, uptime), `StorageInfo`, `interface[]` (IPs, DNS), `ntpstatus`, and `AppCount.MemoryUsage` = the sum of the apps' memory limits vs `MemoryCapicity` (sic) |
| catalog by id | `device-apps app-details --app-id ID` | prefer `--app-id` in loops: `--app-name` breaks on names with spaces or a stray `\r`, and the id is unique |

Verified on a real IEM (M. Dias Branco, 2026-09-25, v1 API): `device list` →
`data[].deviceId/deviceName/deviceStatus/deviceVersion`; `app-details` →
`applicationId`, `versions[].versionId/version/creationDate`; `device list-apps` →
`data[].applicationId/versionNumber/status` plus the field **`verionId`** (typo in
the API, keep it when parsing); `job device-job-list` → `installedJobId`, `batchId`,
`operation`, `status`, and `appVersion` holds the **versionId**, not the number.
`device-apps upload --follow` prints the job as `{"data":"<jobId>"}` then polls to
`COMPLETED` / "Application Imported Successfully". A device's `status` in the IEM
(`ACTIVE`) says nothing about liveness: check the timestamp of the last
`get-statistics` snapshot (a dead BX-59A stayed `ACTIVE` for six weeks).

**`iem` vs `iem-v2`.** In iectl 2.19 the whole `iectl iem` group is marked
*deprecated* and `iectl iem-v2` (IE Management V2) coexists with it. V2 differences:
`device-apps import --file X` (no `--follow`; wait with `job import-job-wait --id`),
details only by `--applicationId` (so the pinned id matters; name lookup is
`device-apps list --filter "name contains '...'"`), an extra
`device-apps publish --applicationId --versionId` before installing, no
`device list-apps` (use `device details --device-id`), `job get-batch-jobs --id`
instead of `job list`. Which group applies depends on the target IEM's generation,
not on the iectl version: ask, don't assume.

**Level 2, through the IEHub** (tenant-scoped; needs an **API user**, see below):

```
iectl config add iehub --name X --url https://iehub.eu1.edge.siemens.cloud --user <email> --password-stdin   # URL without the hub name
iectl iehub product-management create --product-name N --icon-path icon.png     # once per app
iectl iehub product-management version create --product-name N --version 1.2.0  # major.minor.patch+meta <= 32 chars
iectl iehub product-management version upload --app-binary X.app --product-name N --version 1.2.0
iectl iehub product-management version private-release --product-name N --version 1.2.0
iectl iehub library copy-product --product-name N --iem-name <IEM instance>      # one per IEM of the tenant
```
Three things the manual does not say, all hit on 2026-09-21: **`product-management
create` is not idempotent** (a second call creates another product with the same
name, after which every `--product-name` command fails with "more than one product";
look the product up in `product-management list` first and use `--product-id`);
the **binary and the icon get an asynchronous virus scan** (~1 min): `private-release`
before it ends fails with "Version file scan is not completed yet", and `delete` of a
fresh product fails with "product icon scan is not completed yet" (poll
`version get-details` → `files[].virusScan.status == COMPLETED`); and the release
itself is asynchronous: `CREATED` → `PRIVATE_RELEASE_IN_PROGRESS` →
**`ECOSYSTEM_REVIEWED`**, which is the released state (the version then shows in
`iehub library list`). *Private* release makes the version visible only in the tenant's Library, no
Siemens review; *public* release (Marketplace) goes through Siemens. So level 2
only reaches IEMs that live in **your** IEHub tenant (e.g. an IEM Pro you host for a
customer); a customer IEM in the customer's own tenant is level 1 or Marketplace.
Whether an IEM that already copied the product picks up later versions by itself
or needs another `copy-product` is not documented; treat as unverified and copy
every time (the copy is asynchronous and can fail silently, see the sync-pod note
above). Installing on the IEDs from there is still a batch job on the IEM.

**Never upload the same `.app` file to two IEHub tenants.** Reproduced by Hugo
(Mekatronik, 09/2026): the *identical* `.app` released in the house tenant and then in
the customer's tenant (M. Dias Branco) breaks in the second tenant. The `.app` carries
the app id, a random `versionId` from `version create` and `digests.json`; which of
them collides is not known, so it is untested whether a *repackaged* `.app` (same app
id, new version) is enough or whether the second tenant needs its own app id. Until
that is tested: the customer's production apps are released only in the tenant that
owns the IEMs running them (a customer with its own tenant gets them released *there*,
through an API user of that tenant); the house tenant gets **test apps with their own
ids**, never the production `.app`; and if one variant must exist in two tenants,
repackage per tenant at minimum, and prefer a distinct pinned app id per tenant.
**The trigger is in the IEHub, not in the `.app` itself:** the very same file that was
released in the house tenant later imported cleanly by *direct* `iem device-apps
upload` into the customer's IEM (2026-09-25), with app id and `versionId` untouched.
Level 1 is therefore the safe way to get a house-built `.app` onto a customer IEM.

**IEHub API access is not the portal login.** The portal uses the Siemens ID
(SSO, MFA); `iectl` needs a *CLI/API password* generated in the IEHub UI: top-right
menu → **API access management** → **+ Grant API Access**, which shows the
password once. `--user` is the same e-mail as the Siemens ID; the account must be a
member of the tenant with rights on the products (product management) and, for
`iehub run-ied-dev`, "device builder" access. `iectl iehub token fetch` returns a
token that can be exported as `IEHUB_TOKEN` to skip re-authentication per command.
`iectl` itself is downloaded from the IEHub (Download Software → Developer Tools →
Industrial Edge Control Linux/Windows); the zip holds just the binary, the `.7z`
next to it is only the OSS disclosure HTML. `iectl --version` does not exist, but
`iectl version` does (`Release Version: Win-v2.19.8`, build hash, build time). The
Windows build (`iectl.exe`, 176 MB, same build hash as Linux) runs the same bash
scripts unchanged under Git Bash: drop it in a PATH dir as `iectl.exe` and `command -v
iectl` resolves it. Only the image-tar generation needs Docker; uploading and
installing a prebuilt `.app` does not.

**Manifests.** `iectl apply --manifest x.yaml` chains commands with variables and
JSONPath references to earlier outputs (`appid:
"iem.catalog.list#{.data[?(@.title=='app')].applicationId}"`), an alternative to
bash for the same pipelines.

**Compatibility gates:** iectl >= 2.10.1 needs IEM helm >= 1.8.1 (image
compression in the `.app`); iectl >= 2.7.1 cannot talk to IED-OS <= 1.12.0-10.

---

## Physical recovery of an IPC Edge Device (no UI, no IEM)

**There is no physical factory reset.** The IE **Hard Reset** — which removes the
device from the IEM and deletes apps, user data, certificates, the jwt-auth file
and proxy data — is software-only: IEM Management UI, or the device's own UI
(`Settings > System > Hard Reset`). No button, jumper or key combo triggers it.

What the hardware offers is only a **hard reboot**: hold the power button
**> 10 s** (IPC operating instructions: RAM data is lost, disk data *may* be lost,
"perform a hardware reset only in the case of an emergency").

**BX-59A quirk:** a hard reset/reboot can fail leaving **3 LEDs red**. Fix: hold
`<Esc>` while powering on to enter the BIOS, set `Power > XHCI USB Wake
Capability > xHCI Mode = disabled`, then retry the reset/reboot.

**Wait before you wipe.** Two documented IED-OS behaviours look exactly like a
dead device:
- After a power cut or forced shutdown the OS runs a **filesystem recovery that
  takes 2–3 h** (release-note known issue). Leave it powered on that long before
  concluding it is bricked.
- Stuck in the boot phase after a firmware update → power-cycle manually; the
  device comes back on the **previous firmware** (A/B slots), then retrigger the
  update.

### The Service Stick is the only supported repair

If the OS cannot boot, Siemens is categorical: deploy the OS artifact via the
**SIMATIC IPC Industrial Edge Service Stick**. "Any other way is not supported and
will cause the device to lose its authenticity and therefore the device will not be
supported in the Edge Ecosystem."

Anatomy (verified by opening `simatic-ipc-ied-ss-3.0-x86-64.zip`, 08/2026): the SIOS
zip ships **both halves already paired** —
- `simatic-ipc-ied-ss-3.0.0-6-x86-64.wic.gz` — the stick image. GPT layout is
  `data` (FAT32, 6.4 GB) + `ESP` (87 MB); 6.53 GB raw, so a **≥ 8 GB** USB. Burn
  with Rufus in **DD mode** or `dd ... bs=4M oflag=direct`.
- `simatic-ipc-ied-os-3.0.0-51-x86-64.swu` — the firmware itself. Copy **exactly
  one** `.swu` into the `data` partition, eject safely, replug and re-verify its
  SHA-256 (ejecting without safe-removal corrupts it).

On the device: monitor + keyboard attached, USB boot enabled (BIOS `SCU > Boot`) →
Boot Manager → the USB → **"Wipe Data & Reinstall"** → Yes → `Enter` →
`Power Management > Reboot`. Never unplug the stick or cut power mid-install.

⚠ **USB boot is OFF by default on the IPC — enabling it is a mandatory step, not a
check.** Straight out of the box (confirmed in the field on a BX-59A, 08/2026) the
device simply does not list the stick in the Boot Manager, which reads exactly like
"the stick is broken" and sends people off re-burning a perfectly good USB. Enter
the BIOS (hold `<Esc>` at power-on), go to `SCU > Boot`, enable USB boot, save with
`F10`, and only then expect the stick to appear. The
same stick also **collects device logs** (needs ≥ 2 GB free) — do that *before*
wiping, since the logs are what any SR will ask for.

⚠ **The stick pins an OS version — check for a silent downgrade.** The 3.0 stick
installs `3.0.0-51`; a device running e.g. `3.2.0-17` gets rolled back two minor
versions (re-update from the IEM afterwards). Prefer the stick whose `.swu` matches
the installed version when SIOS offers one.

⚠ **Don't reuse the old `ies-os-*.img` medium on modern devices.**
`ies-os-1.1.1-12-amd64.img` is the previous-generation **"Industrial Edge Service
Medium"** (Siemens Industrial OS 2.1.1 buster, kernel 4.19, 2021; Debian package
`service-installer` = "Industrial Edge Service Media"). It works completely
differently from the Service Stick: it mounts the device's `efiboota`/`efibootb`
partitions, backs up `system.efi`/`BGENV.DAT`/`EFILABEL` per device serial onto its
own `SERVICE` partition, copies `system.hardreset.efi` over the device kernel,
disables the watchdog (`bg_setenv -w 0`, EFI Boot Guard) and reboots — so menu item
**2 (restore system files) is mandatory** after item 1, otherwise the device stays
on the reset kernel. Its bundled kernels are `ied-os-1.2.0-57` (hard reset) and
`ied-os-1.0.0-48` (delivery, menu 8, gated to IE 1.0). On a BX-59A that is a
version mismatch that can cost you *both* boot slots. It is still useful as a
**bootable diagnostic shell**: menu item 3 drops to bash — `lsblk -o
NAME,SIZE,PARTLABEL,LABEL,FSTYPE`, mount the device root, `cat /etc/os-release`.

### Device/OS naming to keep straight

`SIMATIC IPC AI IE Device-OS v1.0` (BX-59A only, 08/2024) was **merged into IE
Device-OS V3.0** (`simatic-ipc-ied-os-3.0.0-51`, 03/2025) — there is no separate
"AI" OS line any more, and older docs that say otherwise are stale. V3.0 supports
127E / 227E / 427E / 847E / 227G / BX-39A / BX-59A; **V3.1.1 and later support only
the BX-59A** (MLFB `6AG4133-0DE40-0WN0`: i9-13900E, NVIDIA L4, 32 GB DDR5, 1 TB
NVMe). Latest seen: `3.2.0-17` (06/2026, IEDK 1.26.4, Debian 12 / kernel 6.1).

### BX-59A LED map (4 LEDs: Power / Run / Error / Maintenance)

| State | Power | Run | Error | Maint. |
|---|---|---|---|---|
| Hard reset succeeded (device no longer in IEM) | green | green flashing | – | – |
| Not connected to the IEM | green | green flashing | – | – |
| Connecting to IEM (USB config file inserted) | green | green flashing | – | orange flashing |
| Connected to the IEM | green | green | – | – |
| Connection to the IEM failed | green | – | red flashing | – |
| IED-OS update in progress | green | – | – | orange flashing |
| Shut down | orange | – | – | – |

Re-onboarding after a reinstall needs no browser: put **one** Edge Device
configuration file (generated in the IEM) on a USB stick and insert it — the
process starts automatically and writes `conf-usb.log` / `services.log` back to the
stick for diagnosis.

### Reading the Siemens docs portal programmatically

`docs.industrial-operations-x.siemens.cloud` is a Fluid Topics SPA: plain HTTP
fetches return an empty shell, so search engines and fetch tools see nothing. The
public REST API works and is by far the fastest way to mine it:
`GET /api/khub/maps` (every manual + its id) → `GET /api/khub/maps/{mapId}/toc`
(topic tree with `contentId` and prettyUrl) → `GET
/api/khub/maps/{mapId}/topics/{contentId}/content` (the topic HTML). There is no
exposed search endpoint — walk the TOC and filter titles.

### Getting Siemens to move on a problem (SR etiquette)

The **Support Request is the vehicle**, always. Product owners and account contacts
will help, but they escalate *into* the SR — the Chlorum case (Jan 2026) is the
template: the PO's first answer was "explain the issue, mention the system
information, and upload the logs via our support platform ... from there, customer
requests are handled with highest priority and a dedicated contact". After every
finding on your side, **update the SR too**, not only the e-mail thread — the
escalation stalls when support sees unanswered follow-up questions. Keep the SR
number in every message.

---

## IED device internals — the on-device engine (reverse-engineered from OSS disclosure)

> **Source of truth:** the Siemens-published open-source disclosure bundle for
> **Industrial Edge Device IPC 1.25** (`1DB-IndustrialEdgeDeviceIPC__1.25.0`,
> `Industrial Edge Device Kit 1.25.1`, `meta-cake` Debian-12 layer). This is the
> **device/firmware side** — the layer *below* your app. It exists to explain *why*
> the app rules above are the way they are. The bundle is **source disclosure only**:
> no credentials, no `/etc/shadow`, no private keys ship in it (verified).

### The 4 layers of an IED (bottom → top)

1. **Boot & update** — `swupdate 2023.12` + `EFI Boot Guard 0.19` (a **Siemens** OSS project) + `libubootenv`. Dual-slot **A/B** with automatic rollback.
2. **Base OS** — Debian 12 (bookworm) rootfs assembled by a Yocto/OE layer ("cake"). Container base is `minidebbookworm-rt35` → **PREEMPT_RT (real-time) kernel**. Hardened with AppArmor, `cryptsetup` (LUKS), `audit`, `fail2ban`, `sudo`, `argon2`, `ntpsec`.
3. **Container runtime** — `containerd` (1.5.2 / 1.6.20) + Docker (v24→v28) + `docker-cli`. Your apps and the EdgeCore services all run as **OCI containers**. NVIDIA `open-gpu-kernel-modules 545` is present → on-device GPU/AI inference.
4. **EdgeCore platform services** (containers): **Ory Hydra 2.2.0** (OAuth2/OIDC), **PostgreSQL 17.5**, **Redis 7.0.15**, **fluent-bit 3.2.7** + `rsyslog` (logs/telemetry), `openssh` + `fail2ban`, a proxy layer (`connect-proxy`), `gnupg2` (signature verification). The **Device Kit agent** (Go) is the control plane: HashiCorp **Vault** for secrets, **go-plugin** (gRPC) for extensions, **ACME/boulder + go-rootcerts** for **mTLS** to the IEM.

### Boot + update: A/B with automatic rollback

The device never overwrites the slot it is running from. An update is written to the
**inactive** slot; the bootloader only switches after the new slot is committed, and
**reverts on its own** if the new slot fails to boot. This is why a bad update cannot
brick the device — and why a firmware update always needs the *other* slot free.

`swupdate` also speaks the **Docker REST API** directly (`docker_handler.c`:
`/images/load`, `/containers/create|start|stop`). So platform images **and** app
containers are delivered through the same signed, transactional pipeline.

### The `.swu` package format + trust chain

- A `.swu` is a **cpio archive (`newc`)**. Order matters: **`sw-description` first**,
  then **`sw-description.sig`**, then the payload images (streamed, not buffered).
- `sw-description` (libconfig or JSON) is the manifest: `version`,
  `hardware-compatibility` (refuses the wrong HW revision), `images` (each with a
  **sha256**), optional `scripts` (`.lua` / shell pre/post-install), and the A/B
  `main`/`alt` sets. Because it carries every image's hash, signing the manifest seals
  the whole package.
- **"Only signed images can be installed."** Verification is `sha256` +
  **RSA (PKCS#1 or PSS)**, **CMS/PKCS#7 (X.509 cert chain)**, or **GPG**. IE uses the
  certificate/CMS path (matches the `gnupg2` + ACME/mTLS stack). Without Siemens'
  private key you cannot forge a `.swu`.

### EFI Boot Guard on-disk environment (what flips the slot)

Packed struct `BG_ENVDATA` on a FAT config partition, one per slot:

| field | meaning |
|---|---|
| `kernelfile` / `kernelparams` | UTF-16, 255 chars each |
| `in_progress` (u8) | `1` = update in flight, `0` = settled. **Hard-wired in the bootloader** — cannot be disabled. |
| `ustate` (u8) | `0 OK` · `1 INSTALLED` · `2 TESTING` · `3 FAILED` · `4 UNKNOWN` |
| `watchdog_timeout_sec` (u16) | watchdog arm time for the trial boot |
| `revision` (u32) | highest valid revision wins the boot; `0` = FAILED/lowest |
| `crc32` | integrity of the record |

**State machine:** boots the env with `in_progress==0` and the highest `revision`.
Install → new slot gets `revision+1`, `ustate=INSTALLED`. First boot → bootloader sets
`ustate=TESTING` and arms the **watchdog**. If userspace confirms → `ustate=OK` (update
"blessed"). If the boot hangs → watchdog reboots, sees `TESTING` (never confirmed),
marks the slot `ustate=FAILED, revision=0`, and the previous slot wins again → **rollback**.

### Identity: there is no root password — by design

Access is **not** a fixed factory password. It is:

- **OAuth2 / OpenID Connect via Ory Hydra** (token-based) for platform/API access,
- **mTLS** (ACME-issued certs) between the device and the **IEM**,
- **Vault** for on-device secrets,
- SSH is present but guarded by **fail2ban**; the supported admin path is the **IEM /
  Web UI**, not `ssh root@device`.

If you lose access to a device you own, the supported recovery is
**re-provisioning / factory reset via the IEM**, not credential extraction. (See the
`iem_user` recovery note above for the *management* side.)

### Practical takeaways for app authors

- Firmware updates need the **inactive A/B slot** free — plan disk/maintenance windows around it.
- An app image you ship is deployed through the **same Docker-REST path** the platform uses; it is subject to the device's signature/trust posture.
- The device is **real-time (PREEMPT_RT)** and **AppArmor-confined** — do not assume a container gets free rein; respect the platform's confinement and the RAM `mem_reservation` rules above.
- There is **no root password to hunt for**; integrate with **OAuth2/OIDC + mTLS**, and treat re-provisioning as the recovery path.

## Enrich this skill (do this every time it loads)

This skill is a **shared, growing knowledge base** about Industrial Edge, fed by
lessons from every project that loads it.

**At the end of any session where this skill was used, ask the user:**

> "Did we learn anything new about Industrial Edge in this project that isn't
> already in the `industrial-edge` skill? (New constraint, firmware gotcha,
> resource tuning insight, a name collision that bit us, etc.) If so, I'll add it."

If the user offers a lesson:
1. Confirm it's genuinely IE-general (not app-specific).
2. **Pull before you write** — follow the *Sync with GitHub* protocol below.
3. Add it to the relevant section above, or to the **Lessons learned** log below
   with a short date + context so future sessions can trust it.
4. Keep entries terse and normative. Prefer editing the rules directly when a
   lesson is a firm rule; use the log when it's a nuance/observation.
5. **Commit and push** — again per the protocol below. An enrichment that only
   exists in the working tree is invisible to every other project and machine.

### Sync with GitHub (do this around every enrichment)

This skill is checked out as a git repo and installed as a **symlink** into
`~/.claude/skills/`, so the file being edited *is* the repo working tree — and it
is shared. Other people (or the same user on another machine) push lessons to the
same `main`. Treat the remote as the source of truth and never let a local
enrichment silently clobber someone else's.

**Protocol — every time a lesson is about to be written:**

1. **Sync first.** From the skill's repo root:
   ```bash
   git status --porcelain && git fetch origin && \
     git rev-list --left-right --count origin/main...HEAD
   ```
   The counts read `<behind> <ahead>`.
2. **Behind, clean tree** → `git pull --ff-only origin main`, then **re-read the
   sections you were about to edit**. Upstream may already cover the lesson, or
   cover it differently.
3. **Behind, dirty tree / already ahead** → do *not* force anything. Fetch, then
   diff the incoming changes against the pending edit:
   ```bash
   git diff HEAD origin/main -- SKILL.md
   ```
4. **Divergence check.** If upstream touched the same rule, section, or lesson —
   especially if it states something that **contradicts** what is about to be
   written — stop and ask the user with **AskUserQuestion**. Do not pick a winner
   unilaterally: a contradicting upstream entry usually means someone hit the same
   problem on different firmware/hardware, and both facts may be true under
   different conditions. Frame the question with the *concrete* two versions, e.g.:

   > "Upstream `main` now says X about `<topic>`; the lesson we're about to record
   >  says Y. How should I resolve this?"
   >
   > - **Keep upstream, drop ours** — theirs is newer/more authoritative.
   > - **Keep ours, supersede upstream** — ours is the corrected finding; I'll edit
   >   the upstream entry and note what changed and why.
   > - **Record both, scoped** — they're both true under different conditions
   >   (firmware, IED version, hardware); I'll keep each with its qualifying context.
   > - **Let me look first** — show me the full upstream diff before deciding.

   When "record both" is chosen, always write the *distinguishing condition* into
   each entry ("on IED 1.21 with …"), otherwise the log becomes self-contradictory.
5. **Then write, commit, push.** Conventional message: one line summarizing the
   lesson (`Record IEM activation-IP binding + portable-NIC lessons`), body optional.
   ```bash
   git add -A && git commit -m "<summary>" && git push origin main
   ```
6. **Push rejected (non-fast-forward)** → someone pushed between the fetch and the
   push. Re-run from step 1 against the new `origin/main`; **never** `push --force`
   this repo — a force-push destroys other projects' recorded lessons.
7. If pushing fails for auth/network reasons, say so plainly and leave the commit
   in place — do not silently drop the enrichment.

**At load, ask before syncing — never sync automatically.** Fetching on every
load is noise: most sessions just read the skill and never write to it. So when
the skill loads, do *not* run `git fetch`. Instead ask once, up front, with
**AskUserQuestion**:

> "Sync the `industrial-edge` skill with GitHub before we start?"
>
> - **Skip sync (recommended)** — use the local copy as-is; faster, and fine for
>   read-only use.
> - **Fetch and pull if behind** — run the sync protocol above so the session runs
>   against the current shared knowledge base.

Rules:
- Ask **at most once per session**, and only for the load-time check. If the user
  skips, don't ask again mid-session.
- A skip only covers *reading*. If the session later goes to **write** a lesson,
  the sync protocol above still runs in full (steps 1–7) — pulling before writing
  is mandatory regardless of the load-time answer.
- Don't ask at all if the skill is loaded for a quick lookup inside a larger task
  where a prompt would derail the user; just use the local copy and sync at
  enrichment time.

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
  the IP change; recover from local images) → wait Ready. ⚠ **The IEM binds to the
  IP it was ACTIVATED/onboarded with** — the portal base URL, keycloak issuer and
  each IED's onboarding record all expect that address. So the target IP is NOT a
  free choice: return the IEM to its **activation IP**, and never re-onboard IEDs
  against a different one. (In iem-stellantis the VM was activated on `…179.28`; a
  temporary DHCP address was only used to get internet for the image re-pull, and
  moving back to `.28` restores the IEM's real identity so UI/onboarding line up —
  there's no stale-address mismatch to fix. If you genuinely must change the IEM's
  IP, that's an activation/DNS problem, not just a netplan one: keep a hostname that
  resolves to the activation address, or expect to re-activate/re-onboard.)
  (iem-stellantis.)
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

- [2026-07] **Make a k3s IEM VM portable across hosts — pin the NIC name.** netplan
  and k3s `flannel-iface` both reference the interface by literal name (`ens33`). Copy
  the VM to another host/hypervisor and the NIC may enumerate differently (`ens160`,
  `eth0`, …), which strands the VM (netplan finds no `ens33` → no IP) AND re-crash-loops
  k3s (`flannel-iface: ens33` now points at a nonexistent iface — offline there's no
  default route for flannel to fall back on). Fix: in netplan use `match: {name: "en*"}`
  + `set-name: ens33`, so whatever single ethernet NIC appears is renamed to `ens33` and
  both the static IP and `flannel-iface` stay valid anywhere. Match by **name-glob, not
  MAC** (a VMware "I Copied It" regenerates the MAC). Verify with a reboot: NIC comes up
  `ens33`, k3s `NRestarts` stays 0. Keep the activation IP unchanged (the IEM is bound to
  it); don't run the original and the copy on one network simultaneously (IP/identity
  clash). (iem-stellantis, Hugo, Mekatronik.)

- [2026-07] **The IEM Pro admin password is recoverable — don't redeploy to get it
  back.** `ieprovision install` prints `iem_user`'s password only to STDOUT at the
  end, but it persists in the `keycloak-secret` Secret in ns `iem`:
  `kubectl get secret keycloak-secret -n iem -o jsonpath="{.data.INITIALUSER_PASSWORD}"
  | base64 -d`. Two jsonpath traps: omitting the leading dot (`{data.…}`) →
  `unrecognized identifier data`; brackets with a bare word (`{data[…]}`) →
  `invalid array index` (brackets = array index, not map key). Both malformed
  forms **dump the whole Secret** — all Keycloak client secrets plus
  `CUSTOMER_ADMIN_PASSWORD`/`INITIALADMIN_PASSWORD` — in base64, which is encoding,
  not encryption. If that output leaves the environment (photo, screenshot, paste),
  rotate the credentials and clear shell history/scrollback. go-template form is
  the safer alternative for awkward key names. See the new *Recovering the IEM Pro
  initial admin password* subsection. (Hugo, Mekatronik.)

- [2026-08] **Physical recovery of a corrupted BX-59A** (MK830 / M Dias, Hugo,
  Mekatronik). Device stopped booting after the plant's air-conditioning failed
  (suspected thermal event), with no UI and not connected to any IEM. Findings, now
  captured in the *Physical recovery* section: the IE Hard Reset is software-only
  (IEM UI or device UI) — there is no reset button/jumper; the power button held
  > 10 s is only a hard reboot; a failed BX-59A reset showing 3 red LEDs is fixed by
  disabling xHCI Mode under `Power > XHCI USB Wake Capability` in the BIOS (`<Esc>`
  at power-on); and a device that "won't boot" after a power cut may simply be in
  the documented 2–3 h filesystem recovery. Only the Service Stick may reinstall the
  OS — anything else voids the device's authenticity in the Edge Ecosystem.

- [2026-08] **Service Stick zip ships the stick image AND the matching `.swu`.**
  Opened `simatic-ipc-ied-ss-3.0-x86-64.zip` (SHA-256 verified against SIOS):
  `*-ss-3.0.0-6-x86-64.wic.gz` (GPT: `data` FAT32 6.4 GB + `ESP`; 6.53 GB raw → ≥ 8 GB
  USB) plus `*-ied-os-3.0.0-51-x86-64.swu` (1.07 GB), i.e. the version pairing is
  pre-decided by Siemens — no need to hunt the `.swu` separately. Corollary trap: the
  stick **pins** that OS version, so reinstalling on a device running a newer build
  (the M Dias unit reported `3.2.0-17`) is a two-minor-version **downgrade**; plan the
  re-update from the IEM afterwards, or get the matching stick.

- [2026-08] **`ies-os-*.img` is the old "Industrial Edge Service Medium", not a
  Service Stick.** Reverse-engineered `ies-os-1.1.1-12-amd64.img` (Industrial OS 2.1.1
  buster, 2021): it swaps `system.hardreset.efi` into the device's `efiboota`/`efibootb`
  partitions via EFI Boot Guard and requires the follow-up "restore" menu item, and its
  payload kernels are `ied-os-1.0.0-48` / `1.2.0-57`. Do not point it at a BX-59A; use
  it only as a bootable diagnostic shell. Details in the *Physical recovery* section.

- [2026-08] **Mine the Siemens docs portal through the Fluid Topics REST API.**
  `docs.industrial-operations-x.siemens.cloud` renders nothing to a plain fetch, which
  is why answers about IE hardware are so hard to find. `/api/khub/maps` →
  `/api/khub/maps/{id}/toc` → `/api/khub/maps/{id}/topics/{contentId}/content` returns
  the full manual text. This is how the BX-59A hard-reset, LED-status, Service Stick and
  release-note facts above were sourced.

- [2026-01] **IEVD on Hyper-V + IIH S7 connector = IED UI crash (Chlorum, SR
  1-8083837145).** Activating the S7 connector against a PCS7 made the Edge Device UI
  crash and stop responding after minutes to hours, surviving a completely fresh,
  isolated IED instance. Mekatronik isolated it to **hosting the IEVD on Microsoft
  Hyper-V** — reproduced there, not on VMware/Proxmox, with matching log signatures at
  the customer. Immediate action was moving the IEVD to VMware while keeping the
  affected VM intact for analysis. Practical rule: **prefer VMware ESXi/vSphere for
  IEVD**, and when a connector destabilizes the device UI, suspect the hypervisor
  before the connector config. See also the SR-etiquette subsection — this case is
  where that escalation pattern was learned.

- [2026-08] **USB boot is disabled by default on the IPC BIOS — the Service Stick
  will not even be listed until you turn it on.** Found in the field on the M Dias
  BX-59A: the Boot Manager showed no USB device, which looks identical to a bad
  burn. Fix is `<Esc>` at power-on → `SCU > Boot` → enable USB boot → `F10` save.
  Treat it as a required step of the reinstall procedure (the Siemens manual lists
  it only as a "requirement", which is easy to skim past), and put it *before* the
  "re-burn the stick" branch of any field troubleshooting guide. (MK830 / M Dias.)

- [2026-09] **`iectl` automation of publish and rollout, verified on iectl 2.19.8**
  (mk-data-bridge, Hugo, Mekatronik). New subsection *Automating publish and
  rollout with `iectl`* under Platform ops. Headlines: pin the 32-hex application
  id per app/variant in a versioned registry and pass `--appId` (capital I) or every
  publishing machine creates a different app; `--imagetarjson` with `docker save`
  removes the Docker-TCP-2375 requirement; level 1 = `iem device-apps upload` +
  `iem job batch-create --operation installApplication|updateApplication` over N
  devices; level 2 = `iehub product-management` (create, version create, version
  upload, private-release) + `iehub library copy-product --iem-name` per IEM of the
  tenant; IEHub needs an API password from "API access management", not the SSO
  login. Traps: `iem device list` pages **5** by default; `iem job list` takes
  `--id`; iectl 2.19 marks the whole `iem` group deprecated next to `iem-v2`. The
  docs-portal REST API trick above is how the command reference was mined; the old
  `iectl iem app upload` and `publisher app-project upload catalog` are gone/EOL.
- [2026-09] **The identical `.app` file uploaded to two IEHub tenants breaks in the
  second tenant** (reproduced by Hugo, Mekatronik: house tenant, then M. Dias Branco's;
  mechanism unknown, could be app id, `versionId` or digests). Rule now under
  *Automating publish and rollout*: production apps are released only in the tenant
  that owns their IEMs (MDB's IED is managed by an IEM in MDB's own tenant), the house
  tenant gets test apps with their own ids, and a variant needed in two tenants is at
  least repackaged per tenant, preferably with its own app id. Whether repackaging
  alone is enough is untested.
- [2026-09] **IEHub product pipeline, run for real** (test app `MkTesteDevops`,
  Mekatronik tenant): `product-management create` duplicates on re-run (resolve by
  `list` + `--product-id`); binary and icon are virus-scanned asynchronously and
  release/delete fail until the scan completes; release states are `CREATED` →
  `PRIVATE_RELEASE_IN_PROGRESS` → `ECOSYSTEM_REVIEWED` (= in the tenant Library).
  Folded into the *Automating publish and rollout* subsection.
- [2026-09] **Level 1 run for real against a customer IEM** (M. Dias Branco,
  `mk-data-bridge`, Hugo, Mekatronik, 2026-09-25): v1 API JSON shapes confirmed and
  written into the level-1 table (`verionId` typo, `appVersion` = versionId); IEM app
  ids can be **base62**, not only hex; the same `.app` that breaks a second IEHub
  tenant imports fine by direct IEM upload; `iectl version` exists and the Windows
  binary runs the bash tooling under Git Bash; read-only recon commands
  (`get-statistics`, `device-job-list`, `iem-extensions list`, `device-types`) give a
  full production snapshot without touching a device, and a device's `ACTIVE` status
  is not liveness. Also seen: an IEVD type declares `maxInstalledAppCount: 20` /
  `maxRunningAppCount: 10` while the device runs 21 apps, so those limits are
  metadata, not enforced (at least on `ievd-1.26`).
- [2026-09] IED on-device engine documented from the IPC 1.25 OSS disclosure: swupdate + EFI Boot Guard A/B with auto-rollback, .swu = signed cpio (sw-description first, CMS/X.509), EFI Boot Guard env state machine (in_progress/ustate/revision/watchdog), Ory Hydra OAuth2/OIDC identity + Vault + mTLS - no root password by design. - Funny Shit / OSS audit
