---
name: industrial-edge
description: Deploy/package apps for Siemens Industrial Edge (IE). Industrial Edge apps are Docker Compose stacks with strict quirks that differ from normal Docker. Use whenever building, packaging, writing a docker-compose for, or troubleshooting deployment of an app that will run on Industrial Edge / an industrial edge device — anything mentioning "industrial edge", "IE app", "edge device", or IE-specific compose constraints.
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
   compose file. Not v3.x. The v2.4 schema is what enables the resource keys IE
   requires (`mem_limit`, `mem_reservation`, `cpus`) at the service level.

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

7. **No `build:` — images must be prebuilt and pulled.** IE compose cannot build
   images. Every service must reference an already-built image from a registry the
   device can pull from (e.g. `mekatronik/<app-name>:<version>`). Build & push the
   image separately (a `build.sh` is the convention), pin an explicit version tag
   in the compose (avoid bare `:latest` for reproducible installs), and never use
   a `build:` key.

8. **Multiple containers per app are allowed — but do NOT share a volume between
   containers.** A single IE app's compose can define several services. However,
   mounting the *same named volume* into two containers works on some device
   firmware versions and fails on others. Design so each volume is used by exactly
   one container. If two containers must exchange data, use the network
   (`proxy-redirect`) or give each its own volume — never a shared mount.

---

## Reference skeleton

```yaml
version: '2.4'

services:
  myapp:
    image: myregistry/myapp:1.0.0   # prebuilt + pushed; NO `build:` key
    restart: unless-stopped
    networks:
      - proxy-redirect
    environment:
      - SOME_SETTING=value        # hard-code, don't rely on IE env UI
    volumes:
      - myapp-data:/data          # named volume, app-prefixed name
    mem_reservation: 256m         # guaranteed floor (must still run when capped here)
    mem_limit: 1g                 # hard ceiling
    cpus: 0.8                     # optional CPU cap (v2.4 supports it)

volumes:
  myapp-data:                     # globally-unique, app-prefixed

networks:
  proxy-redirect:
    external: true
```

---

## Pre-deploy checklist

- [ ] `version: '2.4'` at top
- [ ] No `build:` key — every service uses a prebuilt, pushed image with a pinned version tag
- [ ] Zero bind mounts / `./` volumes
- [ ] Every volume named `<appprefix>-*` and declared under top-level `volumes:`
- [ ] No volume mounted into more than one container
- [ ] Every service has both `mem_reservation` and `mem_limit`
- [ ] App still runs (even if degraded) at `mem_reservation`
- [ ] `proxy-redirect` declared `external: true` and attached where needed
- [ ] If field/L2 access is needed: uses external `zzz_layer2_net1` (not macvlan),
      `zzz_`-prefixed so it lands on eth1; `cap_add` only for raw-L2 protocols
- [ ] Config/secrets hard-coded in compose, not in `.env` or IE env UI

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
