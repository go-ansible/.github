<p align="center"><img src="https://raw.githubusercontent.com/go-ansible/brand/main/social/go-ansible.png" alt="go-ansible" width="720"></p>

# go-ansible

**A functional-parity port of Ansible to pure Go — no Python, no C extensions, one static binary.**

Ansible's engine and module library are Python: an interpreter, a package of
dependencies, and a control node that has to carry both to run a playbook.
go-ansible reimplements the same behavior in Go with `CGO_ENABLED=0`, so the
result is a single static binary with no runtime to install and nothing to
version-match between the control node and the environment it runs in.

Parity is pursued piece by piece and only claimed where it is real. Today
that means the building blocks a playbook run depends on before a single
task executes: decrypting secrets, resolving which hosts and groups are in
scope, working out which value wins when the same variable is set in five
places, and rendering the templates that reference the result. Each one is
Ansible-compatible on its own terms — same vault format, same inventory
syntax, same precedence order, same template semantics — and ships as an
independent, importable Go library rather than as part of one large binary.

More components — module execution, the playbook engine, fact gathering, a
Galaxy client, and a CLI — are in progress; they are not yet functional and
are deliberately left off the table below.

## Repositories

| Repo | Role |
| --- | --- |
| [`vault`](https://github.com/go-ansible/vault) | Ansible Vault-compatible AES256 encryption for secrets, pure Go CGO=0. |
| [`inventory`](https://github.com/go-ansible/inventory) | Ansible-compatible inventory: INI/YAML parsers, groups, host/group vars, patterns. |
| [`vars`](https://github.com/go-ansible/vars) | Ansible variable precedence engine: facts, defaults, host/group vars, extra-vars. |
| [`template`](https://github.com/go-ansible/template) | Jinja2-compatible templating with Ansible's filter and test library, pure Go CGO=0. |
| [`brand`](https://github.com/go-ansible/brand) | Logo, favicon and social banner. |
| [`docs`](https://github.com/go-ansible/docs) | Documentation for the go-ansible organisation. |

## Standards

Pure Go, `CGO_ENABLED=0`. Every library above is validated on all six of Go's
64-bit targets — amd64, arm64, riscv64, loong64, ppc64le and s390x — with the
last three run under QEMU in CI, not just cross-compiled. BSD-3-Clause
throughout.

📖 **[go-ansible.github.io](https://go-ansible.github.io/)** · **[Documentation](https://go-ansible.github.io/docs/)**
