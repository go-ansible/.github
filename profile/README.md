<p align="center"><img src="https://raw.githubusercontent.com/go-ansible/brand/main/social/go-ansible.png" alt="go-ansible" width="720"></p>

# go-ansible

**A pure-Go, CGO_ENABLED=0, functional-parity port of Ansible — no Python, no C extensions, one static binary.**

Ansible's engine and module library are Python: an interpreter, a package of
dependencies, and a control node that has to carry both to run a playbook.
go-ansible reimplements the same behavior in Go, so the result is a single
static binary with no runtime to install and nothing to version-match between
the control node and the environment it runs in.

**Scope, stated plainly:** this is a port of `ansible-core`'s own surface —
Vault-compatible secrets, inventory, the variable precedence ladder,
Jinja2-compatible templating, module execution, fact gathering, and the
playbook engine — plus all 8 real `ansible-*` CLI binaries (`ansible`,
`ansible-playbook`, `ansible-vault`, `ansible-galaxy`, `ansible-pull`,
`ansible-doc`, `ansible-config`, `ansible-console`). It is **not** a port of
the full Ansible collections ecosystem (thousands of modules across hundreds
of third-party collections) — module coverage is pursued deliberately,
collection by collection, and only claimed where it is real.

The full `ansible.builtin` collection (62 real modules plus all 9
playbook-engine directives: `add_host`, `group_by`, `import_playbook`,
`import_role`, `import_tasks`, `include_role`, `include_tasks`,
`include_vars`, `meta`) is complete — 71/71, verified against a real
`ansible-core` installation, not just internal review. Beyond builtin, the
`ansible.posix` collection is fully ported (14 modules; `synchronize` is an
honest, always-failing stub rather than a silent approximation — real
`synchronize` runs rsync from the controller directly against the target's
SSH endpoint, which this port's connection abstraction cannot expose from
inside a module), and thirteen curated batches of `community.general` (485 of
577 total) are shipped — package managers, language/dev tooling,
filesystem/storage, networking, system/service management, SELinux,
read-only facts, Pacemaker cluster management (including stonith/
fencing), LDAP, FreeIPA, Redis,
Consul, Kerberos, desktop/system config, LXD/LXC containers, HashiCorp
Nomad, database admin, RHEL subscription management, AIX, Elastic Stack
plugins, InfluxDB, Icinga2, Kopia backup, version control (bzr/hg), web/
app servers, ISO tools, provisioning (including Stacki), Django extensions, IPMI, more
niche package managers, process supervision, Univention UDM, XenServer,
GitHub, GitLab, Keycloak, Scaleway, Huawei Cloud, Jenkins, Equinix Metal,
Linode, OpenNebula, Rundeck, Alibaba Cloud, IBM SoftLayer,
1Password, Pulp, Twilio, Aerospike, Alerta, Heroku, Mattermost, New
Relic, DNSimple, ipinfo.io, Cloudflare, OVHcloud, Dell EMC VNX, IBM
Spectrum Accelerate, HPE 3PAR, the Redfish hardware-vendor family (Dell
iDRAC, HPE iLO, Lenovo XCC), Jira, Honeybadger, Rollbar, Memset,
Airbrake, UptimeRobot, SendGrid, Circonus, Kea DHCP, Broadcom Spectrum,
FusionLayer Infinity IPAM, IP2Location, and a
handful of misc/local-CLI modules
(`bower`/`easy_install`/`file_remove`/`hponcfg`/`ansible_galaxy_install`
among them), deliberately excluding SaaS-API wrappers that have no
comparable official CLI (Slack, PagerDuty, Datadog, etc.) — **except**
GitHub/GitLab/Keycloak/Jenkins, whose official
`gh`/`glab`/`kcadm.sh`/`jenkins-cli.jar` CLIs this port shells out to
directly, **except** Scaleway/Huawei Cloud/Equinix Metal/Linode/
Alibaba Cloud/IBM SoftLayer, cloud-VPS providers this port's own docs
once named as an exclusion example — reconsidered because each also
ships a genuine official CLI (`scw`/KooCLI/`metal`/`linode-cli`/
`aliyun`/`slcli`), and **except** the Redfish hardware-vendor family
(also once named as an exclusion example), reconsidered because HPE's
`ilorest` and Lenovo's `OneCli` are genuine, strong-fit official
Redfish CLIs and Dell's `racadm` is a real official (if older,
parallel) interface — the same CLI-substitution approach already used
for Consul/Redis/Terraform/Icinga2/Kopia, extended across four
explicit, deliberate scope decisions (not applied to platforms with no
comparable official CLI — Western Digital's Redfish gear was checked
and found to have none). Two platforms were investigated and found to
have **no** usable official CLI for what their modules need: Pritunl
(`pritunl_org`/`pritunl_org_info`/`pritunl_user`/`pritunl_user_info`)
and two of OVHcloud's three modules
(`ovh_ip_failover`/`ovh_ip_loadbalancing_backend` — `ovh_monthly_billing`
has full coverage) — all fail loud rather than fake parity, confirmed
gaps, not assumed ones. Rollbar's `rollbar-cli` and Memset's `ma-shell`
were both confirmed, by reading their real source, to have **no**
environment-variable alternative to their access-token/API-key argv
flag — a documented, unavoidable exception to this project's own
no-secrets-in-argv rule, not an oversight. **561 modules registered in
total.** The playbook engine also supports `strategy: free` (each host runs
its whole task list and its own notified handlers independently, with no
per-task barrier across hosts — any other named strategy is still rejected
with an explicit error, never silently accepted), nested role variable
scoping that composes to any depth (a role included from inside another
role's own tasks merges its defaults/vars on top of the enclosing role's
instead of replacing them), and `register:` on a `setup:`/`set_fact:` task
correctly nests its result under `ansible_facts` — all three verified against
a real `ansible-playbook`. What's
still out: ~92 more `community.general` modules — an
increasingly SaaS-only remainder — plus every
cloud-provider collection (amazon.aws/azure/google.cloud and similar —
these need real Go SDK bindings per provider, a fundamentally different kind
of work, not yet started). See the
**[engine feature matrix](https://go-ansible.github.io/)** on the landing page
for the current, code-checked status of each, and
**[BENCHMARKS.md](https://github.com/go-ansible/.github/blob/main/BENCHMARKS.md)**
for a measured comparison against real
`ansible-core` — binary/distribution size, execution latency, and a real
`FROM scratch` container proof (with the one shell-module limitation that
implies, reported honestly).

## Repositories

| Repo | Role |
| --- | --- |
| [`vault`](https://github.com/go-ansible/vault) | Ansible Vault-compatible AES256 encryption for secrets, pure Go CGO=0. |
| [`inventory`](https://github.com/go-ansible/inventory) | Ansible-compatible inventory: INI/YAML parsers, groups, host/group vars, patterns. |
| [`vars`](https://github.com/go-ansible/vars) | Ansible variable precedence engine: facts, defaults, host/group vars, extra-vars. |
| [`template`](https://github.com/go-ansible/template) | Jinja2-compatible templating with Ansible's filter and test library, pure Go CGO=0. |
| [`modules`](https://github.com/go-ansible/modules) | Ansible module execution protocol plus the core module library. |
| [`facts`](https://github.com/go-ansible/facts) | Fact gathering (the `setup` module equivalent), pure Go CGO=0. |
| [`playbook`](https://github.com/go-ansible/playbook) | Playbook/task/handler execution engine: loops, conditionals, blocks, handlers, become. |
| [`cli`](https://github.com/go-ansible/cli) | All 8 CLI binaries: `ansible`, `ansible-playbook`, `ansible-vault`, `ansible-galaxy`, `ansible-pull`, `ansible-doc`, `ansible-config`, `ansible-console`. |

All eight core repositories are shipped and tagged. The low-level SSH/local/
become connection layer lives outside this org, in the project-neutral
[`go-remoteexec/transport`](https://github.com/go-remoteexec/transport)
(shared with `go-puppet-bolt/bolt`); `modules`, `playbook`, and `cli` depend
on it directly. [`brand`](https://github.com/go-ansible/brand) and
[`docs`](https://github.com/go-ansible/docs) hold logo/site assets and
documentation — not port code.

## Standards

Pure Go, `CGO_ENABLED=0`. Every library above is validated on all six of Go's
64-bit targets — amd64, arm64, riscv64, loong64, ppc64le and s390x — with the
last three run under QEMU in CI, not just cross-compiled. BSD-3-Clause
throughout. `cli` also publishes a multi-arch `FROM scratch` OCI image,
[`ghcr.io/go-ansible/cli`](https://github.com/go-ansible/cli/pkgs/container/cli)
(all six — amd64/arm64/riscv64/loong64/ppc64le/s390x — the build stage cross-
compiles from the runner's own native architecture instead of running under
QEMU for every target, which is what makes loong64 possible at all: the
official `golang` image itself publishes no `linux/loong64` manifest), on
every version tag.

📖 **[go-ansible.github.io](https://go-ansible.github.io/)** · **[Documentation](https://go-ansible.github.io/docs/)**
