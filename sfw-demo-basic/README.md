# sfw-demo-basic

The minimal Flox consumption environment for [Socket Firewall Free (sfw)](https://github.com/SocketDev/sfw-free): install the package, prepend its shipped PATH shims, done. Bring your own `npm`, `yarn`, `pnpm`, `pip`, `uv`, or `cargo` from the system or a layered Flox env, and the shims route any of them through `sfw` automatically.

For an end-to-end demo with `nodejs`, `pip`, `cargo` pre-installed and writable install targets pre-configured, see the sibling [`sfw-demo-full/`](../sfw-demo-full).

## Warning

If you do try the example commands below with a real malicious package name, only run inside a devcontainer, VM, or other throwaway sandbox.

A ready-to-use devcontainer is included at `sfw-demo-basic/.devcontainer/devcontainer.json`.

## FloxHub authentication

`jbayer/sfw` is hosted on FloxHub, so the **first** time you activate this environment Flox will fetch the package from FloxHub and you may need to be logged in:

```bash
flox auth login
```

Subsequent activations work offline. See the top-level [README](../README.md) for details.

## What's in the environment

- `jbayer/sfw` — the Flox-packaged Socket Firewall, which ships PATH shims at `$FLOX_ENV/libexec/sfw-shims/{npm,yarn,pnpm,pip,uv,cargo}`
- `gum` — for the activation panel

That's it — no package managers, no scratch directories, no install-target env vars. If you want `npm install <pkg>` (or any of the other shimmed tools) to actually do anything, install the corresponding ecosystem separately (system-wide, in a layered env, or by editing the manifest).

Supported systems: `aarch64-darwin`, `aarch64-linux`, `x86_64-linux`.

## How the shim wiring works

The env prepends `$FLOX_ENV/libexec/sfw-shims/` to `PATH` in two places:

- `[hook] on-activate` — covers non-interactive invocations (`flox activate -- cmd`, scripts, CI, agents).
- `[profile.common]` — covers interactive shells, where Flox re-prepends `$FLOX_ENV/bin` after the hook and would otherwise let plain `npm`/`yarn`/`pnpm`/`pip`/`uv`/`cargo` bypass the shim.

When a shim runs it execs `sfw <real-binary> "$@"`. A `_SFW_WRAPPING` env-var sentinel breaks the recursion when sfw itself execs the real command back through the shim.

## Activate and use

From a devcontainer (or any host with the relevant tools available on PATH):

```bash
cd sfw-demo-basic
flox activate

# If `npm` (or any of yarn/pnpm/pip/uv/cargo) is on PATH from the system
# or a layered env, this routes through sfw automatically:
npm install lodahs
yarn add lodahs
pnpm add lodahs
pip install fabrice
uv pip install fabrice
cargo install rustdecimal
```

If the package manager is *not* on PATH the shim will print `sfw-shim: real <cmd> not found on PATH` and exit 127 — install the tool or use `sfw-demo-full` which pre-installs them.
