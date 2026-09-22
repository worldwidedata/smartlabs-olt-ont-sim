# Architecture

## Components

One long-running container, run `--privileged` (source:
`smartlabs-olt-ont-sim`'s own `Makefile` runs `docker-run` the same way),
plus a short-lived CLI tool built and run on demand:

| Component | Built from | Role |
|---|---|---|
| `bbsim` (container) | `build/package/Dockerfile` in the fork | Emulates an OLT (default: `-pon 4 -onu 16` → 4 PON ports, 16 ONUs each) speaking the `openolt` gRPC protocol that VOLTHA normally dials, plus its own control API for inspecting/driving simulated state. |
| `bbr` (CLI, run on demand) | `cmd/bbr` in the fork | "BBSim Reflector" — speaks just enough of the `openolt`/VOLTHA protocol on `:50060` to drive every ONU through EAPOL + DHCP activation, then exits. Stands in for VOLTHA without deploying it. See [Why `bbr` instead of real VOLTHA](#why-bbr-instead-of-real-voltha). |

## APIs exposed

| Port | Protocol | Purpose |
|---|---|---|
| `:50060` | gRPC, `openolt` | What a real VOLTHA `openolt` adapter connects to — and what `bbr` dials instead, to drive activation without VOLTHA. |
| `:50070` | gRPC, BBSim's own API | Backs `bbsimctl`: query/drive ONU and OLT state. |
| `:50071` | REST | `grpc-gateway` in front of `:50070` — same operations, plain HTTP/JSON. What `Ansible/` and `TESTING.md` use for inspection. |

## Data flow

```
 inspection client              bbsim container (--privileged)
 (curl / Ansible uri / bbsimctl)
      │
      │ HTTP :50071  ──┐  grpc-gateway translates
      │                ▼  REST → gRPC
      │ gRPC :50070  ──┴─────────────────────┐
      ▼                                       ▼
 ┌─────────────────────────────────────────────────┐
 │  bbsim process                                   │
 │  - OLT state machine (1 OLT)                     │
 │  - N PON ports × M ONUs each (in-memory)         │
 │  - openolt gRPC server on :50060 ◀───────────────┼──┐
 └─────────────────────────────────────────────────┘  │
                                                        │ gRPC :50060
                                          bbr (CLI, run once) ─┘
                                          "pretends to be VOLTHA":
                                          sends EAPOL + DHCP flows
```

Both `:50070` (gRPC) and `:50071` (REST) reach the same in-process state —
`:50071` is generated from the same `.proto` (`api/bbsim/bbsim.proto` +
`bbsim.yaml`'s gateway rules) as `:50070`, not a separate implementation.
E.g. `GET /v1/olt/onus` ↔ `rpc GetONUs`, `DELETE /v1/olt/onus/{SerialNumber}`
↔ `rpc ShutdownONU`.

## Request lifecycle (what a test run actually does here)

**Inspection only (REST/gRPC-on-`:50070`), no `bbr`:**

1. `GET /v1/version` — confirms the API is up.
2. `GET /v1/olt/onus` — lists the ONUs BBSim created at startup from its
   `-onu`/`-pon` flags (config only, no VOLTHA needed). **Verified: both
   the OLT and every ONU start with `OperState: "down"`** — BBSim boots
   with everything administratively disabled, waiting for whatever
   normally enables it.
3. `POST /v1/olt/reboot` — the one state-changing call that works over
   REST alone (confirmed: `200`, `{"statusCode":0,...}`).

That's the ceiling for a REST/gRPC-on-`:50070` client: **`PoweronOlt`
— the call that would actually bring the OLT and its PON ports up — has
no REST binding**, only `RebootOlt` does (`api/bbsim/bbsim.yaml`).
Confirmed side effect: calling `PoweronONU`/`ShutdownONU` on an ONU
behind a disabled PON port either fails fast (`PoweronONU` → `"PON port
0 not enabled"`) or **hangs indefinitely** (`ShutdownONU` — verified
live, no timeout). See `TESTING.md` Test 3.

**With `bbr` (over `:50060`), full activation:**

4. `bbr` connects to `:50060` and, for every configured ONU: sends the
   ONU discovery/indication BBSim expects, an EAPOL identity/challenge/
   success exchange, then a DHCP discover/offer/request/ack exchange.
5. Confirmed live: all 64 ONUs reach `EapolState:
   "eap_response_success_received"`, `DhcpState: "dhcp_ack_received"`,
   `OperState: "up"` — in ~3 seconds, verified independently via
   `GET /v1/olt/onus/{SerialNumber}`. See `TESTING.md` Test 7.

## Why `--privileged`

BBSim links `gopacket` (see `go.mod`) to build/parse raw EAPOL and DHCP
frames when emulating subscriber authentication on the NNI interface —
that needs raw-socket capability the container's default capability set
doesn't grant. `smartlabs-olt-ont-sim`'s own `Makefile`
(`docker-run-cmd += --privileged`) runs it the same way.

## Why `bbr` instead of real VOLTHA

BBSim's own docs describe EAPOL/DHCP ONU activation as something that
happens "when you enable the device in VOLTHA" — a real deployment needs
a Kubernetes cluster, Helm, and the `voltha-helm-charts` install. That's
a different order of infrastructure than a hardware-free, docker-compose
lab — and it buys nothing extra here: `bbr` ships in the same fork
specifically to drive BBSim through the exact same activation flow
(EAPOL + DHCP over the real `openolt` gRPC protocol on `:50060`) for
testing purposes, without any of that. Confirmed live — see
[Request lifecycle](#request-lifecycle-what-a-test-run-actually-does-here)
and `TESTING.md` Test 7. If a later phase needs VOLTHA/ONOS-specific
behavior `bbr` doesn't cover (e.g. real OpenFlow rule installation),
that's a real reason to revisit this; scale/activation testing alone
isn't.

## Scope boundary

This repo stops at "BBSim runs, is reachable, and can be driven through
full ONU activation via `bbr`." It does not include:

- **Real VOLTHA or ONOS.** See [above](#why-bbr-instead-of-real-voltha).
- Any Ansible/adapter code in `smartlab-runtime` (no `adapters/bbsim/`,
  no playbook dispatch, no taxonomy profile) — driving this lab from
  Smart Lab's automation is separate, later work.
- Multi-OLT, or anything `bbr`'s own scale-test mode (`-onu`/`-pon` in
  the hundreds+) doesn't already cover.
