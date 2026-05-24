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

## A note on the PyPI and cargo demos

PyPI and crates.io take flagged malicious packages down quickly — often within hours of disclosure. By the time you run these demos, `fabrice` and `rustdecimal` may already be unreachable, in which case the package manager fails with a normal "not found" / "yanked" error and `sfw` never gets to issue a block decision. **For those two demos, treat success as "sfw is correctly wrapping the pip / cargo call and announcing itself as the policy layer."** A live block — where sfw refuses traffic for a still-published malicious package — is best demonstrated with the npm `lodahs` example (npm tends to leave flagged packages reachable longer), or by swapping in any currently-flagged package from [Socket's advisories](https://socket.dev/) at the moment you run the demo.

## Activate the environment

From a devcontainer:

```bash
cd sfw-demo
flox activate
```

On activation the env installs PATH shims for `npm`, `pip`, and `cargo` in `$FLOX_ENV_CACHE/sfw-shims/` and prepends them to `PATH`. The shims `exec sfw <cmd> "$@"`, so every call to those tools — interactive prompts, scripts, child processes, `flox activate -- npm ...`, agent-driven invocations — routes through Socket Firewall automatically. A `_SFW_WRAPPING` sentinel breaks the recursion when sfw itself execs the real package manager.

If you ever need the *unshimmed* binary (e.g. for debugging), call it through its absolute path or temporarily remove the shim dir from `PATH`.

## Demo - npm (`lodahs`)

Copy-paste the whole block:

```bash
mkdir -p /tmp/sfw-demo-npm && cd /tmp/sfw-demo-npm
npm init -y
npm install lodahs
```

Expected: `sfw` wraps npm's network traffic, recognizes `lodahs` as a known-malicious typosquat of `lodash`, and refuses to let npm fetch it.

## Demo - PyPI (`fabrice`)

The nixpkgs build of pip enforces `require-virtualenv`, so the venv steps are not optional — pip will abort before any network call (and sfw will never get to filter) if you skip them. Copy-paste the whole block:

```bash
python3 -m venv /tmp/sfw-demo-venv
source /tmp/sfw-demo-venv/bin/activate
pip install fabrice
```

Expected: `sfw` intercepts pip's PyPI traffic and announces itself. If `fabrice` is still published, sfw blocks it; if PyPI has already taken it down, pip exits with a "no matching distribution" error — either way, seeing the `sfw` banner in front of pip is the demo's point (see [the PyPI/cargo note above](#a-note-on-the-pypi-and-cargo-demos)). `fabrice` is a typosquat of `fabric` that has been observed exfiltrating AWS credentials.

## Demo - cargo (`rustdecimal`)

Copy-paste the whole block:

```bash
mkdir -p /tmp/sfw-demo-cargo && cd /tmp/sfw-demo-cargo
cargo install rustdecimal
```

Expected: `sfw` intercepts cargo's crates.io traffic and announces itself. `rustdecimal` was yanked from crates.io the day it was disclosed in 2022 (it injected a payload during build), so the most likely outcome today is a "could not find rustdecimal" error from cargo — again, seeing the `sfw` banner in front of cargo is what proves the integration is working (see [the PyPI/cargo note above](#a-note-on-the-pypi-and-cargo-demos)).

## Contrast - legitimate installs still work

```bash
npm install lodash
pip install fabric
cargo install ripgrep
```

All three should download normally, demonstrating that sfw isn't blocking *all* installs — just ones that match Socket's risk signals.

## Baseline (unprotected) — DO NOT RUN OUTSIDE A SANDBOX

The shims are scoped to this Flox environment. To compare against the unprotected behaviour, exit `flox activate` (or call the binaries by absolute path) and re-run `npm install lodahs` / `pip install fabrice` / `cargo install rustdecimal`. These will (if still available upstream) successfully fetch and in some cases execute the malicious packages. Only run them in a throwaway environment you intend to discard.
