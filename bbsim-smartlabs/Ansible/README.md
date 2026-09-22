# Ansible

Drives the BBSim lab from Ansible over its REST API.

## Why `ansible_connection: local`

BBSim exposes its control API over REST (`:50071`, a `grpc-gateway` in
front of the gRPC API on `:50070`). The play runs locally
(`ansible_connection: local` in `inventory.yml`) and the built-in
`ansible.builtin.uri` module makes outbound HTTP calls to it.

## Files

- `inventory.yml` — one host, `bbsim`, pointing at
  `http://localhost:50071`
- `bbsim_smoke_test.yml` — the playbook

## What the playbook does

1. `GET /v1/version` — confirms the API is reachable
2. `GET /v1/olt` — asserts the OLT's `OperState == down` (its real boot
   state without VOLTHA — see `../ARCHITECTURE.md`)
3. `GET /v1/olt/onus` — asserts 64 ONUs (matches `compose.yml`'s
   `-onu 16 -pon 4`)
4. `GET /v1/olt/onus/{SerialNumber}` — asserts that ONU is also `down`
5. `POST /v1/olt/reboot` — the one state-changing call that works without
   VOLTHA; asserts `200`
6. `GET /v1/olt/onus/BBSM99999999` (unknown serial) — asserts `500`

Every endpoint and expected value was run live against the lab (`cd
Ansible && ansible-playbook -i inventory.yml bbsim_smoke_test.yml`).
One gotcha hit and fixed along the way: `bbsim_onus.json.items` (dot access) resolves to Python's
`dict.items` *method*, not the JSON key, because `items` collides with a
built-in dict attribute — the playbook uses `bbsim_onus.json['items']`
instead.

It deliberately does **not** power an ONU on/off: `PoweronOlt` (needed
first, since PON ports start disabled) has no REST binding, and calling
`ShutdownONU` on an ONU behind a disabled PON port hangs the request
indefinitely — both verified live. See
`../ARCHITECTURE.md#request-lifecycle-what-a-test-run-actually-does-here`.

## Run it

Lab must already be up (see `../TESTING.md`, section 1).

```shell
cd Ansible
ansible-playbook -i inventory.yml bbsim_smoke_test.yml
```

**Pass:** all 9 tasks `ok`, no `failed_when`/`assert` trips.
