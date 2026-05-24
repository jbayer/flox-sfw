# sfw-demo

A Flox consumption environment that shows how [Socket Firewall Free (sfw)](https://github.com/SocketDev/sfw-free) blocks the installation of a known-malicious npm package.

## Warning

This demo intentionally attempts to install real malicious packages from public registries:

- npm: **`lodahs`** — typosquat of `lodash`
- pypi: **`fabrice`** — typosquat of `fabric`
- crates.io: **`rustdecimal`** — typosquat of `rust_decimal`

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
- `pip` — pulls `python3` in as a runtime dependency, used for the PyPI demo
- `cargo` — used for the crates.io demo (no rustc needed; the registry fetch happens before any build step)

Supported systems: `aarch64-darwin`, `aarch64-linux`.

## Activate the environment

From a devcontainer:

```bash
cd sfw-demo
flox activate
```

## Demo - npm (`lodahs`)

Copy-paste the whole block:

```bash
mkdir -p /tmp/sfw-demo-npm && cd /tmp/sfw-demo-npm
npm init -y
sfw npm install lodahs
```

Expected: `sfw` wraps npm's network traffic, recognizes `lodahs` as a known-malicious typosquat of `lodash`, and refuses to let npm fetch it.

## Demo - PyPI (`fabrice`)

The nixpkgs build of pip enforces `require-virtualenv`, so the venv steps are not optional — pip will abort before any network call (and sfw will never get to filter) if you skip them. Copy-paste the whole block:

```bash
python3 -m venv /tmp/sfw-demo-venv
source /tmp/sfw-demo-venv/bin/activate
sfw pip install fabrice
```

Expected: `sfw` intercepts pip's PyPI traffic and blocks `fabrice` (a typosquat of `fabric` that has been observed exfiltrating AWS credentials).

**Heads up:** PyPI removes flagged malicious packages quickly, often within hours of disclosure. By the time you try this, `fabrice` may already be unreachable on PyPI — in which case pip will fail with a normal "no matching distribution" error rather than an sfw block. That's fine for the demo: the point is to see `sfw` wrapping pip and announcing itself as the policy layer in front of your installs. If you want to see a live block, swap `fabrice` for a currently-flagged malicious package from [Socket's advisories](https://socket.dev/) at the time you're running this.

## Demo - cargo (`rustdecimal`)

Copy-paste the whole block:

```bash
mkdir -p /tmp/sfw-demo-cargo && cd /tmp/sfw-demo-cargo
sfw cargo install rustdecimal
```

Expected: `sfw` intercepts cargo's crates.io traffic and blocks `rustdecimal` (a 2022 typosquat of `rust_decimal` that injected a payload during build).

**Heads up:** crates.io *yanks* flagged crates quickly — `rustdecimal` was yanked the day it was disclosed. A yanked crate may still resolve in some flows but typically fails with a "not found" or "yanked" error before sfw needs to step in. As with the PyPI demo, the goal is to see sfw wrap cargo and announce itself as the policy layer. Swap in a currently-flagged crate from [Socket's advisories](https://socket.dev/) for a live block.

## Contrast - legitimate installs still work

```bash
sfw npm install lodash
sfw pip install fabric
sfw cargo install ripgrep
```

All three should download normally, demonstrating that sfw isn't blocking *all* installs — just ones that match Socket's risk signals.

## Baseline (unprotected) — DO NOT RUN OUTSIDE A SANDBOX

Just so you can compare, the unprotected versions are `npm install lodahs`, `pip install fabrice`, and `cargo install rustdecimal`. These will (if still available upstream) successfully fetch and in some cases execute the malicious packages. Only run them in a throwaway environment you intend to discard.
