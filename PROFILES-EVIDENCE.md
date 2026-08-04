# COI Profiles — Evidence Log

Raw evidence gathered while answering "what are profiles, functionally, in relation to
Incus, and semantically". Companion to [`PROFILES-FINDINGS.md`](PROFILES-FINDINGS.md),
which contains the conclusions. This file is the audit trail: every claim there should be
traceable to a citation or probe output here.

**Investigated checkout:** `master` @ `da67215` (2026-08-01).
**Method:** three passes. Pass 1 read the code, schema, README and wiki. Pass 2 re-derived
every behavioural claim by running the real loader against synthetic profile trees
(probes P1–P3). Pass 3 re-ran all of it after master fast-forwarded from `afb96e8` to
`da67215` (95 commits) — see §L for what that changed.

**Known limitation:** the clone is shallow (`.git/shallow` present), so pre-2026 history is
unavailable. Design-intent chronology (§J) is sourced from `CHANGELOG.md`, not `git log`.

---

## A. Where profiles live and how they are loaded

### A1. Scan locations — three, not two

```go
// internal/config/config.go:792-800
dirs := []string{
	filepath.Join(homeDir, ".coi"), // 1. User home
	filepath.Join(workDir, ".coi"), // 2. Project
}
// COI_CONFIG environment variable: scan its parent dir for profiles too
if envConfig := os.Getenv("COI_CONFIG"); envConfig != "" {
	dirs = append(dirs, filepath.Dir(envConfig))
}
```

Each parent dir is scanned for a `profiles/` subdirectory; each subdirectory containing a
`config.toml` becomes a profile named after the directory
(`internal/config/loader.go`, `loadProfileDirectories`).

### A2. Single flat namespace, duplicates are a hard error

```go
// internal/config/loader.go (loadProfileDirectories)
if existing, ok := cfg.Profiles[profileName]; ok && existing.Source != "" && existing.Source != profileConfigPath {
	return fmt.Errorf(
		"profile %q defined in multiple locations:\n  %s\n  %s\n"+
			"Rename one of them or delete the duplicate so it's clear which profile is being used",
		profileName, existing.Source, profileConfigPath,
	)
}
```

### A3. Load order in `Load()`

```go
// internal/config/loader.go:27-82  (abridged)
cfg := GetDefaultConfig()                                   // 1. embedded defaults
for _, dir := range GetProfileParentDirs() {                // 2. scan profile dirs
	loadProfileDirectories(cfg, dir, isTrustedProfileDir(dir))
}
for _, path := range GetConfigPaths() {                     // 3. ~/.coi, ./.coi, $COI_CONFIG
	loadConfigFileScoped(cfg, path, isTrustedConfigPath(path))
}
if _, exists := cfg.Profiles["default"]; !exists {          // 4. synthesize built-ins
	cfg.Profiles["default"] = synthesizeDefaultProfile(cfg)
}
if _, exists := cfg.Profiles["hardened"]; !exists {
	cfg.Profiles["hardened"] = synthesizeHardenedProfile()
}
cfg.ResolveProfileInheritance()                             // 5. flatten inherits
```

Step 4 runs **after** step 3 — so the synthesized `default` profile is a snapshot of the
config **after** user and project config were merged in. Verified empirically in P1/Q2.

### A4. Schema validation is applied to profile files only

```go
// internal/config/loader.go (loadProfileDirectories)
var rawMap map[string]any
toml.DecodeFile(profileConfigPath, &rawMap)
if schemaErr := coischema.ValidateProfileMap(rawMap); schemaErr != nil {
	return fmt.Errorf("profile %q at %s failed schema validation: %w", ...)
}
```

`loadConfigFileScoped` (used for `config.toml`) has no equivalent call.
`schema/profile.schema.json` sets `"additionalProperties": false`.

---

## B. The `ProfileConfig` shape

```go
// internal/config/config.go:357-382
type ProfileConfig struct {
	Inherits    string            `toml:"inherits"`
	Container   ContainerConfig   `toml:"container"`
	Context     string            `toml:"context"`
	Environment map[string]string `toml:"environment"`
	EnvCommands map[string]string `toml:"env_commands"`
	Limits      *LimitsConfig     `toml:"limits"`
	Tool        *ToolConfig       `toml:"tool"`
	Mounts      []MountEntry      `toml:"mounts"`
	Sockets     []SocketEntry     `toml:"sockets"`
	Ports       *PortsConfig      `toml:"ports"`
	Credentials []CredentialEntry `toml:"credentials"`
	Network     *NetworkConfig    `toml:"network"`
	ForwardEnv  []string          `toml:"forward_env"`
	Source      string            `toml:"-"`

	// Extended fields — previously Config-only, now available in profiles
	Paths      *PathsConfig      `toml:"paths"`
	Incus      *IncusConfig      `toml:"incus"`
	Git        *GitConfig        `toml:"git"`
	SSH        *SSHConfig        `toml:"ssh"`
	Security   *SecurityConfig   `toml:"security"`
	Monitoring *MonitoringConfig `toml:"monitoring"`
	Timezone   *TimezoneConfig   `toml:"timezone"`
	Shell      *ShellConfig      `toml:"shell"`
}
```

Note there is **no top-level `model`** — the model lives at `[tool.claude] model`:

```go
// internal/config/config.go:394-398
type ClaudeToolConfig struct {
	EffortLevel string `toml:"effort_level"`
	Model       string `toml:"model"` // delivered as ANTHROPIC_MODEL
}
```

### B1. Struct ↔ schema are in sync

```
$ python3 - <<'PY'   # ProfileConfig toml tags vs schema/profile.schema.json properties
in schema not struct: []
in struct not schema: []
PY
```

Both sides: `container, context, credentials, env_commands, environment, forward_env, git,
incus, inherits, limits, monitoring, mounts, network, paths, ports, security, shell,
sockets, ssh, timezone, tool`.

### B2. Profile shape vs. global `Config` shape — deliberate deltas

`Config` has `[defaults]` and `[detection]`; `ProfileConfig` has neither. Instead:

| Global config | Profile equivalent |
|---|---|
| `[defaults] environment` / `forward_env` / `env_commands` | top-level `[environment]` / `forward_env` / `[env_commands]` |
| `[defaults] profile` | **no profile equivalent** (a profile can't pick the default profile) |
| `[defaults] env_command_timeout` | **no profile equivalent** |
| `[[mounts.default]]` | `[[mounts]]` |
| `[detection]` | **no profile equivalent** |

---

## C. Two different merge algorithms

### C1. `ApplyProfile` — override scalars, **append** slices

```go
// internal/config/config.go (ApplyProfile, abridged)
if len(profile.ForwardEnv)  > 0 { c.Defaults.ForwardEnv = MergeStringSliceUnique(...) }
if len(profile.Mounts)      > 0 { c.Mounts.Default = append(c.Mounts.Default, profile.Mounts...) }
if len(profile.Sockets)     > 0 { c.Sockets       = append(c.Sockets, profile.Sockets...) }
if len(profile.Credentials) > 0 { c.Credentials   = append(c.Credentials, profile.Credentials...) }
```

Because the slice fields append, `ApplyProfile` is **not idempotent** — which is why
`ReapplyProfileContainer` exists:

```go
// internal/config/config.go (ReapplyProfileContainer doc comment)
// Unlike a full ApplyProfile, this is idempotent:
// mergeContainerInto/mergeBuildInto are field-level overrides with no
// appends, so it is safe to call after a profile was already applied
// (a full re-apply would duplicate profile mounts and sockets).
```

### C2. `mergeProfiles` (inheritance) — override scalars, **replace** slices

```go
// internal/config/config.go:1287-1299
// Arrays: if child defines them, they fully replace parent's. If not, inherit.
if result.Mounts == nil      { result.Mounts = parent.Mounts }
if result.Sockets == nil     { result.Sockets = parent.Sockets }
if result.Ports == nil       { result.Ports = parent.Ports }
if result.Credentials == nil { result.Credentials = parent.Credentials }
if result.ForwardEnv == nil  { result.ForwardEnv = parent.ForwardEnv }
```

All five array-shaped fields are covered. (On `afb96e8` `Sockets` was **missing** from
this block — see §L.)

### C3. Inheritance resolution

`ResolveProfileInheritance`: first pass walks each chain to detect cycles and enforce
`maxInheritanceDepth = 10`; second pass merges parent into child and flattens.
`merged.Inherits = parentName` is preserved for display only.

---

## D. Probe P1 — inheritance, built-ins, untrusted sanitization

A temporary `internal/config/zz_probe_test.go` built a synthetic tree and called the real
`Load()`. Setup: a trusted `~/.coi/config.toml` with `mode = "open"`,
`[tool.claude] model = "user-global-model"` and one `[[sockets]]`; a user profile `base`
with mounts/sockets/credentials/ports/env; a user profile `child` with `inherits = "base"`
and nothing array-shaped; an untrusted project profile `evil` attempting downgrades.

```
=== namespace ===
  base      source=<tmp>/001/.coi/profiles/base/config.toml   inherits=""
  child     source=<tmp>/001/.coi/profiles/child/config.toml  inherits="base"
  evil      source=<tmp>/002/.coi/profiles/evil/config.toml   inherits=""
  default   source=(built-in)                                 inherits=""
  hardened  source=(built-in)                                 inherits=""

=== Q1: inheritance of array-shaped fields ===
  child.Mounts      len=1
  child.ForwardEnv  [PARENT_VAR]
  child.Sockets     len=1          <-- inherited (was the gap on afb96e8)
  child.Credentials len=1
  child.Ports       true
  child.Environment map[FROM_PARENT:1]    <-- KEEP_ME cleared by ""

=== Q2: is `default` a clone of RESOLVED config? ===
  default.Network.Mode="open"      (embedded default is "restricted"; user cfg said open)
  default.Tool.Claude.Model="user-global-model"
  default.Sockets len=1            (the user config's socket)

=== Q3: hardened ===
  Source="(built-in)" Network.Mode="restricted" Persistent=false

=== Q4: untrusted project-profile sanitization ===
  Network.Mode="" BlockPrivateNetworks=<nil> Hosts=[]
  Security.DisableProtection=false Git.WritableHooks=<nil> Git.Name="" Git.Email=""
  EnvCommands=map[]
  Mounts[0].Untrusted=true
  Credentials[0].Untrusted=true
  Ports.PoolUntrusted=true

=== Q5: ApplyProfile idempotence ===
  mounts before=0 after 1x=1 after 2x=2      <-- non-idempotent
  sockets=3 credentials=2
```

Warnings emitted to stderr during the untrusted load (verbatim):

```
WARNING: ignoring security-downgrading "network.block_private_networks=false" in project config .../evil/config.toml; move it to ~/.coi/config.toml or set COI_CONFIG to apply it.
WARNING: ignoring security-downgrading "network.mode=open" in project config .../evil/config.toml; ...
WARNING: ignoring security-downgrading "network.hosts" in project config .../evil/config.toml; ...
WARNING: ignoring 'security.disable_protection' in project config .../evil/config.toml; removing read-only protection is a security downgrade. ...
WARNING: ignoring 'git.writable_hooks' in project config .../evil/config.toml; ...
WARNING: ignoring 'git.name'/'git.email' in project config .../evil/config.toml; a project checkout must not choose the container commit identity. ...
WARNING: ignoring 'env_commands' in project profile .../evil/config.toml; running a host command is host code execution. Move it to a profile under ~/.coi/profiles to apply it.
```

The untrusted branch of `loadProfileDirectories` calls, in order:
`sanitizeUntrustedNetwork`, `sanitizeUntrustedSecurity`, `sanitizeUntrustedGit`,
`markUntrustedMounts`, `markUntrustedSockets`, `markUntrustedPorts`,
`markUntrustedCredentials`, then strips `env_commands`.

Rationales, quoted from the code:

```go
// sanitizeUntrustedNetwork — network.hosts
// A name→IP mapping is a spoofing primitive (redirect api.anthropic.com to
// an attacker's box) and reachability punches a firewall hole. Honor it
// only from trusted scope.

// sanitizeUntrustedGit — git.name / git.email
// a cloned/agent-planted repo must not decide who its commits appear to be
// authored by (the whole reason COI reads only the host's *global* git
// config, never project-local).

// markUntrustedPorts
// a repo declaring host listeners can squat well-known localhost ports (e.g.
// a fake postgres on 5432 capturing the host's own local connections).
```

---

## E. Probe P2 — profile vs. workspace-overlay precedence

Setup: trusted user profile `tight` (`image = "profile-image"`, `mode = "allowlist"`,
`allowed_domains = ["only-this.example"]`, `memory.limit = "1GiB"`,
`permission_mode = "interactive"`). A *different* workspace ships an untrusted
`.coi/config.toml` widening all four.

```
after ApplyProfile(tight)          image="profile-image" mode="allowlist" domains=[only-this.example]            mem="1GiB"  perm="interactive"
after OverlayProjectConfig(other)  image="repo-image"    mode="allowlist" domains=[anything.example evil.example] mem="64GiB" perm="bypass"
after ReapplyProfileContainer      image="profile-image" mode="allowlist" domains=[anything.example evil.example] mem="64GiB" perm="bypass"
in-workspace: Load+ApplyProfile    image="profile-image" mode="allowlist" domains=[only-this.example]            mem="1GiB"  perm="interactive"
```

This mirrors the real cross-workspace code path:

```go
// internal/cli/phases_shell.go (alias path, abridged)
if s.aliasArg != "" {
	resolved, _ := alias.ResolveAliasForLaunch(s.aliasArg)
	a.cfg.OverlayProjectConfig(resolved.Workspace)      // untrusted repo config, AFTER the profile
	if resolved.Profile != "" && !cmd.Flags().Changed("profile") { ... }
}
if cmd.Flags().Changed("profile") && a.profile != "" {
	a.cfg.ReapplyProfileContainer(a.profile)            // re-asserts [container] ONLY
}
```

Same shape in `internal/cli/run.go` (`overlayWorkspaceConfig`, reached when `--workspace`
differs from cwd).

`mergeNetworkInto` replaces the domain list wholesale rather than intersecting:

```go
if src.AllowedDomains != nil { dst.AllowedDomains = src.AllowedDomains }
```

`sanitizeUntrustedNetwork` rejects only `mode=open`, `block_private_networks=false`,
`block_metadata_endpoint=false`, `allow_local_network_access=true`, and `network.hosts`.
It does not touch `allowed_domains`, `[limits]`, or `[tool] permission_mode`.

---

## F. Probe P3 — validation asymmetry

```
main config typo + inline [profiles.x]: err=<nil>
  inline profile registered? false ; network.mode="restricted"

profile-dir typo: err=profile "typo" at .../profiles/typo/config.toml failed schema
  validation: - at '/network': additional properties 'mdoe' not allowed
```

Relevant declarations:

```go
// internal/config/config.go
Profiles map[string]ProfileConfig `toml:"-"` // Populated by loadProfileDirectories, not from TOML
```

`internal/config/deprecated.go` contains migration traps for `[defaults] image`,
`[defaults] persistent`, top-level `[build]`, and the profile-root `image`/`persistent`/
`[build]` — but none for an inline `[profiles.*]` table.

`coi validate` has exactly one subcommand: `validateCmd.AddCommand(validateProfileCmd)`
(`internal/cli/validate.go:111`). There is no `coi validate config`.

---

## G. Relationship to Incus

### G1. COI never creates or attaches an Incus profile

```go
// internal/container/commands.go:488-496
args := []string{"init", imageAlias, containerName}
if ephemeral { args = append(args, "--ephemeral") }
if pool != "" { args = append(args, "-s", pool) }
if err := IncusExec(args...); err != nil { return err }
```

No `-p` / `--profile` is ever passed — `grep -rn '"-p"|"--profile"'` across
`internal/container/`, `internal/session/`, `internal/image/` returns only an unrelated
`mkdir -p`. The only global flag COI injects is the project:

```go
// internal/container/commands.go
incusArgs := append([]string{"--project", IncusProject}, args...)
```

### G2. Every reference to an *Incus* profile is read-only inspection of Incus's `default`

| Site | Purpose |
|---|---|
| `internal/container/security.go:30-38` | refuse to launch if `security.privileged=true` on the default Incus profile |
| `internal/health/checks.go:2575` | same, as a CRITICAL health check |
| `internal/health/checks.go:2628` | seccomp / AppArmor posture on the default Incus profile |
| `internal/network/nft_filter.go` (`GetIncusBridgeName`) | `incus profile device show default` → bridge name for nft rules |
| `install.sh` | `incus profile device set default root pool=zfs-pool` — the one *write*, at install time, not per-session |

### G3. COI profile fields → per-instance Incus state

| Profile field | Mechanism |
|---|---|
| `[container] image` | `incus init <image> <name>` |
| `[container] persistent = false` | `incus init --ephemeral` |
| `[container] storage_pool` | `incus init -s <pool>`; validated by `container.ValidateStoragePool` |
| `[incus] project` | `--project` on every `incus` call |
| `[limits.*]` | `incus config set` (`internal/limits/applier.go`) |
| `[[mounts]]` | `incus config device add … disk` (`manager.go`) |
| `[[sockets]]`, `[ssh] forward_agent` | `incus config device add … proxy` (`manager.go`) |
| `[ports]` | `incus config device add … proxy` (host listener → container port) |
| `[limits.disk] tmpfs_size` | `incus config device override <name> tmp disk` |
| `[network]` | host-side nft/iptables rules, **not** Incus config |
| `[[credentials]]`, `[security]`, `[monitoring]`, `[tool]`, `context` | COI-side: file copies, read-only bind mounts, host `chattr +i`, monitor goroutine |

Baseline hardening is applied per instance regardless of profile:
`security.nesting`, `security.syscalls.intercept.mknod`/`setxattr`,
`security.guestapi=false`, `security.idmap.isolated=true`
(`internal/container/commands.go`).

### G4. Profile is not part of container identity

```go
// internal/session/naming.go:43-47
func ContainerName(workspacePath string, slot int) string {
	hash := WorkspaceHash(workspacePath)
	prefix := GetContainerPrefix()
	return fmt.Sprintf("%s%s-%d", prefix, hash, slot)
}
```

Reuse short-circuits the launch entirely — `internal/session/setup.go` sets
`skipLaunch = true` at lines 214, 246 and 306, and the launch block is guarded by
`if !skipLaunch` (line 379). `skipLaunch` bypasses the code that consumes
`[container] image`.

### G4a. The exact Incus key surface COI writes

Extracted with
`grep -rhoE '"(limits|security|raw|environment|boot|cloud-init|user|linux|migration|snapshots|nvidia)\.[a-zA-Z0-9._]+' internal/ --include=*.go | sort -u`,
then filtered to genuine Incus keys (see §G4b for the false friends the raw grep also
catches).

**Instance-level (`incus config set`):**

```
limits.cpu                                limits.memory
limits.cpu.allowance                      limits.memory.enforce
limits.cpu.priority                       limits.memory.swap
limits.disk.priority                      limits.processes
raw.idmap                                 security.guestapi
security.nesting                          security.idmap.isolated
security.syscalls.intercept.mknod         security.syscalls.intercept.setxattr
linux.sysctl.net.ipv4.ip_unprivileged_port_start
linux.sysctl.net.ipv6.conf.all.disable_ipv6
linux.sysctl.net.ipv6.conf.default.disable_ipv6
user.coi.alias
```

**Device-level (`incus config device set`):**

```
root disk : limits.read, limits.write, limits.max      (internal/limits/applier.go:124-136)
nic       : security.ipv4_filtering, security.mac_filtering, security.port_isolation
                                                        (internal/container/commands.go:596-616)
```

**Read-only — never written:** `security.privileged`, `raw.apparmor`, `raw.seccomp`.
These appear only in the privileged guard and the health posture checks (§G2), which
inspect Incus's `default` profile.

Notably absent: `environment.*`. Incus profiles have first-class per-instance environment
keys; COI's `[environment]` never uses them and injects at exec time instead.

### G4b. False friends in the grep output

The same grep also matches COI's own TOML key names, which are not Incus keys:

| Looks like an Incus key | Actually |
|---|---|
| `security.protected_paths`, `security.writable_paths`, `security.host_immutable`, `security.disable_protection`, `security.secret_paths` | COI config keys — implemented as read-only bind mounts and host `chattr +i` |
| `limits.runtime.max_duration`, `limits.runtime.auto_stop` | a host-side Go monitor (`internal/limits/`), no Incus key |
| `user.name`, `user.email`, `user.useConfigOnly` | `git config` keys, not Incus `user.*` metadata |

### G5. Profile name *is* persisted per session and per alias

```go
// internal/session/cleanup.go
ProfileName string // Profile used for this session (saved in metadata for --resume)
ProfileName string `json:"profile_name"`
```

```go
// internal/cli/phases_shell.go
if !cmd.Flags().Changed("profile") && metadata.ProfileName != "" {
	a.profile = metadata.ProfileName
	a.cfg.ApplyProfile(a.profile)
	fmt.Fprintf(os.Stderr, "Inherited profile '%s' from session\n", a.profile)
}
```

### G6. …but nothing on the Incus side records it

`grep -rn "user.coi" internal/ --include=*.go` (excluding tests) returns **only**
`user.coi.alias` — set in `internal/session/setup.go:584` and
`internal/cli/run.go:437`, read back in `internal/alias/resolve.go:157` and
`internal/cli/list.go:191`. There is no `user.coi.profile`.

So the container carries no Incus-visible trace of which COI profile shaped it; the only
record is `metadata.json` under the sessions dir on the host. Editing a COI profile
therefore cannot re-shape existing containers, which is the opposite of an Incus profile's
live-reference semantics.

### G7. Creation-only vs. re-established every session

`skipLaunch` (set on the reuse paths, §G4) guards the block beginning at
`internal/session/setup.go:379`, which is where limits are applied:

```go
// internal/session/setup.go:379,498
if !skipLaunch {
	...
	if err := limits.ApplyResourceLimits(applyOpts); err != nil { ... }
```

Other settings are deliberately re-applied on reuse. From the trust chokepoint
(`internal/session/setup.go` ~318-330):

```go
// This deliberately runs on the REUSE paths too: sockets, credentials
// (resume), and ports are re-applied from the current config every
// session, so gating only at creation would let an untrusted repo config
// smuggle them onto a reused container. Mount devices are the exception —
// they persist from creation and can't be re-gated here, so on reuse we
// warn instead.
```

and the reuse warning it emits:

```
Warning: N untrusted mount(s) remain attached from when this container was created;
recreate it (coi kill + relaunch) to apply mount-trust changes
```

Protected paths and secret masks are also reconciled on reuse — `StripSecurityDevices`
plus a re-run of the shared `applySessionSecurity`
(`internal/session/security.go:84-138`):

```go
// This is the reuse-path analogue of RemoveStalePortDevices and the heart of the
// issue #610 fix: on a fresh launch SetupSecurityMounts / SetupSecretMasks /
// SetupCommonDirProtection materialize each source and attach the device, but on
// reuse those functions never ran, so a device attached at first launch keeps its
// original host source forever. ... Stripping here + re-running the SAME validated
// setup means ... protection is re-established to match the CURRENT workspace
// (paths added, removed, or replaced since first launch).
```

Network rules are re-applied too: `network.ApplyBootBlockRule` runs on the restart path
(`internal/session/setup.go:293`) before `SetupForContainer` installs the full ruleset.

| Baked in at creation | Re-established every session |
|---|---|
| image, storage pool, `[limits.*]`, `[[mounts]]`, `security.*` hardening keys, `raw.idmap` | `[[sockets]]`, `[ports]`, `[[credentials]]`, network egress rules, `[security] protected_paths` + `secret_paths` |

Roughly — though not exactly — the creation-only column is the state COI writes into Incus
instance config, and the per-session column is COI's own host-side and device work.
`[[mounts]]` is the clean counter-example: an Incus disk device, but creation-only.

---

## H. The built-in profiles and the default-profile setting

### H1. `default` — a live clone of resolved config

```go
// internal/config/config.go (synthesizeDefaultProfile, abridged)
// synthesizeDefaultProfile creates a ProfileConfig from the loaded Config,
// representing the "default" built-in profile.
p := ProfileConfig{ Container: container, Environment: cloneMap(...), ..., Source: "(built-in)" }
```

Everything is copied by value / cloned — a past bug (`CHANGELOG.md`) was pointer aliasing
letting profile operations mutate the global config.

The embedded source of those defaults:

```
# profiles/default/config.toml:1-3
# COI Default Profile — Single Source of Truth for All Defaults
# This file is embedded into the binary and parsed by GetDefaultConfig().
# All values here define the system defaults.
```

`diff internal/config/embedded/default_config.toml profiles/default/config.toml` → identical.

### H2. `hardened` — a fixed baseline, not a clone

```go
// internal/config/config.go (synthesizeHardenedProfile, abridged)
// Unlike the "default" profile this is a FIXED baseline, not a clone of the
// user's resolved config: it sets only the hardened overrides and lets every
// other field fall through. It can be overridden by a same-named disk profile.
return ProfileConfig{
	Source:     "(built-in)",
	Container:  ContainerConfig{Persistent: &f},
	Network:    &NetworkConfig{Mode: NetworkModeRestricted, BlockPrivateNetworks: &t,
	                           BlockMetadataEndpoint: &t, AllowLocalNetworkAccess: &f},
	SSH:        &SSHConfig{ForwardAgent: &f},
	Security:   &SecurityConfig{HostImmutable: &t, SecretPaths: cloneSlice(HardenedProfileSecretPaths)},
	Monitoring: &MonitoringConfig{Enabled: &t, AutoPauseOnHigh: &t, AutoKillOnCritical: &t,
	                              NFT: NFTMonitoringConfig{Enabled: &t}},
}
```

### H3. Both built-ins are equally protected from `edit`/`delete`

```go
// internal/cli/profile.go:636 (edit) and :697 (delete)
if p.Source == "(built-in)" {
	return fmt.Errorf("cannot edit built-in profile 'default'")
}
```

The guard keys on `Source`, so it also fires for `hardened` — but the message names only
`default`. P1/Q3 confirms `hardened.Source == "(built-in)"`.

### H4. `coi profile create default` is special-cased away from profiles entirely

```go
// internal/cli/profile.go
// "default" is the built-in profile; it has no profiles/ directory — its
// values are backed by the main config. So scaffold the main config instead.
if name == "default" { return a.scaffoldMainConfig(cmd) }
```

### H5. `[defaults] profile` — the lowest-priority selector

```go
// internal/config/config.go:269-277
type DefaultsConfig struct {
	// Profile names the profile to apply when `--profile` is not passed, so a
	// user's opinionated setup applies without retyping it (#607). `coi` gives
	// this profile; `coi --profile default` still gives the synthesized clone of
	// global config. Honored ONLY from trusted-scope config ...
	Profile string `toml:"profile"`
```

```go
// internal/cli/root.go:134-155
func (a *App) applyDefaultProfileFallback(cmd *cobra.Command) (bool, error) {
	if a.profile != "" || cmd.Flags().Changed("profile") { return false, nil }
	name := a.cfg.Defaults.Profile
	if name == "" { return false, nil }
	if a.cfg.GetProfile(name) == nil {
		return false, fmt.Errorf(
			"[defaults] profile = %q does not name a known profile; "+
				"run 'coi profile list' to see available profiles", name)
	}
	...
}
```

Called from exactly two sites — `internal/cli/phases_shell.go:249` and
`internal/cli/run.go:115` — both **after** the resume-metadata and alias branches have had
their chance to set `a.profile`. The early return on `a.profile != ""` is what makes it
lowest-priority and non-stacking. It is stripped from untrusted config by
`sanitizeUntrustedDefaultProfile`.

---

## I. Documentation cross-reference

### I1. Version alignment

```
$ git log -1 --format='%H %ad' master
da672154... Sat Aug 1 16:39:39 2026
```

Wiki clone (`code-on-incus.wiki.git`) head: `ea3c7a9` "Correct release version to 0.11.0",
Wed Jul 29 2026 — three days before this master. The four features the wiki documented
that were missing on `afb96e8` (`[defaults] profile`, `[[credentials]]`,
`[tool.claude] model`, `[container] ready_timeout`) are all present now (§L).

### I2. Wiki claims still wrong against `da67215`

- "Profile directories are scanned at two config levels" + a 2-row table. Code has three
  (§A1) — `dirname($COI_CONFIG)` is missing.
- "The built-in `default` profile is always present and reflects the embedded default
  configuration." It reflects the **resolved** config: embedded defaults + user config +
  project config (§A3, P1/Q2). The wiki's own `[defaults] profile` section contradicts
  this two screens later, calling it "a clean clone of your global config".
- Merge-strategy bullet names only `mounts` and `forward_env`; the code covers five
  array-shaped fields including `sockets`, `ports`, `credentials` (§C2).
- Field table omits `sockets`, `ports`, `shell`, `env_commands`, `network.hosts`, and all
  of `security.{secret_paths, writable_paths, additional_protected_paths,
  disable_protection}`.
- "The built-in `default` profile cannot be edited or deleted" — true of `hardened` too (§H3).
- The `coi profile list` sample output has no `INHERITS` column; the code renders
  `NAME, IMAGE, PERSISTENT, INHERITS, SOURCE` (`internal/cli/profile.go:90`).
- The Profiles page never mentions that project-scoped profiles are **untrusted** and
  sanitized (§D/Q4) — a load-bearing security property, documented only in code comments
  and `CHANGELOG.md`.

### I3. Wiki claims that check out

- The `[defaults] profile` section, including its 4-level precedence table
  (`[defaults] profile` < resume-remembered < alias-saved < explicit `--profile`) and the
  trusted-scope-only rule — matches §H5 exactly, including the "hard error only at session
  launch" behaviour.
- The `hardened` table (every row) — matches `synthesizeHardenedProfile` (§H2).
- `[tool.claude]` listing both `model` and `effort_level`, with `model` delivered as
  `ANTHROPIC_MODEL` — matches `ClaudeToolConfig` (§B).
- `[[credentials]]` with `bundle` or ad-hoc `host`/`container`/`mode` — present in
  `ProfileConfig` and `schema/defs/CredentialEntry.json`.

### I4. README, checked against code

`README.md` states the hierarchy as `defaults < user config < project config < profile`.
True for the in-workspace path; false for the alias / `--workspace` path (§E).

`README.md` — "Each profile is a self-contained directory (`.coi/profiles/<name>/`)
bundling a `config.toml` plus optional build script and context file" — matches
`loadProfileDirectories` + `resolveRelativePath`.

---

## J. Design-intent chronology (from `CHANGELOG.md`, oldest first)

| Entry |
|---|
| "TOML-based configuration system with profile support" |
| "Named profiles with environment override support" |
| "Per-profile domain allowlists for different security contexts" |
| Limits via config, profile dirs, **or CLI flags**; precedence then *CLI flags > profile limits > config file* |
| (#114) "Self-contained profile directories" — profiles become directories under `profiles/`; "Precedence: user dirs < project dirs < CLI flags" |
| Scan locations narrowed to `~/.coi/profiles/` + `./.coi/profiles/`; single namespace; duplicate = refuse to start |
| Dropped `/etc/coi/` and `~/.config/coi/` |
| Profile inheritance (`inherits`), 10-level chains, cycle detection, INHERITS column |
| `profile create` / `edit` / `delete`; later `profile show` → `profile info` |
| 0.8.0: `[container]` section unifies profile and global shape; legacy layouts refused with migration errors |
| Embedded default profile as "single source of truth for all defaults"; built-in `default` synthesized; `ProfileConfig` gains all `Config` fields |
| Per-profile `storage_pool` — "project A on fast NVMe and project B on bulk spinning storage" |
| `[container] alias` |
| Bug fixes: default-profile pointer aliasing; `additional_protected_paths` replaced instead of merged; 6 commands ignoring `--profile` |
| 0.9/0.10: config-shaped CLI flags and all env-var overrides removed in favour of config/profiles |
| `hardened` profile; its secret-mask set broadened; `coi health` proves it at runtime |
| (#607) `[defaults] profile` — pick the profile a bare `coi` uses |
| `[[credentials]]`, `[ports]`, `[[network.hosts]]` become profile-settable, each with its own trust rule |

---

## K. Reproduction notes

- Probes P1–P3 were temporary `*_test.go` files in `internal/config/`, run with
  `go test ./internal/config/ -run TestZZProbe -v`, then deleted. They used `t.TempDir()`
  + `t.Setenv("HOME", ...)` + `os.Chdir` so the real `Load()` ran against a synthetic tree.
- `go test ./internal/config/... ./schema/...` passes clean (Go 1.25.0).
- The wiki was read from a clone of `https://github.com/mensfeld/code-on-incus.wiki.git`.

---

## L. What changed between `afb96e8` and `da67215`

The first pass of this investigation ran against `afb96e8` (2026-07-05), before master was
fast-forwarded 95 commits. Re-running every probe against `da67215` gave:

**Resolved:**

- **`sockets` are now inherited.** `mergeProfiles` gained `Sockets`, `Ports` and
  `Credentials` branches (§C2); on `afb96e8` a child profile silently lost its parent's
  `[[sockets]]`. Confirmed by P1/Q1.

**New profile surface:**

| Change | Detail |
|---|---|
| `[defaults] profile` | new lowest-priority profile selector, trusted-scope only (§H5) |
| `[[credentials]]` | new profile field; ad-hoc entries trust-gated, catalog bundles not |
| `[ports]` | new profile field; section-level untrusted marking (`markUntrustedPorts`) |
| `[[network.hosts]]` | new; refused outright from untrusted scope as a spoofing primitive |
| `[container] ready_timeout` | new |
| `git.name` / `git.email` / `git.seed_host_identity` | new; stripped from untrusted scope |
| `model` relocation | top-level profile `model` **and** `[defaults] model` both removed; the model is now only `[tool.claude] model` |
| default allowlist | `8.8.8.8` / `1.1.1.1` dropped (DNS egress is blocked, so allowlisting resolvers bought nothing); Vertex AI endpoints added as comments |

**Unchanged — re-confirmed on `da67215`:**

- three profile scan dirs (§A1); duplicate-name hard error (§A2)
- `default` is a clone of *resolved* config, not embedded defaults (P1/Q2)
- `hardened` is a fixed baseline; both built-ins guarded by `Source == "(built-in)"` (§H2, H3)
- `ApplyProfile` appends and is non-idempotent (P1/Q5)
- cross-workspace precedence asymmetry (P2, §E)
- schema strictness on profiles vs. silence on `config.toml`; inline `[profiles.x]`
  silently ignored (P3, §F)
- no `-p`/`--profile` ever passed to `incus`; container name excludes the profile;
  `skipLaunch` bypasses image selection on reuse (§G)
