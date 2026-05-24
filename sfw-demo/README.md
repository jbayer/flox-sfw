# sfw-demo

A Flox consumption environment that shows how [Socket Firewall Free (sfw)](https://github.com/SocketDev/sfw-free) blocks the installation of a known-malicious npm package.

## Warning

This demo intentionally attempts to install real malicious packages from public registries:

- npm: **`lodahs`** — typosquat of `lodash`
- pypi: **`fabrice`** — typosquat of `fabric`

**Do not run on your workstation.** Run only inside an isolated, throwaway environment such as a devcontainer, VM, or ephemeral cloud sandbox.

A ready-to-use devcontainer is included at `sfw-demo/.devcontainer/devcontainer.json` — open this directory in VS Code (or any devcontainer-aware tool) and choose "Reopen in Container" to get an isolated Linux environment with Flox preinstalled.

## FloxHub authentication

`jbayer/sfw` is hosted on FloxHub, so the **first** time you activate this environment Flox will fetch the package from FloxHub and you may need to be logged in:

```bash
flox auth login
```

After the first successful activation, the package is cached in the local `/nix/store` and **subsequent activations work offline** — no FloxHub auth required.

You only need to log in again when:

- Checking for or pulling a newer version (`flox upgrade`, `flox install jbayer/sfw` against a newer release)
- Activating on a different machine that hasn't fetched the package before

## What's in the environment

- `jbayer/sfw` — the Flox-packaged Socket Firewall (from this repo's published package)
- `nodejs` — node + npm
- `python3` + `pip` — for the PyPI demo

Supported systems: `aarch64-darwin`, `aarch64-linux`.

## Demo - npm (`lodahs`)

From a devcontainer:

```bash
cd sfw-demo
flox activate
mkdir -p /tmp/sfw-demo-npm && cd /tmp/sfw-demo-npm
npm init -y

# Baseline (unprotected) - DO NOT RUN OUTSIDE A SANDBOX
# npm install lodahs

# With Socket Firewall: the typosquat should be blocked before install
sfw npm install lodahs
```

Expected: `sfw` wraps npm's network traffic, recognizes `lodahs` as a known-malicious typosquat of `lodash`, and refuses to let npm fetch it.

## Demo - PyPI (`fabrice`)

```bash
cd sfw-demo
flox activate
python3 -m venv /tmp/sfw-demo-venv
source /tmp/sfw-demo-venv/bin/activate

# Baseline (unprotected) - DO NOT RUN OUTSIDE A SANDBOX
# pip install fabrice

# With Socket Firewall: should be blocked before any code from
# fabrice is downloaded or executed
sfw pip install fabrice
```

Expected: `sfw` intercepts pip's PyPI traffic and blocks `fabrice` (a typosquat of `fabric` that has been observed exfiltrating AWS credentials).

## Contrast - legitimate installs still work

```bash
sfw npm install lodash
sfw pip install fabric
```

Both should download normally, demonstrating that sfw isn't blocking *all* installs — just ones that match Socket's risk signals.
