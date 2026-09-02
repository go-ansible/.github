# Benchmarks: go-ansible vs. real ansible-core

Real, measured numbers comparing `go-ansible`'s `ansible-playbook` binary
(the `cli` repo) against a real `ansible-core` install. Every number below
came from a command that is printed next to it — rerun it yourself and you
should land close to these figures.

**Single machine, not a statistically rigorous benchmark suite.** One Apple
M4 Max, macOS 26.6.2, Go 1.26.4, Docker Desktop 29.7.2 (server 29.7.2, arm64
Linux VM), measured 2026-09-02. `ansible_connection: local` on both sides —
no network variance, no SSH round trips. N=10 runs per figure, first run
discarded as warm-up, min/median/p90 taken over the remaining 9. That is
enough to see the shape of the difference (which is large — see below), not
enough to defend a fourth significant digit.

## ansible-core reference install

A real `ansible-core` in a Python venv, installed with:

```
python3 -m venv ansible-venv
ansible-venv/bin/pip install ansible-core
```

```
$ ansible-venv/bin/ansible-playbook --version
ansible-playbook [core 2.21.3]
  python version = 3.14.5 (Clang 21.0.0)
  jinja version = 3.1.6
  pyyaml version = 6.0.3 (with libyaml v0.2.5)
```

`pip list` inside the venv: `ansible-core 2.21.3`, `cryptography 50.0.1`,
`cffi 2.1.1`, `pycparser 3.0`, `Jinja2 3.1.6`, `MarkupSafe 3.0.3`,
`PyYAML 6.0.3`, `packaging 26.3`, `resolvelib 1.2.1`, `pip 26.1.1`.

**Note on `ansible-playbook --version`:** it errors with `ERROR: Ansible
requires blocking IO on stdin/stdout/stderr` when stdin/stdout aren't real
terminals/files (true in most CI shells and agent harnesses, not just this
one). Every command below redirects `< /dev/null` to work around it —
that's a real ansible-core quirk, not a go-ansible one, but worth knowing if
you rerun these.

## 1. Binary / distribution size

go-ansible ships one binary; real Ansible needs a Python interpreter plus a
venv of pip packages. Built with:

```
cd cli
GOWORK=off go build -o go-ansible-playbook ./cmd/ansible-playbook
GOWORK=off GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build -o go-ansible-playbook-linux-amd64 ./cmd/ansible-playbook
```

| Artifact | Size |
| --- | --- |
| `go-ansible-playbook`, darwin/arm64, unstripped | 14M (`du -h`) |
| `go-ansible-playbook`, darwin/arm64, `-ldflags="-s -w"` | 9.5M |
| `go-ansible-playbook`, linux/amd64, `CGO_ENABLED=0`, unstripped | 14M |
| `go-ansible-playbook`, linux/amd64, `CGO_ENABLED=0`, `-ldflags="-s -w"` | 9.9M |

`file` on the linux/amd64 build confirms it: `ELF 64-bit ... x86-64,
statically linked` — no dynamic interpreter, no libc dependency to satisfy
at runtime.

Real Ansible's footprint, measured with `du -sh`:

| What | Size |
| --- | --- |
| Whole venv (`ansible-venv/`, symlinked interpreter + site-packages) | 51M |
| `site-packages/ansible/` (the `ansible-core` package itself) | 15M |
| `site-packages/ansible_core-2.21.3.dist-info/` | 252K |
| `site-packages/ansible_test/` (bundled test-runner, ships with the package, not needed to run playbooks) | 4.8M |
| `site-packages/cryptography/` (Vault's AES backend — C extension via a prebuilt wheel) | 13M |
| `site-packages/jinja2/`, `yaml/`, `cffi/`, `pycparser/`, `packaging/`, `resolvelib/`, `markupsafe/`, `pip/` | remainder of the 51M |

That 51M venv is *not* the whole story: a Python venv only symlinks the
interpreter, it doesn't copy it. The interpreter this venv actually runs —
`readlink -f ansible-venv/bin/python3.14` →
`/opt/homebrew/Cellar/python@3.14/3.14.5/.../Versions/3.14` — is a separate
**81M** on disk (`du -sh` on that path). A machine that doesn't already have
Python 3.14 installed pays that 81M too. So the honest total for "what does
a fresh machine need to become an Ansible controller" is **51M (venv) + 81M
(interpreter) ≈ 132M**, against go-ansible's **one ~14M static binary** (9.5M
stripped) — roughly **9–14× smaller**, and that's before counting that real
Ansible's controller also needs network access to a package index the first
time, where go-ansible needs nothing beyond the binary itself.

This also doesn't count what real Ansible needs on the *managed* side for
many modules (a Python interpreter on the target, since Ansible copies a
Python module script over and executes it there) — go-ansible's modules run
their logic on the controller and only reach the target through
shell/SFTP-style primitives (see `modules/` README), so the target needs
nothing but a shell.

## 2. Execution latency

One playbook, run unmodified against both tools (YAML is YAML; no syntax
had to change), targeting `localhost` with `ansible_connection: local`.
8 tasks: `debug` ×3, `set_fact`, `file`, `copy`, `template`, `command` (with
`register`). Full playbook and inventory are in `docker/` in this PR's diff
history / reproducible from the snippet below.

```yaml
- hosts: all
  gather_facts: false
  vars: {tool_name: benchmark, run_mode: comparison}
  tasks:
    - {name: debug a static message, debug: {msg: "starting benchmark run"}}
    - {name: set a fact, set_fact: {greeting: "hello from {{ tool_name }}"}}
    - {name: debug the fact, debug: {msg: "{{ greeting }}"}}
    - {name: ensure work directory exists, file: {path: work, state: directory}}
    - {name: copy a small file, copy: {content: "benchmark payload\n", dest: work/copied.txt}}
    - {name: render a template, template: {src: bench.j2, dest: work/rendered.txt}}
    - {name: run a shell command, command: echo "command task ran", register: cmd_out}
    - {name: debug the command output, debug: {msg: "{{ cmd_out.stdout }}"}}
```

Measured with a wrapper that wipes the `work/` output dir and times each
invocation with `date +%s%N` before/after (see "Reproducing this" below for
the full script):

| | min | median | p90 |
| --- | --- | --- | --- |
| go-ansible (`/tmp/go-ansible-playbook -i inventory.ini playbook.yml`) | 23ms | 25ms | 26ms |
| real ansible (`ansible-venv/bin/ansible-playbook -i inventory.ini playbook.yml`) | 1683ms | 1694ms | 1711ms |

**~68× faster, median to median**, on this 8-task playbook, on this machine.

### Startup-only cost

`--version` is now wired on `ansible-playbook` (and all three other CLI
binaries), so this section's original workaround is no longer required —
kept anyway to isolate process-startup overhead from playbook parsing/setup
cost, which a `--version` invocation wouldn't exercise. Both tools ran a
single-task no-op playbook (`debug: msg: noop`) instead:

| | min | median | p90 |
| --- | --- | --- | --- |
| go-ansible | 7ms | 7ms | 7ms |
| real ansible | 297ms | 300ms | 308ms |

That ~300ms is CPython interpreter start + importing `ansible.cli.playbook`
and its dependency chain (jinja2, cryptography, yaml, ...) — a fixed cost
real Ansible always pays before doing any work, every single invocation. Go
has no equivalent: the binary is already machine code, there's no
interpreter to boot or module graph to import. This is the single biggest
contributor to the 68× gap above — most of the 1.7s full-playbook run is
still Python/module-import overhead, not the 8 tasks themselves.

## 3. `FROM scratch` container proof

The differentiator: a `CGO_ENABLED=0` Go binary has zero runtime
dependencies, so it can run in an empty container. Real Ansible fundamentally
cannot — it needs a libc, a CPython interpreter, and several C-extension
pip packages (`cryptography` for Vault, `cffi`/`pycparser` underneath it)
just to start.

### go-ansible in `FROM scratch`

```dockerfile
FROM scratch
COPY ansible-playbook /ansible-playbook
COPY scratch-playbook.yml /scratch-playbook.yml
COPY inventory.ini /inventory.ini
ENTRYPOINT ["/ansible-playbook"]
CMD ["-i", "/inventory.ini", "/scratch-playbook.yml"]
```

The binary is cross-compiled on the host (`CGO_ENABLED=0 GOOS=linux
GOARCH=arm64 go build`) — no build stage needed, there's nothing scratch
needs to link against. Built and run natively (arm64 host, arm64 image, no
QEMU emulation):

```
$ docker build --platform linux/arm64 -t go-ansible-scratch:bench-arm64 .
$ docker images go-ansible-scratch:bench-arm64
IMAGE                            DISK USAGE   CONTENT SIZE
go-ansible-scratch:bench-arm64   21.3MB       7.16MB

$ docker run --rm go-ansible-scratch:bench-arm64

TASK [debug a static message]
ok: [localhost]

TASK [set a fact]
ok: [localhost]

TASK [debug the fact]
ok: [localhost]

PLAY RECAP
localhost                : ok=3    changed=0    failed=0    skipped=0
exit=0
```

It works: **21.3MB image, real output, exit 0.** (A cross-platform
amd64-in-scratch build under QEMU emulation also ran clean at 7.79MB image
size — smaller because it wasn't cross-compiled for the host's own arch, so
compare the arm64/arm64 native number above as the fair one.)

**Limitation found, reported honestly:** the playbook above deliberately
avoids the `command`/`shell` modules. When the benchmark playbook's
`command: echo "..."` task is pointed at the same `FROM scratch` image, it
fails:

```
TASK [run a shell command]
failed: [localhost] => transport: local exec: fork/exec /bin/sh: no such file or directory
exit=2
```

`scratch` has no `/bin/sh`, and go-ansible's `ansible_connection: local`
executes `command`/`shell` tasks by forking `/bin/sh`. `debug`, `set_fact`,
and any module whose logic runs entirely on the controller (no shell fork)
work fine in `scratch`; `command`, `shell`, and anything that shells out to
the target do not, unless the image provides a shell (e.g. `FROM
busybox:musl` instead of `scratch` — not tested here, but architecturally
that's the fix). This is a real constraint of the `scratch` proof, not a
go-ansible bug: real Ansible's `command`/`shell` modules have the identical
shell dependency, just on whatever's managing the target instead.

### The best real Ansible can do

`scratch` is not on the table for real Ansible at all — it needs a libc and
a CPython interpreter before `import ansible` even resolves. The smallest
realistic base is a slim Python distro:

```dockerfile
FROM python:3.13-slim
RUN pip install --no-cache-dir ansible-core
ENTRYPOINT ["ansible-playbook"]
```

```
$ docker images python:3.13-slim
python:3.13-slim   DISK USAGE 215MB   CONTENT SIZE 48.8MB

$ docker images real-ansible-slim:bench
real-ansible-slim:bench   DISK USAGE 284MB   CONTENT SIZE 62.4MB

$ docker run --rm --platform linux/arm64 \
    -v $PWD/inventory.ini:/inventory.ini:ro -v $PWD/noop.yml:/noop.yml:ro \
    real-ansible-slim:bench -i /inventory.ini /noop.yml
ok: [localhost] => {"msg": "noop"}
exit=0
```

It runs, and correctly — but at **284MB, roughly 13× the go-ansible scratch
image (21.3MB) and larger than the entire go-ansible venv+interpreter
footprint measured in §1.** `python:3.13-slim` alone (before `ansible-core`
is even installed) is already 215MB — 10× go-ansible's whole container.
There's no lower rung available: Alpine+`pip install` would need to compile
`cryptography` from source (no manylinux wheels for musl libc) or accept a
much larger image pulling in a Rust toolchain, so `-slim` glibc-based is
genuinely close to the floor for a real, working Ansible controller image.

## Gaps this benchmarking run surfaced (since fixed)

Not asked for, but found along the way when this benchmark was first run —
reported per the project's own stated policy of only claiming parity where
it's real. Engine work landed after this benchmark, so all three are now
fixed; kept here as a record of what this run actually found at the time,
rather than quietly deleted:

- ~~No `--version` flag on `ansible-playbook`~~ — fixed. All four CLI
  binaries (`ansible`, `ansible-playbook`, `ansible-vault`, `ansible-galaxy`)
  now support `--version`, and `ansible-playbook` also wires `--tags`/`-t`
  and `--skip-tags` (confirmed by reading `cli/cmd/*/main.go` directly).
- ~~`playbook_dir` is not populated as a magic variable~~ — fixed.
  `playbook/engine.go` now sets it (`vc.SetVar(vars.Inventory,
  "playbook_dir", e.BaseDir)`) after host_vars, alongside
  `inventory_hostname`, so a same-named host_var can't shadow it.
- ~~`inventory_hostname` is not populated in the template rendering
  context~~ — fixed, same commit as above (`vc.SetVar(vars.Inventory,
  "inventory_hostname", h.Name)`).

The measured numbers above (binary size, latency, `FROM scratch`) predate
this fix and were not re-run for it — they're independent of these three
gaps and remain valid.

## Reproducing this

Everything above came from real commands, not estimates. In order:

```bash
export GOWORK=off

# Binary sizes
cd cli
go build -o /tmp/go-ansible-playbook ./cmd/ansible-playbook
GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build -o /tmp/go-ansible-playbook-linux-amd64 ./cmd/ansible-playbook
du -h /tmp/go-ansible-playbook /tmp/go-ansible-playbook-linux-amd64
file /tmp/go-ansible-playbook-linux-amd64

# Reference ansible-core install
python3 -m venv /tmp/ansible-venv
/tmp/ansible-venv/bin/pip install ansible-core
du -sh /tmp/ansible-venv
du -sh /tmp/ansible-venv/lib/python*/site-packages/ansible*
du -sh "$(python3 -c "import os;print(os.path.realpath('/tmp/ansible-venv/bin/python3'))" | xargs dirname | xargs dirname | xargs dirname)"

# Latency (N=10, first run discarded, playbook.yml/inventory.ini as in §2)
for i in $(seq 1 10); do
  rm -rf work; mkdir -p work
  start=$(date +%s%N); TOOL -i inventory.ini playbook.yml </dev/null >/dev/null 2>&1; end=$(date +%s%N)
  echo $(( (end-start)/1000000 ))ms
done

# FROM scratch
GOOS=linux GOARCH=arm64 CGO_ENABLED=0 go build -o docker/ansible-playbook ./cmd/ansible-playbook
docker build --platform linux/arm64 -t go-ansible-scratch:bench docker/
docker images go-ansible-scratch:bench
docker run --rm go-ansible-scratch:bench

# Real-Ansible contrast image
docker build --platform linux/arm64 -t real-ansible-slim:bench docker/realansible/
docker images real-ansible-slim:bench python:3.13-slim
```

Substitute `TOOL` with `/tmp/go-ansible-playbook` or
`/tmp/ansible-venv/bin/ansible-playbook` (redirect stdin from `/dev/null` —
real ansible-core errors on non-blocking stdio otherwise, see above).
