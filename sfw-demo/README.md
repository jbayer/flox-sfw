# sfw-demo

A Flox consumption environment that shows how [Socket Firewall Free (sfw)](https://github.com/SocketDev/sfw-free) blocks the installation of a known-malicious npm package.

## Warning

This demo intentionally attempts to install `lodahs`, a real typosquat of `lodash` that has been flagged as malware. **Do not run on your workstation.** Run only inside an isolated, throwaway environment such as a devcontainer, VM, or ephemeral cloud sandbox.

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

Supported systems: `aarch64-darwin`, `aarch64-linux`.

## Demo

From a devcontainer:

```bash
cd sfw-demo
flox activate
mkdir -p /tmp/sfw-demo-project && cd /tmp/sfw-demo-project
npm init -y

# Baseline (unprotected) - DO NOT RUN OUTSIDE A SANDBOX
# npm install lodahs

# With Socket Firewall: the typosquat should be blocked before install
sfw npm install lodahs
```

Expected: `sfw` wraps the package manager's network traffic, recognizes `lodahs` as a known-malicious typosquat of `lodash`, and refuses to let npm fetch it. The unprotected `npm install lodahs` would proceed (and infect the sandbox), which is why you only run it where a wipe-and-redeploy is cheap.

## Contrast - a legitimate install still works

```bash
sfw npm install lodash
```

That should download `lodash` normally, demonstrating that sfw isn't blocking *all* installs — just ones that match Socket's risk signals.
