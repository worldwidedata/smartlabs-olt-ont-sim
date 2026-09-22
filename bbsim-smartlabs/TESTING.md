# Testing Guide

> **Verification status:** every command and expected result below was
> run live against the container (build → up → curl). Two things below
> only became clear from running it, not from reading the source — see
> the callouts in Tests 3 and 5.

## 1. Create the lab

Requires Docker + Compose v2. Your user needs to be in the `docker`
group (`sudo usermod -aG docker $USER`, then a new shell) — otherwise
prefix every command below with `sudo`.

This directory lives inside the `smartlabs-olt-ont-sim` fork —
`compose.yml`'s build context is `..`, the fork root, so there's nothing
else to check out.

```shell
sudo systemctl start docker   # start the Docker daemon

cd smartlabs-olt-ont-sim/bbsim-smartlabs
docker compose build     # builds from .. (the fork root)
docker compose up -d     # -d = detached: run bbsim in the background

docker ps   # expect one container, "bbsim", Up
```

## 2. Access the lab

| What | Where / command |
|---|---|
| REST API | `http://localhost:50071` |
| gRPC API (`bbsimctl`) | `localhost:50070` |
| openolt gRPC | `localhost:50060` (what `bbr`, Test 7, dials — no VOLTHA needed) |
| Build the image | `docker compose build` |
| Start | `docker compose up -d` |
| Status | `docker ps` |
| Restart | `docker compose restart bbsim` |
| Stop | `docker compose down` |
| Logs | `docker compose logs -f bbsim` |
| Shell | `docker exec -it bbsim sh` |

```shell
curl -s http://localhost:50071/v1/version
```

`bbsimctl` ships inside the image — no separate client to install:

```shell
docker exec bbsim bbsimctl onu list
```

## 3. Run tests

### Test 1 — API reachable

```shell
curl -s http://localhost:50071/v1/version
```

**Pass:** `{"version":"", "buildTime":"", "commitHash":"", "gitStatus":""}`
— fields are blank because `docker compose build` doesn't pass
`VERSION`/`GIT_STATUS` build-args (neither does BBSim's own `make
docker-build`, so this is expected, not broken). What matters: valid
JSON, no connection error.

### Test 2 — ONU inventory matches config

Container starts with `-onu 16 -pon 4` (see `compose.yml`) → 4 PON
ports × 16 ONUs = 64 ONUs.

```shell
curl -s http://localhost:50071/v1/olt/onus | python3 -c "import sys,json; print(len(json.load(sys.stdin)['items']))"
```

**Pass:** prints `64`.

### Test 3 — OLT/ONU start powered down, and why

```shell
curl -s http://localhost:50071/v1/olt | python3 -c "import sys,json; print(json.load(sys.stdin)['OperState'])"
# down

SN=BBSM00000001
curl -s http://localhost:50071/v1/olt/onus/$SN | python3 -c "import sys,json; print(json.load(sys.stdin)['OperState'])"
# down
```

**Pass:** both print `down`. This isn't a lab misconfiguration — it's
BBSim's real startup state. `PoweronOlt`/`EnableOlt` (the calls that
would bring the OLT and its PON ports up) have **no REST binding** — only
`RebootOlt` does (see `api/bbsim/bbsim.yaml`). Normally VOLTHA enables the
OLT over gRPC; with no VOLTHA in this lab, there's no REST-reachable way
to power it on. So:

```shell
curl -s -X POST http://localhost:50071/v1/olt/onus/$SN
```

**correctly fails** with `{"code":2, "message":"PON port 0 not enabled"}`
— proving the simulator enforces OLT/PON state, not a bug.

> **Don't run `DELETE /v1/olt/onus/{SerialNumber}` (`ShutdownONU`) on an
> ONU whose PON port was never enabled — verified live, it hangs the
> request indefinitely** (no timeout, no error). If you trigger it, kill
> the container (`docker compose restart bbsim`) rather than waiting.
>
> This is the ceiling for REST/gRPC-on-`:50070` clients specifically —
> `:50060` (openolt) is a different protocol, and **Test 7** drives real
> activation over it without any REST call.

### Test 4 — Restart resilience

```shell
docker compose down    # stop and remove the container
docker compose up -d   # recreate and start it
docker ps              # expect "bbsim", Up
```

**Pass:** container `Up` again; re-run Test 2 to confirm 64 ONUs again
(state resets — BBSim keeps ONU state in memory only). Confirmed: after
a down/up cycle the ONU count was 64 again.

### Test 5 — Failure path

```shell
curl -s http://localhost:50071/v1/olt/onus/BBSM99999999
```

Querying a serial number that doesn't exist. **Pass (confirmed):** HTTP
`500`, body
`{"code":2, "message":"cannot-find-onu-by-serial-number-BBSM99999999", "details":[]}`
— bad input fails loudly with a specific message, not a silent empty
response.

### Test 6 — OLT reboot (the one mutating call that works standalone)

```shell
curl -s -X POST http://localhost:50071/v1/olt/reboot
```

**Pass (confirmed):** HTTP `200`,
`{"statusCode":0, "message":"OLT restart triggered."}`.

### Test 7 — Full ONU activation, no VOLTHA required

Tests 1–6 only ever inspect BBSim from the outside — nothing brings the
OLT/ONUs `up`, because that normally requires VOLTHA driving the
`openolt` gRPC protocol on `:50060`. Standing up real VOLTHA for that
means a Kubernetes cluster + Helm charts — far more than this lab needs.

`bbr` ("BBSim Reflector") already ships in the fork (`../cmd/bbr`, one
level up from this directory) for exactly this: it speaks just enough of
the `openolt`/VOLTHA protocol on `:50060` to drive every ONU through
EAPOL auth + DHCP, without deploying VOLTHA at all.

Build it once (output goes to `./bin`, gitignored — source stays in the
fork, untouched):

```shell
mkdir -p bin
docker run --rm \
  -v "$(pwd)/..:/src:z" \
  -v "$(pwd)/bin:/out:z" \
  -w /src golang:1.25-trixie \
  sh -c "go build -mod vendor -buildvcs=false -o /out/bbr ./cmd/bbr"
```

Run it against the live `bbsim` container — flags must match how it was
started (`-onu 16 -pon 4`, per `compose.yml`). `bbr` needs BBSim's
`configs/` in its working directory (same as `bbsim` itself), so run it
from the fork root (one level up), pointing at the binary in
`bbsim-smartlabs/bin/`:

```shell
cd ..
bbsim-smartlabs/bin/bbr -onu 16 -pon 4 -logfile /tmp/bbr.log
cd -
```

**Pass (confirmed):** log ends with

```
level=info msg="64 ONUs matching expected state" ExpectedState=dhcp_ack_received
level=info msg="BBR done!" Duration=3.099464688s
```

Verify via REST that the state actually changed, not just that `bbr`
claimed success:

```shell
curl -s http://localhost:50071/v1/olt/onus/BBSM00000001 | python3 -m json.tool
```

**Confirmed:** `OperState: "up"`, and each service under `unis[].services`
shows `EapolState: "eap_response_success_received"`,
`DhcpState: "dhcp_ack_received"` — real activation, verified independently
of `bbr`'s own report.

## 4. Automate it with Ansible

Tests 1, 3, 5, and 6 above, driven from a playbook instead of hand-typed
`curl` — see `Ansible/README.md` for exactly what each task checks:

```shell
cd Ansible
ansible-playbook -i inventory.yml bbsim_smoke_test.yml
```

**Pass:** 9 tasks, all `ok`, `failed=0`.

## 5. Teardown

```shell
docker compose down   # stop and remove the container
rm -rf bin             # drop the bbr build artifact, if you built one (Test 7)
```

## Troubleshooting

- **`ShutdownONU` (`DELETE /v1/olt/onus/{SN}`) hangs** — see the callout
  in Test 3. `docker compose restart bbsim`, don't wait it out.
- **`PoweronONU` fails with `"PON port 0 not enabled"`** — expected
  without `bbr` having enabled the OLT first. Run Test 7.
- **`docker compose build` fails with a permission error on the
  socket** — your user isn't in the `docker` group yet, or you haven't
  opened a new shell since adding yourself. See section 1.
- **SELinux host, bind mount gives `Permission denied`** — add `:z` to
  the `-v` flag (already done in Test 7's commands above).
