# flox-sfw

Flox packaging of [Socket Firewall Free (sfw)](https://github.com/SocketDev/sfw-free) — a wrapper that filters network traffic for package managers to block installation of malicious packages.

This repo repackages SocketDev's prebuilt binaries as a Flox manifest build, publishable to FloxHub for `aarch64-darwin`, `aarch64-linux`, and `x86_64-linux`.

## Install

```bash
flox install jbayer/sfw
```

The first time you install or activate, Flox fetches the package from FloxHub — you may need to authenticate first:

```bash
flox auth login
```

Once the package is in the local `/nix/store`, subsequent activations work offline. You only need to log in again to check for or pull newer versions.

> [!IMPORTANT]
> `jbayer/sfw` is the **catalog path** for one specific FloxHub account (`jbayer`). FloxHub catalogs are per-account, and a published package is only visible to the publishing account and anyone that account has explicitly granted access. **If you are not `jbayer` (and haven't been granted access to that catalog), `flox install jbayer/sfw` will fail.**
>
> `flox auth login` authenticates *you* against FloxHub — it does not grant access to other users' catalogs. To use this packaging yourself, you'll want to [publish under your own FloxHub account](#publish-under-your-own-floxhub-account) (clone, swap a few `jbayer` references for your own org, push a tag).

Then:

```bash
sfw npm install
sfw pip install requests
```

## Transparent shims (opt-in)

The package also ships per-ecosystem PATH shims at `$FLOX_ENV/libexec/sfw-shims/` for the package managers Socket Firewall supports:

- `npm`, `yarn`, `pnpm`
- `pip`, `uv`
- `cargo`

Each shim resolves the real binary on `PATH` and execs `sfw <real-bin> "$@"`, so calls are routed through Socket Firewall transparently — including in scripts, child processes, `flox activate -- <cmd>`, CI, and agent-driven invocations where a shell alias wouldn't help. A `_SFW_WRAPPING` env-var sentinel breaks the recursion when sfw itself execs the real command.

Opt in by prepending the shim dir to `PATH` in your `[hook] on-activate`:

```toml
[install]
sfw.pkg-path = "jbayer/sfw"
nodejs.pkg-path = "nodejs"

[hook]
on-activate = '''
export PATH="$FLOX_ENV/libexec/sfw-shims:$PATH"
'''
```

After that, plain `npm install <pkg>` (no `sfw` prefix) routes through Socket Firewall.

> [!TIP]
> The PATH prepend needs to live in **both** `[hook] on-activate` (so it applies in `flox activate -- cmd`, scripts, and CI) and `[profile.common]` (so it survives Flox's interactive-shell PATH setup, which can re-prepend `$FLOX_ENV/bin` after the hook). The two example demos below set it up both ways.

### Sourceable shell helpers

For bash and zsh users, the package also ships activation helpers at `$FLOX_ENV/libexec/sfw-shims/activate.{bash,zsh}` that handle both the initial `PATH` prepend AND install a prompt hook (`PROMPT_COMMAND` / `precmd`) to **re-prepend the shim dir on every prompt**. That last part matters when you `source` a script *after* `flox activate` that mutates `PATH` — most commonly a Python venv's `bin/activate`. Without the prompt hook, the venv's `bin/pip` lands in front of the sfw shim and `pip` calls inside the venv bypass Socket Firewall.

Recommended consumer profile:

```toml
[hook]
on-activate = '''
export PATH="$FLOX_ENV/libexec/sfw-shims:$PATH"   # covers `flox activate -- cmd`
'''

[profile]
common = '''
export PATH="$FLOX_ENV/libexec/sfw-shims:$PATH"   # fallback for shells without specific profiles (fish, sh)
'''
bash = '''
source "$FLOX_ENV/libexec/sfw-shims/activate.bash"
'''
zsh = '''
source "$FLOX_ENV/libexec/sfw-shims/activate.zsh"
'''
```

Both demos in this repo use this exact pattern.

## Example consumption environments

Two demos in this repo show what installing `jbayer/sfw` looks like in practice:

- [`sfw-demo-basic/`](./sfw-demo-basic) — the minimum: install `sfw`, wire the shipped shims onto `PATH`. Bring your own `npm`/`pip`/`cargo` from system or a layered env. Best starting point if you just want to see the moving parts.
- [`sfw-demo-full/`](./sfw-demo-full) — pre-installs `nodejs`, `pip`, and `cargo`; sets per-env install targets (`NPM_CONFIG_PREFIX`, `CARGO_HOME`, `PIP_TARGET`) so installs work without sudo or `$HOME` pollution; walks through three blocked-package demos (`lodahs`, `fabrice`, `rustdecimal`).

Both ship a `.devcontainer/devcontainer.json` so you can reopen-in-container for an isolated test environment.

## Build locally

```bash
flox activate
flox build sfw
./result-sfw/bin/sfw --version
```

## Bumping the upstream version

Edit `.flox/env/manifest.toml` and update:

- `[build.sfw] version`
- `[vars] SFW_VERSION`
- `[vars] SFW_SHA256_DARWIN_ARM64`
- `[vars] SFW_SHA256_LINUX_ARM64`
- `[vars] SFW_SHA256_LINUX_X86_64`

The sha256 values can be pulled from `gh release view --repo SocketDev/sfw-free vX.Y.Z --json assets`.

The `update-check` workflow runs daily and opens a PR automatically when SocketDev publishes a new release.

## Publishing (this repo)

The `publish` workflow runs on release tags (`v*`) and publishes for all three supported systems (`aarch64-darwin`, `aarch64-linux`, `x86_64-linux`) to FloxHub under the `jbayer` org.

Requires the `FLOXHUB_TOKEN` repository secret.

## Publish under your own FloxHub account

FloxHub catalogs are per-account; you can't publish to `jbayer/sfw` unless you *are* `jbayer`. If you want this packaging available under your own catalog (e.g. `acme/sfw`), the path is short — clone, swap a handful of `jbayer` references, push a tag.

1. **Create a FloxHub account** if you don't have one, and run `flox auth login` locally.
2. **Fork or clone this repo** into your own GitHub account/org.
3. **Replace `jbayer` with your FloxHub org slug** in these files:
   - `.github/workflows/publish.yml` → the `flox publish --org jbayer sfw` line
   - `sfw-demo-basic/.flox/env/manifest.toml` → `sfw.pkg-path = "jbayer/sfw"`
   - `sfw-demo-full/.flox/env/manifest.toml` → `sfw.pkg-path = "jbayer/sfw"`
   - `README.md` → the install example (optional, for your own docs)
4. **Add a `FLOXHUB_TOKEN` repo secret** in your fork. Easiest way:
   ```bash
   flox auth token | gh secret set FLOXHUB_TOKEN -R <your-gh-org>/flox-sfw
   ```
   The token is encrypted at rest by GitHub and never displayed back.
5. **Tag and push** to trigger the publish workflow:
   ```bash
   git tag -a v1.10.0 -m "Publish under <your-org>"
   git push origin v1.10.0
   ```
6. Once the workflow succeeds, anyone using your account (or with access to your catalog) can `flox install <your-org>/sfw`.

The publish workflow already accepts re-publishing the same `vX.Y.Z` with a new build hash, so you can iterate on the manifest and re-tag the same version while testing.
