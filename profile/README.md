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
playbook engine — targeting a core set of roughly 40–60 `ansible.builtin`
modules and the four `ansible-*` CLI binaries (`ansible-playbook`, `ansible`,
`ansible-vault`, `ansible-galaxy`). It is **not** a port of the full Ansible
collections ecosystem (thousands of modules across hundreds of third-party
collections), and it does not include `ansible-doc`, `ansible-config`,
`ansible-pull`, or `ansible-console`. This was a deliberate scope decision,
not an oversight.

Parity is pursued piece by piece and only claimed where it is real: some
playbook keys (`roles`, `tags`, `serial`, `delegate_to`, `vars_files`,
`include_tasks`/`import_tasks`) are parsed — some not even that — but not yet
acted on by the engine. See the
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
| [`cli`](https://github.com/go-ansible/cli) | The four CLI binaries: `ansible-playbook`, `ansible`, `ansible-vault`, `ansible-galaxy`. |

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
throughout.

📖 **[go-ansible.github.io](https://go-ansible.github.io/)** · **[Documentation](https://go-ansible.github.io/docs/)**
