# bbsim-smartlabs

Hardware-free [BBSim](https://github.com/opencord/bbsim) lab for testing
against an OLT/ONU device simulator. One `bbsim` container, built from
the fork this directory lives in (`smartlabs-olt-ont-sim`), exposing its
gRPC and REST APIs on `localhost`. No physical OLT/ONU hardware needed.

This directory is a thin wrapper — compose file + docs + an Ansible
example — around the BBSim source one level up.

## Prerequisites

- Linux host with `sudo` (for `--privileged`, and for `docker` unless
  your user is in the `docker` group — `sudo usermod -aG docker $USER`,
  then a new shell)
- Docker + Compose v2 plugin
- This directory checked out on the right branch, nothing else:
  ```shell
  git clone https://github.com/worldwidedata/smartlabs-olt-ont-sim.git
  cd smartlabs-olt-ont-sim && git checkout noman/bbsim
  cd bbsim-smartlabs
  ```
  `compose.yml`'s build context is `..`, the fork root this directory
  sits in — don't move this directory out on its own.

## Getting started

[`TESTING.md`](./TESTING.md) is the single start-to-end guide — build,
run, access, all 7 tests (including full ONU activation via `bbr`,
no VOLTHA needed), Ansible, teardown, troubleshooting. See
[`ARCHITECTURE.md`](./ARCHITECTURE.md) for how BBSim's gRPC/REST APIs
fit together and the design rationale. See [`Ansible/`](./Ansible/) for
the playbook itself.

## Notes

- Built from source (the fork this directory lives in), not a pinned
  public image tag — BBSim doesn't publish one at a known registry path,
  and we're already forking it. Pick up a new BBSim version by updating
  the fork checkout itself, one level up.
- Scope: local test lab + a standalone Ansible example. No real
  VOLTHA/ONOS — `bbr` (in `../cmd/bbr`) drives full ONU activation
  (EAPOL + DHCP) instead, without needing a Kubernetes cluster; see
  [`ARCHITECTURE.md`](./ARCHITECTURE.md#why-bbr-instead-of-real-voltha)
  and `TESTING.md` Test 7. No integration into `smartlab-runtime`'s
  adapter/playbook framework — that's separate, later work.
