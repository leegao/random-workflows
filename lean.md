# Lean 4 + Mathlib Airgapped Bundle

A self-contained Debian 12 (bookworm) bundle for running Lean 4, Mathlib, the Lean REPL, and lean-interact in an offline / airgapped gVisor sandbox.

## What's Inside

| Component | Location in bundle | Purpose |
|---|---|---|
| Debian `.deb` packages | `debs/` | Offline system dependencies (git, curl, build-essential, libgmp-dev, python3-venv, etc.) |
| Python wheels | `wheels/` | Offline wheels for `lean-interact`, `leanclient`, `pip`, `setuptools`, `wheel` |
| Elan + Lean toolchain | `elan/` | Pinned Lean 4 toolchain managed by Elan |
| Mathlib `.olean` cache | `mathlib-cache/` | Precompiled Mathlib artifacts (`XDG_CACHE_HOME`) |
| Lake workspace | `workspace/` | Prebuilt workspace with Mathlib + REPL sources and `.olean`s |
| Installer | `install.sh` | Offline installation script |

A smaller helper tarball (`lean-python-repl-bookworm-*.tar.xz`) is also produced containing just `wheels/` and the prebuilt `repl` package.

## Prerequisites (Target Machine)

- Debian 12 (bookworm) or compatible
- Root access (for `.deb` installation)
- Python 3.11+
- ~5 GB free disk space

## Installation

1. **Transfer the bundle** to the airgapped target machine:

   ```sh
   scp lean-mathlib-bookworm-v4.31.0.tar.xz user@target:/tmp/
   ```

2. **Extract the bundle:**

   ```sh
   cd /tmp
   tar -xf lean-mathlib-bookworm-v4.31.0.tar.xz
   cd lean-mathlib-bookworm-v4.31.0
   ```

3. **Run the installer** (as root for `.deb` installation):

   ```sh
   sudo ./install.sh /opt/lean
   ```

   The optional argument `/opt/lean` is the target prefix (default: `/opt/lean`).

   The installer will:
   - Install offline `.deb` packages via `dpkg -i`
   - Copy `elan/`, `mathlib-cache/`, and `workspace/` to the target prefix
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
| `XDG_CACHE_HOME` | `/opt/lean/mathlib-cache` |
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
- **Mathlib import errors** — Ensure `XDG_CACHE_HOME` points to `/opt/lean/mathlib-cache` (set by `env.sh`).
- **Python import errors** — Activate the venv: `source /opt/lean/venv/bin/activate`.