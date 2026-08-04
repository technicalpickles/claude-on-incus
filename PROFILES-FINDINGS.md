# COI Profiles — Findings

What profiles are, how they relate to Incus, what they mean semantically, and where the
documentation has drifted from the code.

Scope: `master` @ `da67215` (2026-08-01). Every claim here is backed by a citation or a
probe in [`PROFILES-EVIDENCE.md`](PROFILES-EVIDENCE.md); section refs like (§E) point
there. The investigation first ran against `afb96e8` and was re-run in full after master
fast-forwarded 95 commits — §7 records what that changed.

---

## 1. The one-line answer

A COI profile is a **named, versionable bundle of session policy** — a directory holding a
`config.toml` (plus optional build script and agent context file) that overrides the
resolved configuration for one invocation, selected with `--profile <name>`.

It is **not an Incus profile**, despite the shared word. The two overlap substantially in
*what they can express* — CPU/memory/disk limits, bind mounts, proxy devices, hardening
keys — and share no mechanism whatsoever: COI writes all of it straight to the instance
and never creates, names or attaches an Incus profile object (§3.2, §G).

---

## 2. Functionally

### 2.1 A directory, not a config table

```
~/.coi/profiles/rust-dev/          ← or ./.coi/profiles/rust-dev/
├── config.toml                    ← the profile (required; absent ⇒ dir skipped)
├── build.sh                       ← optional, referenced by [container.build] script
└── CONTEXT.md                     ← optional, referenced by context =
```

The **directory name is the profile name**. Paths inside `config.toml` resolve relative to
the profile directory, which is what makes the bundle relocatable.

Scanned from **three** parent directories (§A1): `~/.coi`, `$CWD/.coi`, and
`dirname($COI_CONFIG)`. Results land in one flat namespace, and a name defined in two
locations is a **hard startup error**, not a precedence rule (§A2) — an explicit design
choice that it must always be obvious which file is in play.

### 2.2 Two built-ins exist without any directory

- **`default`** — synthesized *after* config files are merged, so it is a live clone of the
  resolved configuration: embedded defaults + `~/.coi/config.toml` + `./.coi/config.toml`
  (§A3, §H1). It is the "vanilla" escape hatch and the conventional inheritance root.
- **`hardened`** — a fixed baseline, not a clone: restricted network with private-network
  and metadata-endpoint blocks, 16 secret-mask globs, host immutability, ephemeral
  container, no SSH-agent forwarding, monitoring + nft on (§H2). It sets *only* those
  fields and lets everything else fall through.

Both carry `Source = "(built-in)"`, both are shadowable by a same-named disk profile, and
both are refused by `profile edit` / `profile delete` (§H3).

`coi profile create default` doesn't create a profile at all — it scaffolds the *main
config*, because that's the file backing the built-in `default` (§H4).

### 2.3 Four ways a profile gets selected

In precedence order, lowest first (§H5, §G5):

1. `[defaults] profile` in trusted config — the profile a bare `coi` uses.
2. The profile recorded in a resumed session's metadata (`--resume`).
3. The profile saved against an alias.
4. An explicit `--profile <name>`.

They never stack: each path sets `a.profile`, and `applyDefaultProfileFallback` returns
early if anything already claimed it. `[defaults] profile` is stripped from untrusted
project config — a cloned repo must not redirect your no-flag default to a weaker profile.

### 2.4 The field set mirrors the global config, with deliberate deltas

`ProfileConfig` (§B) covers `container`, `limits`, `tool`, `network`, `paths`, `incus`,
`git`, `ssh`, `security`, `monitoring`, `timezone`, `shell`, `mounts`, `sockets`, `ports`,
`credentials`, `environment`, `env_commands`, `forward_env`, `context`, `inherits`.

It is a *mirror*, not a copy (§B2): profiles have no `[defaults]` (its members are hoisted
to top level — `[environment]`, `forward_env`, `[env_commands]`), no `[detection]`, no
`env_command_timeout`, no way to set `[defaults] profile`, and use `[[mounts]]` where the
global config uses `[[mounts.default]]`. The model is at `[tool.claude] model`, not
top-level. The 0.8.0 `[container]` refactor was what made the two shapes symmetric for
image/persistence/build (§J).

Struct and JSON Schema are exactly in sync (§B1), and the schema is exported for third
parties via `coi schema profile` / `coi validate profile <path>`.

### 2.5 Two merge algorithms — the single most surprising thing here

| | Inheritance (`inherits =`) | Application (`--profile`) |
|---|---|---|
| when | at load, flattened once | per invocation, onto resolved config |
| scalars | child wins | profile wins |
| maps (`environment`) | deep-merge, `""` clears a parent key | overlay |
| `mounts`, `sockets`, `ports`, `credentials`, `forward_env` | **replace** if child defines them | **append** |
| struct sections | field-by-field | field-by-field |

Because application appends, `ApplyProfile` is **not idempotent**; that's why
`ReapplyProfileContainer` exists as a container-section-only, re-runnable variant (§C1).

Inheritance chains are capped at 10 levels with cycle detection, and resolve across
scopes — a project profile may inherit from a user profile (§C3).

---

## 3. Relationship to Incus

### 3.1 A COI profile is not an Incus profile

This is the finding most likely to be assumed wrong. Incus has its own first-class
`profile` object (a reusable set of instance config keys and devices, attachable with
`incus init -p <name>`). **COI never creates, names, writes, or attaches one.**
`incus init` is invoked with only `<image> <name>`, optionally `--ephemeral` and
`-s <pool>` (§G1), so every COI container gets Incus's own `default` profile implicitly.

COI touches Incus profiles in exactly five places, four of them read-only inspections of
Incus's `default` profile (§G2): the privileged-container guard, two `coi health` posture
checks, bridge-name discovery for nft rules, and one *install-time* write in `install.sh`
that points the default profile's root device at the ZFS pool.

So the word "profile" is overloaded across the two systems. When a COI doc says "the
default profile has `security.privileged=true`", that's the *Incus* one; when it says
"`--profile hardened`", that's the *COI* one.

### 3.2 They do overlap — on vocabulary, not plumbing

The two genuinely share a lot of expressible policy. What they don't share is any
mechanism: every key in the middle column below is one COI writes **directly to the
instance**, never through an Incus profile object.

| Only an Incus profile | Both can express it | Only a COI profile |
|---|---|---|
| `gpu`, `usb`, `tpm`, `pci`, `infiniband`, `unix-char`/`unix-block` devices | **CPU limits** — `limits.cpu`, `.allowance`, `.priority` | **Which image to launch** (`[container] image`) — an Incus profile cannot carry an image at all |
| The root `disk` device definition itself (`pool`, `size`) | **Memory limits** — `limits.memory`, `.enforce`, `.swap` | **Network egress policy** — `restricted`/`allowlist`, allowed domains, private-net + metadata blocks (host nft/iptables) |
| `boot.autostart`, `boot.host_shutdown_timeout` | **Disk I/O limits** — root device `limits.read`/`write`/`max`, `limits.disk.priority` | `[[network.hosts]]` — static name→IP in the container's `/etc/hosts` |
| `cloud-init.*` (user-data, vendor-data, network-config) | **Process cap** — `limits.processes` | **Secret masking** (`secret_paths`), **protected paths**, host `chattr +i` immutability |
| `snapshots.schedule`, `.expiry`, `.pattern` | **Bind mounts** — `disk` devices | **Threat monitoring** — auto-pause/kill, nft monitoring |
| `migration.stateful`, `nvidia.*`, `raw.lxc`, `limits.kernel.*`, `limits.hugepages.*` | **Socket forwarding & port publishing** — `proxy` devices | **Tool selection** — `[tool] name`, `permission_mode`, `[tool.claude] model`/`effort_level` |
| `environment.*` (Incus's own env-injection keys) | **Container hardening** — `security.nesting`, `.idmap.isolated`, `.guestapi`, `.syscalls.intercept.*` | **Agent context injection** (`context = "CONTEXT.md"`) |
| `security.protection.delete` / `.shift` | **sysctls** — `linux.sysctl.*` | **Image build recipe** (`[container.build]`) |
| Arbitrary `user.*` metadata (COI writes only `user.coi.alias`) | **Storage pool** — Incus via the root device, COI via `incus init -s` | **Credentials**, **ephemerality**, **runtime limits** (`max_duration`, `auto_stop`), **timezone**, **git identity/hooks**, **shell/tmux**, **inheritance**, **the trust model**, **session/alias memory** |
| VM-specific keys (COI is containers-only) | **Environment variables** — same goal, disjoint mechanism (below) | |
| **Composition**: an instance takes an *ordered list* of profiles, later wins | | **Composition**: exactly one COI profile applies, flattened at load |

**Environment variables are the instructive near-miss.** Incus profiles have first-class
`environment.FOO=bar` keys. COI's `[environment]` never uses them — it injects at exec
time. Same capability, entirely disjoint mechanism.

**False friends.** Several COI TOML keys borrow Incus-looking names and are not Incus keys
at all: `[security] protected_paths` / `writable_paths` / `host_immutable` /
`disable_protection` / `secret_paths` are COI concepts (read-only bind mounts + host
`chattr +i`), and `[limits.runtime] max_duration` / `auto_stop` are a host-side Go monitor.
Conversely `security.privileged`, `raw.apparmor` and `raw.seccomp` *are* real Incus keys,
but COI only ever **reads** them — on Incus's `default` profile — to refuse a launch or
fail a health check (§G2).

### 3.3 "Both produce an Incus container" — a frame that half-holds

It is tempting to unify them as *two kinds of input to producing a container*. That is a
fair first approximation for the middle column above, and it breaks in three places worth
knowing.

**Neither actually creates one.** An Incus profile cannot, even in principle — profiles
carry config and devices, never an image; `incus launch <image> <name> -p <profile>`
creates the instance and the profile only shapes it. A COI profile is closer, since it
does carry `[container] image`, but it still needs a workspace and slot to produce a name.
Both are shapers; the Incus one is strictly less sufficient.

**The binding has opposite lifetimes.** An Incus profile stays *attached*: it is a live
reference in the instance's config, profile-derived values show as inherited, and editing
the profile re-shapes every instance that references it. A COI profile is *consumed* —
flattened at load, applied as imperative calls, and then nothing on the Incus side
remembers it. The only COI metadata written to the instance is `user.coi.alias`; there is
no `user.coi.profile` (§G6). The record lives in a host-side session JSON. Edit a COI
profile and existing containers do not change.

And it is not uniformly consumed-once. The split (§G7):

| Baked in at creation | Re-established every session |
|---|---|
| image, storage pool, `[limits.*]`, `[[mounts]]`, the `security.*` hardening keys, `raw.idmap` | `[[sockets]]`, `[ports]`, `[[credentials]]`, network egress rules, `[security] protected_paths` and `secret_paths` |

Both halves are deliberate. Sockets, ports and credentials are re-applied every launch
specifically so an untrusted repo config cannot smuggle them onto a *reused* container
(gating only at creation would leave that hole). Protected paths and secret masks are
stripped and re-added on reuse as of #610, so protection always matches the *current*
workspace rather than the one that existed at first launch. `[[mounts]]` is the
acknowledged exception — creation-only, with a warning telling you to kill and relaunch.

**A COI profile's effect is not confined to the container.** Host firewall rules on the
bridge, `chattr +i` on host workspace files, a host monitor process, host `[paths]`
directories — none of that is instance state, and Incus has no vocabulary for it. And
`coi build --profile X` produces an *image*, not a container.

So the frame that holds:

> An **Incus profile** is a live, named, server-side fragment of instance state that an
> instance references.
> A **COI profile** is a recipe for one session — part baked into an instance at creation,
> part re-applied on every launch, and part executed on the host entirely outside the
> container.

They overlap on *what a container should look like*, and diverge on *who remembers it, for
how long, and how much of it is even inside the container*.

### 3.4 What a COI profile actually becomes

Per-instance Incus state, applied imperatively at launch (§G3):

| Profile declares | Becomes |
|---|---|
| `[container] image` | `incus init <image> <name>` |
| `[container] persistent = false` | `incus init --ephemeral` |
| `[container] storage_pool` | `incus init -s <pool>` (validated up front) |
| `[incus] project` | `--project` on every `incus` call |
| `[limits.*]` | `incus config set` (`limits.cpu`, `limits.memory`, …) |
| `[[mounts]]` | `incus config device add … disk` |
| `[[sockets]]`, `[ssh] forward_agent`, `[ports]` | `incus config device add … proxy` |
| `[network]` | **not Incus at all** — host-side nft/iptables rules |
| `[[credentials]]`, `[security]`, `[monitoring]`, `[tool]`, `context` | COI-side: file copies, read-only bind-mount sets, host `chattr +i`, the monitor goroutine |

The baseline container hardening (`security.nesting`, syscall intercepts,
`security.guestapi=false`, `security.idmap.isolated`) is applied to every instance
regardless of profile — a profile can't opt out of it.

### 3.5 Profile is not part of container identity — a real consequence

Container names are `prefix + workspaceHash + slot`; the profile name is nowhere in them
(§G4). For a **persistent** container, reuse is decided by that name alone and sets
`skipLaunch = true`, bypassing the code that consumes `[container] image`.

**Therefore:** launch a persistent container under `--profile a` (image `img-a`), then
relaunch the same workspace/slot under `--profile b` (image `img-b`), and you get the
original `img-a` container back with `b`'s runtime settings layered on. The image change
is silent. Profile switching is only fully honoured on ephemeral containers or after
killing the persistent one.

---

## 4. Semantically — what they're *for*

Read across the code and the CHANGELOG, profiles carry five distinct jobs. They started as
job 1 and accreted the rest.

1. **Reusable workload template.** The original intent, and still the documented best
   practice: one profile per *kind of work* (`rust-dev`, `python-ml`), not per project.
   Per-project settings belong in that project's `.coi/config.toml`.

2. **Security-posture selector.** Present almost from the start — "per-profile domain
   allowlists for different security contexts" (§J) — and now the headline use via
   `hardened`: one flag flips network mode, secret masking, ephemerality, SSH-agent
   forwarding, and monitoring together. A profile is the unit at which you say "I trust
   this code this much."

3. **Image-build unit.** `[container.build]` with a `base` plus a script or inline
   commands makes the profile the definition of a custom image. `coi build --profile X`
   builds one; `coi build --all` builds every visible profile that declares a build. The
   built image lands in the same storage pool the profile resolves to.

4. **Tool, credential and agent-context unit.** `[tool] name` selects the coding agent
   (`claude`, `opencode`, …), `[tool.claude] model` picks the model, `[[credentials]]`
   declares what gets copied in, and `context = "CONTEXT.md"` injects profile-specific
   instructions into `~/SANDBOX_CONTEXT.md` and the tool's native context file. "A per-tool
   profile carries the tool's whole setup, not just its name" is the stated rationale for
   deleting `--tool` (§J).

5. **The replacement for CLI flags.** The 0.9→0.10 arc deleted every config-shaped flag
   (`--image`, `--persistent`, `--tmux`, `--tool`, `--limit-*`, `--mount`, `--env`,
   `--network`, …) and every env-var override. Flags are now strictly per-invocation
   (`--workspace`, `--slot`, `--resume`, `--profile`); everything else is config or
   profile. Profiles absorbed most of what those flags did, which is why `ProfileConfig`
   grew to near-parity with `Config`.

### 4.1 …and a sixth, implicit job: participating in the trust boundary

A profile's **location determines its authority** (§D/Q4). Profiles under `~/.coi` or
`dirname($COI_CONFIG)` are trusted. Profiles under the workspace's `./.coi` are
**untrusted** — a cloned repo or an in-container agent can plant one — so at load time COI:

- strips security-weakening network settings (`mode=open`,
  `block_private_networks=false`, `block_metadata_endpoint=false`,
  `allow_local_network_access=true`) and `[[network.hosts]]` outright, the latter because a
  name→IP mapping is a spoofing primitive;
- strips `security.disable_protection`, `protected_paths` (a full replace),
  `writable_paths`, `host_immutable`, `git.writable_hooks`, and `git.name`/`git.email`
  (a checkout must not choose the container's commit identity);
- strips `env_commands` outright — running a host command is host code execution, and
  unlike mounts there is no approve flow for it;
- keeps `[[mounts]]`, `[[sockets]]`, `[ports]` and ad-hoc `[[credentials]]` but tags them
  `Untrusted` with their source path, so escaping mounts, all forwarded sockets, host port
  listeners and ad-hoc credential copies are gated behind `coi trust` at apply time.

Each stripped field prints a warning naming the file and the fix. An untrusted profile can
always *add* protection (`additional_protected_paths`), never remove it.

---

## 5. Documentation drift

The wiki (head `ea3c7a9`, 2026-07-29) is now roughly aligned with master (2026-08-01).
Its `[defaults] profile` section — including the 4-level precedence table and the
trusted-scope rule — the `hardened` table, `[tool.claude] model`, and `[[credentials]]`
all check out against the code (§I3). What remains:

### 5.1 Wiki claims that are wrong

- **Scan locations** — documented as two levels; the code has three. `dirname($COI_CONFIG)`
  is undocumented (§A1).
- **What `default` contains** — "reflects the embedded default configuration." It reflects
  the *resolved* config, user and project overrides included (§A3, P1/Q2). Practical
  consequence: if `~/.coi/config.toml` sets `mode = "open"`, then `--profile default` is
  open — it is not a safe-defaults profile. The wiki contradicts itself two sections
  later, calling it "a clean clone of your global config", which is the accurate reading.
- **Merge strategy** — the array bullet names only `mounts` and `forward_env`; the code
  now covers five array-shaped fields, including `sockets`, `ports` and `credentials` (§C2).
- **Field table** — omits `sockets`, `ports`, `shell`, `env_commands`, `network.hosts`, and
  all of `security.{secret_paths, writable_paths, additional_protected_paths,
  disable_protection}`.
- **"The built-in `default` profile cannot be edited or deleted"** — the guard keys on
  `Source == "(built-in)"`, so `hardened` is equally protected; only the error message
  says "default" (§H3).
- **The untrusted-profile trust model is entirely absent from the Profiles page** — it
  lives only in code comments and CHANGELOG entries, and it has grown (ports, credentials,
  network.hosts, git identity all gained rules). For a security tool this is the most
  consequential omission, since it's the difference between "a repo can configure my
  sandbox" and "a repo can *weaken* my sandbox."
- **Sample `coi profile list` output** lacks the `INHERITS` column the code renders.

### 5.2 Code-level gaps found while validating

1. **Profile precedence inverts on the cross-workspace path** (§E, probe P2). The README's
   `defaults < user < project < profile` holds when you run inside the workspace. But with
   an alias or `--workspace <elsewhere>`, the target repo's `.coi/config.toml` is overlaid
   **after** the profile, and only `[container]` is re-asserted afterwards. Measured: the
   repo's config overrode the profile's `allowed_domains` (narrow allowlist → the repo's
   list), `limits.memory` (1GiB → 64GiB), and `tool.permission_mode`
   (`interactive` → `bypass`). Untrusted sanitization still blocks `mode=open`, the
   block-flags and `network.hosts`, so this can't reach *below* built-in defaults — but it
   can override a profile that is *stricter* than default, which is exactly what one asks a
   profile for. The `ReapplyProfileContainer` comment says the intent was "an explicitly
   requested `--profile` must keep winning"; it only achieves that for `[container]`.
   **This is the one worth acting on.**

2. **Validation asymmetry** (§F, probe P3). A profile `config.toml` is schema-validated
   with `additionalProperties: false`, so `mdoe = "open"` is a hard, precisely-located
   error. The same typo in `~/.coi/config.toml` is silently ignored — no schema, no strict
   decode, and `coi validate` has only a `profile` subcommand. Given the 0.10 design where
   config files are the *only* configuration surface, silent key-drop in the main config is
   the higher-risk half.

3. **Legacy inline `[profiles.<name>]` tables are silently ignored.** `Config.Profiles` is
   tagged `toml:"-"`, and `deprecated.go` has migration traps for the 0.8.0
   image/persistent/build moves but none for inline profile tables. Anyone with an old
   config gets a profile that quietly doesn't exist rather than a migration error.

4. **Persistent-container reuse ignores a changed profile image** (§3.5 above).

---

## 6. Practical guidance implied by the above

- Treat `--profile default` as "whatever my config resolves to", not as "safe defaults".
  For safety use `--profile hardened`.
- Prefer running from inside the workspace when the profile's non-container settings
  matter; the alias / `--workspace` path lets the repo's config override them.
- Switching profiles on a persistent container won't change its image. Kill it first.
- Run `coi validate profile <path>` after editing a profile; there is no equivalent safety
  net for `config.toml`, so double-check key names there by hand.
- Keep profiles as workload archetypes and per-project deltas in `.coi/config.toml` — the
  merge order is designed around that split, and profiles-per-project multiplies the
  duplicate-name hazard.
- If you want an opinionated setup without typing `--profile` every time, set
  `[defaults] profile` in `~/.coi/config.toml`; `coi --profile default` still gets you back
  to the plain clone of global config.

---

## 7. What the fast-forward to `da67215` changed

The first pass ran against `afb96e8`, 95 commits earlier. Re-running every probe (§L):

**One finding resolved.** `sockets` were silently dropped by profile inheritance —
`mergeProfiles` handled `Mounts` and `ForwardEnv` but had no `Sockets` branch, so a child
inheriting from a parent that declared `[[sockets]]` got none. It now has branches for
`Sockets`, `Ports` and `Credentials` (§C2, P1/Q1).

**New profile surface:** `[defaults] profile`, `[[credentials]]`, `[ports]`,
`[[network.hosts]]`, `[container] ready_timeout`, and `git.name`/`git.email`/
`git.seed_host_identity` — each arriving with its own untrusted-scope rule, which is why
§4.1 is longer than it was. The model moved: top-level profile `model` and `[defaults]
model` were both removed in favour of `[tool.claude] model`. The default allowlist dropped
`8.8.8.8`/`1.1.1.1`, since DNS egress is blocked and allowlisting resolvers bought nothing.

**Everything else re-confirmed unchanged** — the Incus relationship, container identity,
both merge algorithms, the `default`-is-a-clone behaviour, the cross-workspace precedence
asymmetry, and the validation asymmetry.
