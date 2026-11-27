# Lean 4 + Mathlib Airgapped Bundle

A self-contained Debian 12 (bookworm) bundle for running Lean 4, Mathlib, the Lean REPL, and lean-interact in an offline / airgapped gVisor sandbox.

## What's Inside

The bundle is split into two tarballs (each under GitHub's 2 GB asset limit):

### Part 1 — `lean-mathlib-bookworm-*-part1.tar.xz`

| Component | Location | Purpose |
|---|---|---|
| Debian `.deb` packages | `debs/` | Offline system dependencies (git, curl, build-essential, libgmp-dev, python3-venv, etc.) |
| Elan + Lean toolchain | `elan/` | Pinned Lean 4 toolchain managed by Elan (docs/tests/src stripped) |
| Installer | `install.sh` | Offline installation script |
| Documentation | `lean.md` | This file |

### Part 2 — `lean-mathlib-bookworm-*-part2.tar.xz`

| Component | Location | Purpose |
|---|---|---|
| Lake workspace | `workspace/` | Prebuilt workspace with Mathlib + REPL sources and `.olean`s |
| Python wheels | `wheels/` | Offline wheels for `lean-interact`, `leanclient`, `pip`, `setuptools`, `wheel` |

## Prerequisites (Target Machine)

- Debian 12 (bookworm) or compatible
- Root access (for `.deb` installation)
- Python 3.11+
- ~5 GB free disk space

## Installation

1. **Transfer both parts** to the airgapped target machine:

   ```sh
   scp lean-mathlib-bookworm-*-part1.tar.xz lean-mathlib-bookworm-*-part2.tar.xz user@target:/tmp/
   ```

2. **Extract both parts into the same directory:**

   ```sh
   mkdir -p /tmp/lean-bundle
   tar -xf /tmp/lean-mathlib-bookworm-*-part1.tar.xz -C /tmp/lean-bundle
   tar -xf /tmp/lean-mathlib-bookworm-*-part2.tar.xz -C /tmp/lean-bundle
   ```

3. **Run the installer** (as root for `.deb` installation):

   ```sh
   cd /tmp/lean-bundle
   sudo ./install.sh /opt/lean
   ```

   The optional argument `/opt/lean` is the target prefix (default: `/opt/lean`).

   The installer will:
   - Install offline `.deb` packages via `dpkg -i`
   - Copy `elan/` and `workspace/` to the target prefix
   - Create a Python venv and install `lean-interact` + `leanclient` offline
   - Generate an `env.sh` activation script

## Usage After Install

### Activate the environment

```sh
source /opt/lean/env.sh
```

This sets:

| Variable | Value |
|---|---|
| `ELAN_HOME` | `/opt/lean/elan` |
| `LEAN_WORKSPACE` | `/opt/lean/workspace` |
| `PATH` | Includes `/opt/lean/elan/bin` and `/opt/lean/venv/bin` |

### Verify the installation

```sh
lean --version
lake --version
python3 -c "import lean_interact; print('lean-interact OK')"
```

### Run Lean files

```sh
cd /opt/lean/workspace
lake build AirgappedMathlib
```

Create your own Lean file and check it:

```sh
cat > /tmp/test.lean <<'EOF'
import Mathlib.Data.Nat.Prime.Basic
theorem test (n : Nat) : n + 0 = n := by simp
EOF
lean /tmp/test.lean
```

### Use the Lean REPL

The prebuilt REPL binary is at `/opt/lean/workspace/.lake/packages/repl`:

```sh
cd /opt/lean/workspace
lake build repl
echo '{"cmd": "import Mathlib"}' | lake exe repl
```

### Use lean-interact (Python)

```python
from lean_interact import LeanServer

server = LeanServer(lean_workspace="/opt/lean/workspace")
result = server.run("theorem test (n : Nat) : n + 0 = n := by simp")
print(result)
```

### Run in a gVisor sandbox

The `env.sh` script includes a `ulimit -s unlimited` safeguard for gVisor environments. If your sandbox restricts this, the script continues silently.

## Troubleshooting

- **`lean: command not found`** — Run `source /opt/lean/env.sh` first.
- **`dpkg` errors during install** — The installer uses `|| true` to continue past non-fatal `.deb` issues; check `dpkg -i` output for real failures.
- **Mathlib import errors** — The `.olean` cache is prebuilt in `workspace/.lake/packages/mathlib/.lake/build`; ensure you're running from `/opt/lean/workspace`.
- **Python import errors** — Activate the venv: `source /opt/lean/venv/bin/activate`.