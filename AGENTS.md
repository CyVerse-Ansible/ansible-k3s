# AGENTS.md — ansible-k3s

Agent-facing conventions for this role. User-facing docs (inventory shape, variable
table, example playbook) live in `README.md`; do not duplicate them here.

## 1. Purpose

Installs k3s as a single master plus optional agents from the official
`https://get.k3s.io` script, driven by `/etc/rancher/k3s/config.yaml` templates. Optional
features, all off by default: PostgreSQL datastore (installed on the first master),
NVIDIA GPU support via the GPU Operator Helm chart (26.x, containerd only), ufw firewall
management (Ubuntu only), Docker runtime (`--docker`, not combinable with GPU).
Helm is installed on masters whenever the role runs. Inventory must define
`k3s_masters`, `k3s_agents`, and `k3s_cluster` (see README).

Supported platforms: Ubuntu 22.04 / 24.04, Rocky and AlmaLinux 9 / 10 (enforced by an
assert in `tasks/main.yml`; matches `meta/main.yml`). PostgreSQL comes from PGDG, pinned
major 17 (`K3S_POSTGRESQL_MAJOR`).

Deliberately NOT done: installing Docker (must pre-exist when `K3S_DOCKER_ENABLE` is
true), HA / multiple masters (`postgresql.yml` and token lookup assume
`groups['k3s_masters'][0]`), installing NVIDIA host drivers or the host `nvidia-container-toolkit`, managing firewalld rules on
RedHat family (it is stopped and disabled, not configured), uninstall unless
`K3S_FORCE_UNINSTALL` is set.

## 2. Repository layout

| Path | Holds | Put new things here when… |
|---|---|---|
| `tasks/main.yml` | Entry point: loads `vars/`, platform/GPU asserts, sysctl, GPU host prep (CDI spec), `/opt/k3s`, then `import_tasks` for postgresql, firewall, uninstall, helm, k3s-install | a step must run on every host before install |
| `tasks/k3s-install.yml` | Downloads installer once (controller), master block, token fetch, GPU import, agent block | the change touches the k3s install itself |
| `tasks/k3s-install-gpu-nvidia-drivers.yml` | Helm install of the GPU Operator (masters only) | GPU/Helm workloads |
| `tasks/firewall-Ubuntu.yml`, `tasks/firewall-Redhat.yml` | Per-distro firewall / selinux | distro-specific firewall logic |
| `tasks/postgresql.yml` | PGDG install (gated on `K3S_POSTGRESQL_INSTALL`), password generation, user + db on first master | datastore changes |
| `tasks/postgresql-repo-Debian.yml`, `tasks/postgresql-repo-RedHat.yml` | PGDG repo setup, `include_tasks` by `ansible_os_family` | per-family repo logic |
| `defaults/main.yml` | Public `K3S_*` defaults | every new user-facing variable |
| `vars/<Distribution>[-<major>].yml` | `POSTGRESQL_PACKAGES`, `POSTGRESQL_SERVICE`, `POSTGRESQL_SETUP_BIN` (RedHat), `ansible_python_interpreter` | distro-specific values; never in `defaults/` |
| `templates/*.j2` | k3s master/agent `config.yaml`, GPU Operator values | anything rendered onto the host |
| `handlers/main.yml` | Empty | — (see §4) |
| `tests/test.yml`, `tests/inventory` | Galaxy skeleton, `hosts: localhost` | — |
| `requirements.yaml` | Collections (§6) | any new collection |

No `files/`, `molecule/`, `meta/argument_specs.yml`, CHANGELOG, lint config, or CI exist (§10).

## 3. Variables

- **Public API**: `K3S_*` in `defaults/main.yml` plus the undefined-by-default ones the
  tasks test with `is defined` (`K3S_VERSION`, `K3S_MASTER_IP`, `K3S_CLUSTER_TOKEN`,
  `K3S_CLUSTER_CIDR`, `K3S_FLANNEL_BACKEND` (aliased to `K3s_FLANNEL_BACKEND` in `tasks/main.yml`), `K3S_DOCKER_ENABLE`, `K3S_FIREWALL_MANAGE`,
  `K3S_FIREWALL_ADD_PORTS`, `K3S_FORCE_UNINSTALL`, `K3S_POSTGRESQL_PASS`). Uppercase
  `K3S_` prefix is the convention. Known inconsistency: templates still read the
  legacy mixed-case `K3s_FLANNEL_BACKEND`; README lists `K3S_REGISTRIES_MIRRORS`, which no task implements.
- **Internal**: `INTERNAL_DOCKER_ENABLE` (defaults + vars + set_fact in `tasks/main.yml`),
  `POSTGRESQL_PACKAGES` (vars only), lowercase facts set at runtime (`k3s_version`,
  `k3s_docker_flag`, `k3s_master_ip`, `k3s_master_url`, `k3s_token`,
  `k3s_master_token_file`, `HELM_INSTALL_DIR`, `gpu_operator_installed`, `helm_found`,
  `pass_file_check`, `k3s_postgres_pass`, `nvidia_runtime_bin`).
  Never document these in README; never read them from a playbook.
- **defaults vs vars**: `defaults/` = overridable by users. `vars/` = distro facts chosen
  by `with_first_found` in `tasks/main.yml` (`<Distribution>-<major>.yml` →
  `<Distribution>.yml` → `<os_family>.yml`). Docker is off by default on every
  supported platform (`INTERNAL_DOCKER_ENABLE: false` in defaults).
- **Adding a public variable**: add to `defaults/main.yml` with a one-line comment if
  non-obvious → add a row to the README table → reference it in tasks/templates with
  `|bool` for booleans → if it may be undefined, guard with `is defined`. No
  `argument_specs.yml` or Molecule verify exists yet to update (§10).
- **Renaming** a `K3S_*` variable is a breaking change: do not do it without discussion.
  If approved, keep the old name working via `set_fact` and note the deprecation in README.

## 4. Task conventions

- **Module names**: FQCN for collection modules (`ansible.posix.selinux`,
  `community.general.snap`, `kubernetes.core.helm`). Known inconsistency: builtins and
  some collection modules use short names (`shell`, `file`, `copy`, `systemd`, `ufw`,
  `sysctl`, `postgresql_user`, `postgresql_db`, `service`). Use FQCN in new tasks; do not
  mass-rewrite existing ones in an unrelated change.
- **Naming**: `MODULE_IN_CAPS; what it does` (e.g. `SHELL; helm repo update`,
  `SET_FACT; obtain the k3s_master_ip`). Blocks are `Block ...` or unnamed. Follow the
  `CAPS; description` form for new tasks; name every block.
- **Idempotency**: templates, `file`, `package`, `helm` are idempotent. Known
  inconsistencies: `shell` tasks without `creates`/`changed_when` (helm repo update, k3s
  installer, token `cat`, password `echo`) always report changed; the installer always
  re-runs on masters when `K3S_MASTER_INSTALL` is true. New `shell`/`command` tasks must
  set `creates:` or `changed_when:`; prefer a module.
- **Tags**: none are used. Do not introduce a partial scheme.
- **Handlers**: `handlers/main.yml` is empty; restarts are inline with
  `when: <registered>.changed` (none remain in the role today). Keep that pattern
  unless converting all restarts at once.
- **changed_when / failed_when**: `failed_when: false` + `ignore_errors` for probes
  (`helm list`, `command -v helm`, uninstall). Use `failed_when: false` for probes, not
  `ignore_errors`, in new code.
- **Secrets**: token/password tasks and the `config.yaml` templates use `no_log: true`;
  the debug task prints only the master URL. Add `no_log: true` to any new task that
  handles `K3S_POSTGRESQL_PASS`, `k3s_token`, or `K3S_CLUSTER_TOKEN`.
- **Check mode**: not supported; `shell` tasks whose output is `register`ed and used
  later will break `--check`. Do not claim check-mode support.
- **OS conditionals**: `ansible_distribution == "Ubuntu"` / `"AlmaLinux"` /
  `"Rocky"` with `ansible_distribution_major_version|int` comparisons; file selection via
  `with_first_found` on distribution name. `ansible_os_family` is only a fallback. Match
  this; do not mix in `ansible_facts['os_family']` style in the same file.
- **Host-role conditionals**: `inventory_hostname in groups['k3s_masters']` /
  `groups['k3s_agents']`; first-master-only is `groups['k3s_masters'][0]` with
  `delegate_to` + `run_once`.

## 5. Testing

No Molecule, lint config, or CI exists. What can be run today:

```bash
ansible-galaxy collection install -r requirements.yaml
ansible-playbook --syntax-check -i tests/inventory tests/test.yml   # role must be resolvable as "k3s"
yamllint .                                 # upstream defaults; expect warnings (no .yamllint yet)
ansible-lint                               # upstream defaults; expect FQCN/name/no-changed-when findings
```

`tests/test.yml` targets `localhost` and needs the role symlinked or installed under the
name `k3s` (e.g. `ln -s "$PWD" ../k3s` or `ANSIBLE_ROLES_PATH=..`). It is a syntax
fixture, not a functional test.

"Passing" before a change is done: syntax check clean, `ansible-lint` introduces no new
findings beyond the pre-existing ones, and a real run against a disposable Ubuntu 22.04
VM (master-only inventory) with the feature flags you touched. Run twice; the second run
must not fail. Do not run `tests/test.yml` against your workstation — it installs k3s.

## 6. Dependencies

`requirements.yaml` (added 2026-10-06) lists the collections the tasks use:
`ansible.posix` (selinux, sysctl), `community.general` (ufw, snap),
`community.postgresql` (postgresql_user, postgresql_db), `kubernetes.core` (helm,
helm_repository). Install: `ansible-galaxy collection install -r requirements.yaml`.
`kubernetes.core.helm` needs the `helm` binary on the master; the role installs it.
No role dependencies (`meta/main.yml` `dependencies: []`). Docker is an external
prerequisite when `K3S_DOCKER_ENABLE` is true.

`min_ansible_version` in `meta/main.yml` is `2.13.11` (collection floor); tested on
ansible-core 2.21. Host-side needs: `curl`, `jq` (installed for GPU), `python3-psycopg2`
(from `POSTGRESQL_PACKAGES`), `snapd` on Ubuntu for the helm fallback.

## 7. Making changes

1. Decide the file by §2; keep master/agent/GPU/firewall/postgres logic in their files.
2. New variable → `defaults/main.yml` + README table + `|bool` / `is defined` guards (§3).
3. New distro → `vars/<Distribution>[-<major>].yml` with `POSTGRESQL_PACKAGES`, and extend
   the firewall `when:` lists in `tasks/main.yml` if applicable; update README platforms.
4. New collection module → add to `requirements.yaml` and use its FQCN.
5. Templates: edit the `.j2`, not `gpu-operator-values.yaml`; keep `is defined and |bool`
   guards for optional variables.
6. Run syntax check and `ansible-lint`; run on a disposable VM twice (§5).
7. No CHANGELOG; describe behaviour changes in the commit message and README if user-visible.

## 8. Things that break easily

- **Order in `tasks/k3s-install.yml`**: master block → `k3s_master_ip` facts → token
  `cat` from master[0] → GPU → agents. Agents need `k3s_master_url` and `k3s_token`; the
  token file only exists after the master's k3s service is `active`.
- **`K3S_MASTER_INSTALL: false`** skips the master block but the token fetch still runs;
  it fails if k3s was never installed on master[0].
- **Installer download** happens on the controller to `/tmp/k3s_install.sh` with
  `creates:` — a stale copy persists across runs; delete it to refresh.
- **`ansible_default_ipv4`** is used for the master URL unless `K3S_MASTER_IP` is set;
  multi-NIC masters need `K3S_MASTER_IP`.
- **Templates are coupled to facts**: `agent-k3s-config.yaml.j2` reads
  `k3s_master_token_file.stdout`; `master-k3-sconfig.yaml.j2` reads
  `K3S_POSTGRESQL_PASS`, which only exists after `postgresql.yml` ran on master[0].
- **PostgreSQL password** is generated once into `/opt/k3s/postgres-<user>.pass` and
  reused; changing `K3S_POSTGRESQL_USER` creates a new file and new password.
- **`K3S_DOCKER_ENABLE` vs `INTERNAL_DOCKER_ENABLE`**: templates check
  `K3S_DOCKER_ENABLE`, tasks check `INTERNAL_DOCKER_ENABLE`. Setting Docker via
  `vars/` alone renders no `docker:` line in config.yaml (known inconsistency).
- **GPU Operator**: chart `v26.7.1` (`K3S_NVIDIA_GPU_OPERATOR_CHART_VERSION`), containerd
  only; `K3S_GPU_ENABLE` with Docker fails an assert. k3s ignores
  `/etc/containerd/config.toml` and detects `nvidia-container-runtime` itself. The helm task
  has no install guard, so a chart bump upgrades the release (25.x to 26.x is a major
  upgrade). Bump the chart only after a GPU VM test.
- **CDI spec**: with the host toolkit (`K3S_NVIDIA_GPU_OPERATOR_TOOLKIT: false`) and CDI
  enabled, `tasks/main.yml` runs `nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml` on
  hosts that have `/usr/bin/nvidia-container-runtime`; others are skipped. Unverified on a
  GPU VM; drop the task if the operator generates the spec itself.
- **PostgreSQL / PGDG**: repo, packages, initdb (RedHat) and service start run only when
  `K3S_POSTGRESQL_INSTALL` is true; user/db creation runs whenever `K3S_POSTGRESQL_ENABLE`
  is true. The repo file is chosen with `include_tasks` on `ansible_os_family`. Existing
  distro-PostgreSQL data is not migrated to 17.
- **Helm install path**: `HELM_INSTALL_DIR=/usr/bin` only when `/usr/local/bin` is not in
  `ansible_env.PATH` (sudo PATH on RedHat family); the snap/package fallback runs only if both
  `command -v helm` probes fail.
- **Firewall-Redhat runs unconditionally** on Alma/Rocky (no `K3S_FIREWALL_MANAGE`
  check) and sets selinux disabled + crypto policies LEGACY.
- **`vars/Ubuntu.yml` forces** `ansible_python_interpreter: /usr/bin/python3`.

## 9. Out of scope / do not

- Do not rename or remove `K3S_*` variables, inventory group names (`k3s_masters`,
  `k3s_agents`, `k3s_cluster`), or `/opt/k3s` file names without discussion.
- Do not add collections, roles, Galaxy roles, or new Helm charts without discussion.
- Do not hardcode IPs, tokens, or passwords; do not commit `/opt/k3s/*.pass` content,
  kubeconfigs, or node tokens. Do not add tasks that print secrets.
- Do not add HA-master logic piecemeal; it requires rework of `postgresql.yml`, token
  handling, and templates together.
- Do not make the role install Docker or NVIDIA host drivers.
- Do not reformat or mass-FQCN existing tasks in a feature change.
- Do not edit `docker_runs_private.md` (untracked, local workflow file).

## 10. Active Backlog & TODOs

Recommended, not yet done. Pick up with user agreement; each is its own change.

- [ ] **Molecule**: add `molecule/default/` (docker driver, Ubuntu 22.04 image with
  systemd) running `converge` → `idempotence` → `verify` (k3s service active, `kubectl
  get nodes` Ready). Add a `gpu` scenario only when a GPU runner exists.
- [ ] **Lint config**: add `.ansible-lint` and `.yamllint`; fix FQCN, `name`,
  `no-changed-when`, and `risky-file-permissions` findings in one pass.
- [ ] **CI**: GitHub Actions running lint + syntax check + Molecule default.
- [ ] **`meta/argument_specs.yml`** for all `K3S_*` variables.
- [x] **Secrets**: `no_log: true` on token/password tasks; remove the token `debug`.
- [ ] **Idempotency**: `changed_when`/`creates` on shell tasks; `K3S_MASTER_INSTALL`
  skip-if-installed check.
- [x] Remove dead code: `tasks/postgresql-install-Ubuntu.yml`, commented-out `set_fact`
  blocks in `k3s-install.yml`, root `gpu-operator-values.yaml`; fix duplicate sysctl.
- [ ] Implement or drop `K3S_REGISTRIES_MIRRORS` (README only); finish the `K3S_FLANNEL_BACKEND`
  rename (alias exists, templates still read the old name); wire `K3S_DOCKER_ENABLE` into templates via
  `INTERNAL_DOCKER_ENABLE`.
- [ ] Gate `firewall-Redhat.yml` on `K3S_FIREWALL_MANAGE`; (the `apt-key` item is
  done: PGDG now uses a signed-by keyring).
- [ ] CHANGELOG.md.
