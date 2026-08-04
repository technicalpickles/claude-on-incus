# COI Profiles — Evidence Log

Raw evidence gathered while answering "what are profiles, functionally, in relation to
Incus, and semantically". Companion to [`profiles-findings.md`](profiles-findings.md),
which contains the conclusions. This file is the audit trail: every claim there should be
traceable to a citation or probe output here.

**Investigated checkout:** `technicalpickles/code-on-incus` @ `afb96e8`
(master, 2026-07-05).
**Method:** two passes. Pass 1 read the code, schema, README and wiki. Pass 2 re-derived
every behavioural claim by running the real loader against synthetic profile trees
(probes P1–P3 below), and cross-checked docs against upstream.

**Known limitation:** the local clone is shallow (324 commits, `.git/shallow` present), so
pre-2026 history is unavailable. Design-intent chronology below is sourced from
`CHANGELOG.md`, not from `git log`.

---

## A. Where profiles live and how they are loaded

### A1. Scan locations — three, not two

```go
// internal/config/config.go:608
func GetProfileParentDirs() []string {
	dirs := []string{
		filepath.Join(homeDir, ".coi"), // 1. User home
		filepath.Join(workDir, ".coi"), // 2. Project
	}
	if envConfig := os.Getenv("COI_CONFIG"); envConfig != "" {
		dirs = append(dirs, filepath.Dir(envConfig))   // 3.
	}
	return dirs
}
```

Each parent dir is scanned for a `profiles/` subdirectory; each subdirectory containing a
`config.toml` becomes a profile named after the directory (`internal/config/loader.go:342-431`).

### A2. Single flat namespace, duplicates are a hard error

```go
// internal/config/loader.go:364
if existing, ok := cfg.Profiles[profileName]; ok && existing.Source != "" && existing.Source != profileConfigPath {
	return fmt.Errorf(
		"profile %q defined in multiple locations:\n  %s\n  %s\n"+
			"Rename one of them or delete the duplicate so it's clear which profile is being used",
		profileName, existing.Source, profileConfigPath)
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

Note step 4 runs **after** step 3 — so the synthesized `default` profile is a snapshot of
the config **after** user and project config were merged in. Verified empirically in P1/Q3.

### A4. Schema validation is applied to profile files only

```go
// internal/config/loader.go:385-392
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
// internal/config/config.go:258-283
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
	Network     *NetworkConfig    `toml:"network"`
	ForwardEnv  []string          `toml:"forward_env"`
	Source      string            `toml:"-"`

	// Extended fields — previously Config-only, now available in profiles
	Model      string            `toml:"model"`
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

### B1. Struct ↔ schema are in sync in this checkout

```
$ python3 - <<'PY'   # compares ProfileConfig toml tags to schema/profile.schema.json properties
in schema not struct: []
in struct not schema: []
PY
```

Both sides: `container, context, env_commands, environment, forward_env, git, incus,
inherits, limits, model, monitoring, mounts, network, paths, security, shell, sockets,
ssh, timezone, tool`.

### B2. Profile shape vs. global `Config` shape — deliberate deltas

`Config` (`internal/config/config.go:26-44`) has `[defaults]`, `[detection]`, and
`[mounts.default]`; `ProfileConfig` has none of those. Instead:

| Global config | Profile equivalent |
|---|---|
| `[defaults] model` | top-level `model` |
| `[defaults] environment` / `forward_env` / `env_commands` | top-level `[environment]` / `forward_env` / `[env_commands]` |
| `[defaults] env_command_timeout` | **no profile equivalent** |
| `[[mounts.default]]` | `[[mounts]]` |
| `[detection]` | **no profile equivalent** |

### B3. `ClaudeToolConfig` in this checkout has only `effort_level`

```go
// internal/config/config.go:296-298
type ClaudeToolConfig struct {
	EffortLevel string `toml:"effort_level"`
}
```

---

## C. Two different merge algorithms

### C1. `ApplyProfile` — override scalars, **append** slices

```go
// internal/config/config.go:860-949 (abridged)
if len(profile.ForwardEnv) > 0 { c.Defaults.ForwardEnv = MergeStringSliceUnique(...) }
if len(profile.Mounts)  > 0 { c.Mounts.Default = append(c.Mounts.Default, profile.Mounts...) }
if len(profile.Sockets) > 0 { c.Sockets       = append(c.Sockets, profile.Sockets...) }
```

Because the slice fields append, `ApplyProfile` is **not idempotent** — which is exactly
why `ReapplyProfileContainer` exists:

```go
// internal/config/config.go:951-960
// Unlike a full ApplyProfile, this is idempotent:
// mergeContainerInto/mergeBuildInto are field-level overrides with no
// appends, so it is safe to call after a profile was already applied
// (a full re-apply would duplicate profile mounts and sockets).
```

### C2. `mergeProfiles` (inheritance) — override scalars, **replace** slices

```go
// internal/config/config.go:1036-1042
// Arrays: if child defines them, they fully replace parent's. If not, inherit.
if result.Mounts == nil     { result.Mounts = parent.Mounts }
if result.ForwardEnv == nil { result.ForwardEnv = parent.ForwardEnv }
```

`result.Sockets` is **absent from this block** — grep for `Sockets` in
`internal/config/config.go` returns only lines 34, 268, 496, 709-710, 908-909; none of
them are inside `mergeProfiles`.

### C3. Inheritance resolution

`ResolveProfileInheritance` (`internal/config/config.go:1252`): first pass walks each
chain to detect cycles and enforce `maxInheritanceDepth = 10` (line 1001); second pass
merges parent into child and flattens. `merged.Inherits = parentName` is preserved for
display only (line 1308).

---

## D. Probe P1 — inheritance, built-ins, untrusted sanitization

A temporary `internal/config/zz_probe_test.go` built a synthetic tree and called the real
`Load()`. Setup: a trusted `~/.coi/config.toml` with `model = "user-global-model"`,
`mode = "open"` and one `[[sockets]]`; a user profile `base` with mounts/sockets/env; a
user profile `child` with `inherits = "base"` and nothing array-shaped; an untrusted
project profile `evil` attempting several downgrades.

```
=== PROFILE NAMESPACE ===
  base       source=<tmp>/001/.coi/profiles/base/config.toml   inherits=""
  child      source=<tmp>/001/.coi/profiles/child/config.toml  inherits="base"
  evil       source=<tmp>/002/.coi/profiles/evil/config.toml   inherits=""
  default    source=(built-in)                                 inherits=""
  hardened   source=(built-in)                                 inherits=""

=== Q1: does inheritance carry SOCKETS? ===
  base.Sockets  = [{Host:/parent.sock Container:/tmp/parent.sock Env: ...}]
  child.Sockets = []            <-- NOT inherited

=== Q2: does inheritance carry mounts / forward_env / env? ===
  child.Mounts      = [{Host:/parent/mount Container:/parent ...}]     <-- inherited
  child.ForwardEnv  = [PARENT_VAR]                                     <-- inherited
  child.Environment = map[FROM_PARENT:1]     <-- KEEP_ME cleared by ""

=== Q3: is built-in `default` a clone of RESOLVED config? ===
  default.Model        = "user-global-model"
  default.Network.Mode = "open"          (embedded default is "restricted")
  default.Sockets      = [{Host:~/user.sock Container:/tmp/user.sock ...}]

=== Q4: hardened profile ===
  hardened.Source="(built-in)" inherits=""
  hardened.Model=""                      (empty = falls through)
  hardened.Network.Mode="restricted"
  hardened.Container.Persistent=false

=== Q5: untrusted project profile sanitization ===
  evil.Network.Mode=""                        (open stripped)
  evil.Network.BlockPrivateNetworks=<nil>     (stripped)
  evil.Security.DisableProtection=false       (stripped)
  evil.Git.WritableHooks=<nil>                (stripped)
  evil.EnvCommands=map[]                      (stripped)
  evil.Mounts[0].Untrusted=true SourcePath="<tmp>/002/.coi/profiles/evil/config.toml"
                                              (mount SURVIVES, gated at apply time)

=== Q6: ApplyProfile — additive vs replace on mounts ===
  cfg.Mounts.Default before=0  after ApplyProfile(base)=1
  after applying base TWICE = 2               <-- non-idempotent

=== Q7: hardened over an `open` global network ===
  cfg2.Network.Mode before = "open"
  after ApplyProfile(hardened) = "restricted", forward_agent=false,
                                 persistent=false, monitoring=true
  secret_paths count = 16
```

Warnings emitted to stderr during the untrusted load (verbatim):

```
WARNING: ignoring security-downgrading "network.block_private_networks=false" in project config .../evil/config.toml; move it to ~/.coi/config.toml or set COI_CONFIG to apply it.
WARNING: ignoring security-downgrading "network.mode=open" in project config .../evil/config.toml; ...
WARNING: ignoring 'security.disable_protection' in project config .../evil/config.toml; removing read-only protection is a security downgrade. ...
WARNING: ignoring 'git.writable_hooks' in project config .../evil/config.toml; ...
WARNING: ignoring 'env_commands' in project profile .../evil/config.toml; running a host command is host code execution. Move it to a profile under ~/.coi/profiles to apply it.
```

Corresponding code: `loadProfileDirectories`'s untrusted branch,
`internal/config/loader.go:410-423`.

---

## E. Probe P2 — profile vs. workspace-overlay precedence

Setup: trusted user profile `tight` (`image = "profile-image"`, `mode = "allowlist"`,
`allowed_domains = ["only-this.example"]`, `memory.limit = "1GiB"`,
`permission_mode = "interactive"`). A *different* workspace ships an untrusted
`.coi/config.toml` widening all four.

```
=== after ApplyProfile(tight) ===
  image="profile-image" mode="allowlist" domains=[only-this.example] mem="1GiB"  perm="interactive"

=== after OverlayProjectConfig(otherWS) ===
  image="repo-image"    mode="allowlist" domains=[anything.example evil.example] mem="64GiB" perm="bypass"

=== after ReapplyProfileContainer(tight) ===
  image="profile-image" mode="allowlist" domains=[anything.example evil.example] mem="64GiB" perm="bypass"

=== in-workspace: Load() then ApplyProfile(tight) ===
  before: image="repo-image"    domains=[anything.example evil.example] mem="64GiB" perm="bypass"
  after : image="profile-image" mode="allowlist" domains=[only-this.example] mem="1GiB" perm="interactive"
```

This mirrors the real cross-workspace code path:

```go
// internal/cli/phases_shell.go:80-106 (alias path, abridged)
if s.aliasArg != "" {
	resolved, _ := alias.ResolveAliasForLaunch(s.aliasArg)
	a.cfg.OverlayProjectConfig(resolved.Workspace)      // untrusted repo config, AFTER the profile
	if resolved.Profile != "" && !cmd.Flags().Changed("profile") { ... }
}
if cmd.Flags().Changed("profile") && a.profile != "" {
	a.cfg.ReapplyProfileContainer(a.profile)            // re-asserts [container] ONLY
}
```

Same shape in `internal/cli/run.go:380-406` (`overlayWorkspaceConfig`, reached when
`--workspace` differs from cwd).

`mergeNetworkInto` replaces the domain list wholesale rather than intersecting:

```go
// internal/config/config.go:1135
if src.AllowedDomains != nil { dst.AllowedDomains = src.AllowedDomains }
```

Sanitization of the untrusted overlay (`sanitizeUntrustedNetwork`,
`internal/config/loader.go:259-284`) only rejects `mode=open`,
`block_private_networks=false`, `block_metadata_endpoint=false`, and
`allow_local_network_access=true`. It does not touch `allowed_domains`, `[limits]`, or
`[tool] permission_mode`.

---

## F. Probe P3 — validation asymmetry

```
main-config typos/inline-profiles: err=<nil>
  inline [profiles.legacy] became a profile? false   (silently ignored)
  network.mode="restricted"                          (typo 'mdoe' silently ignored)

profile-dir typo: err=profile "typo" at .../profiles/typo/config.toml failed schema
  validation: - at '/network': additional properties 'mdoe' not allowed
```

Relevant declarations:

```go
// internal/config/config.go:42
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
// internal/container/commands.go:390-398
func initAndConfigureContainer(imageAlias, containerName, pool string, ephemeral bool, preStart func() error) error {
	args := []string{"init", imageAlias, containerName}
	if ephemeral { args = append(args, "--ephemeral") }
	if pool != "" { args = append(args, "-s", pool) }
	if err := IncusExec(args...); err != nil { return err }
```

No `-p` / `--profile` is ever passed. The only global flag COI injects is the project:

```go
// internal/container/commands.go:706
incusArgs := append([]string{"--project", IncusProject}, args...)
```

### G2. Every reference to an *Incus* profile is read-only inspection of Incus's `default`

`grep -rin "incus profile"` over `*.go`, `*.md`, `*.sh`, `*.py`:

| Site | Purpose |
|---|---|
| `internal/container/security.go:28-38` | refuse to launch if `security.privileged=true` on the default Incus profile |
| `internal/health/checks.go:2604` (`CheckPrivilegedProfile`) | same, as a CRITICAL health check |
| `internal/health/checks.go:2646` | seccomp / AppArmor posture on the default Incus profile |
| `internal/network/nft_filter.go:757` (`GetIncusBridgeName`) | `incus profile device show default` → bridge name for nft rules |
| `install.sh:521` | `incus profile device set default root pool=zfs-pool` — the one *write*, at install time, not per-session |

### G3. COI profile fields → per-instance Incus state

| Profile field | Mechanism |
|---|---|
| `[container] image` | `incus init <image> <name>` (`commands.go:391`) |
| `[container] persistent = false` | `--ephemeral` (`commands.go:393`) |
| `[container] storage_pool` | `-s <pool>` (`commands.go:396`); validated by `container.ValidateStoragePool` (`storage.go:39`) |
| `[incus] project` | `--project` on every call (`commands.go:706`) |
| `[limits.*]` | `incus config set` (`commands.go:731`, `internal/limits/applier.go`) |
| `[[mounts]]` | `incus config device add <name> <dev> disk ...` (`manager.go:100`) |
| `[[sockets]]`, `[ssh] forward_agent` | `incus config device add <name> <dev> proxy ...` (`manager.go:118`) |
| `[limits.disk] tmpfs_size` | `incus config device override <name> tmp disk` (`manager.go:138`) |
| `[network]` | host-side nft/iptables rules, not Incus config (`internal/network/`) |

Baseline hardening is applied per instance regardless of profile:
`security.nesting`, `security.syscalls.intercept.mknod`/`setxattr`,
`security.guestapi=false`, `security.idmap.isolated=true`
(`internal/container/commands.go:443-485`).

### G4. Profile is not part of container identity

```go
// internal/session/naming.go:43-47
func ContainerName(workspacePath string, slot int) string {
	hash := WorkspaceHash(workspacePath)
	prefix := GetContainerPrefix()
	return fmt.Sprintf("%s%s-%d", prefix, hash, slot)
}
```

And reuse short-circuits the launch entirely:

```go
// internal/session/setup.go:210-217
if opts.Persistent || opts.ContainerName != "" || opts.ResumeFromID != "" {
	opts.Logger("Container already running, reusing...")
	skipLaunch = true
}
// :222-229  stopped + persistent → Manager.Start(), also no re-launch
```

`skipLaunch` bypasses the code that would consume `[container] image`.

### G5. Profile name *is* persisted per session and per alias

```go
// internal/session/cleanup.go:23
ProfileName string // Profile used for this session (saved in metadata for --resume)
// internal/session/cleanup.go:245
ProfileName string `json:"profile_name"`
```

```go
// internal/cli/phases_shell.go:212-219
if !cmd.Flags().Changed("profile") && metadata.ProfileName != "" {
	a.profile = metadata.ProfileName
	a.cfg.ApplyProfile(a.profile)
	fmt.Fprintf(os.Stderr, "Inherited profile '%s' from session\n", a.profile)
}
```

Alias-saved profile: `internal/cli/phases_shell.go:84-89`.

---

## H. The two built-in profiles

### H1. `default` — a live clone of resolved config

```go
// internal/config/config.go:460-507 (abridged)
// synthesizeDefaultProfile creates a ProfileConfig from the loaded Config,
// representing the "default" built-in profile.
func synthesizeDefaultProfile(cfg *Config) ProfileConfig {
	...
	p := ProfileConfig{
		Container: container, Model: cfg.Defaults.Model,
		Environment: cloneMap(cfg.Defaults.Environment), ...
		Source: "(built-in)",
	}
```

Everything is copied by value / cloned — a past bug (`CHANGELOG.md:426`) was pointer
aliasing letting profile operations mutate the global config.

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
// internal/config/config.go:523-568 (abridged)
// Unlike the "default" profile this is a FIXED baseline, not a clone of the
// user's resolved config: it sets only the hardened overrides and lets every
// other field fall through. It can be overridden by a same-named disk profile.
//
// Limitation: profile merges are additive for slice fields, so it cannot
// *subtract* a globally-configured forward_env, nor force protections back on if
// the user globally set disable_protection=true.
func synthesizeHardenedProfile() ProfileConfig {
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
}
```

`HardenedProfileSecretPaths` (`internal/config/config.go:511-521`) has 16 entries.

### H3. Both built-ins are equally protected from `edit`/`delete`

```go
// internal/cli/profile.go:634-636 (edit) and 695-697 (delete)
if p.Source == "(built-in)" {
	return fmt.Errorf("cannot edit built-in profile 'default'")
}
```

The guard is on `Source`, so it also fires for `hardened` — but the message names only
`default`. P1 confirms `hardened.Source == "(built-in)"`.

### H4. `coi profile create default` is special-cased away from profiles entirely

```go
// internal/cli/profile.go:505-509
// "default" is the built-in profile; it has no profiles/ directory — its
// values are backed by the main config. So scaffold the main config instead.
if name == "default" {
	return a.scaffoldMainConfig(cmd)
}
```

`scaffoldMainConfig` writes `config.EmbeddedStarterConfig` to `~/.coi/config.toml`
(or `./.coi/config.toml` with `--project`), refusing to overwrite an existing file
(`internal/cli/profile.go:558-609`).

---

## I. Documentation cross-reference

### I1. The wiki describes a *newer* codebase, not a stale one

```
$ git remote -v
origin  https://github.com/technicalpickles/code-on-incus
$ git log -1 --format='%H %ad' master
afb96e89... Sun Jul 5 21:08:49 2026
$ git log -1 --oneline upstream/master
da67215 docs(changelog): add the missing #673 double-v version fix entry (#677)
$ git rev-list --count master..upstream/master
95
```

Wiki clone (`code-on-incus.wiki.git`) head: `ea3c7a9` "Correct release version to 0.11.0",
Wed Jul 29 2026 — i.e. authored ~3 weeks after this checkout's master.

Local `CHANGELOG.md` head reads `## 0.10.0 (Unreleased)`.

### I2. Four wiki-documented profile features absent here, present upstream

| Feature | This checkout | `upstream/master` |
|---|---|---|
| `[defaults] profile` (default profile for a bare `coi`) | absent — `DefaultsConfig` (`config.go:190-204`) has only `Model`, `ForwardEnv`, `Environment`, `EnvCommands`, `EnvCommandTimeout`; `git log --all -S "defaults] profile"` finds nothing | `internal/config/config.go:277`: `Profile string \`toml:"profile"\`` |
| `[[credentials]]` in profiles | absent — no `credentials` key in `ProfileConfig` or `schema/profile.schema.json` | present in `internal/config/config.go`, `loader.go`, `schema/profile.schema.json`, plus `credential_entry_test.go` |
| `[tool.claude] model` → `ANTHROPIC_MODEL` | absent — `ClaudeToolConfig` has only `EffortLevel` | `internal/config/config.go:397`: `Model string \`toml:"model"\` // Claude model, delivered as ANTHROPIC_MODEL` |
| `[container] ready_timeout` | absent — `ContainerConfig` (`config.go:64-72`) has no such field; `grep -rn ready_timeout .` → no hits | `internal/config/config.go:103` + `schema/defs/ContainerConfig.json` |

### I3. Wiki claims that are wrong *even against upstream*

- "Profile directories are scanned at two config levels" + a 2-row table. Code has three
  (§A1) — `dirname($COI_CONFIG)` is missing.
- "The built-in `default` profile is always present and reflects the embedded default
  configuration." It reflects the **resolved** config, embedded defaults + user config +
  project config (§A3, P1/Q3). The wiki's own `[defaults] profile` section contradicts
  this two screens later, calling it "a clean clone of your global config".
- Merge-strategy bullet: "Arrays (`mounts`, `forward_env`) fully replace if the child
  defines them" — accurate but incomplete; `sockets` is neither replaced nor inherited
  (§C2, P1/Q1).
- Field table omits `sockets`, `shell`, `env_commands`, and all of
  `security.{secret_paths, writable_paths, additional_protected_paths, disable_protection}`.
- "The built-in `default` profile cannot be edited or deleted" — true of `hardened` too (§H3).
- The `coi profile list` sample output has no `INHERITS` column; the code renders
  `NAME, IMAGE, PERSISTENT, INHERITS, SOURCE` (`internal/cli/profile.go:90`).
- The Profiles page never mentions that project-scoped profiles are **untrusted** and
  sanitized (§D/Q5) — a load-bearing security property, documented only in code comments
  and in `CHANGELOG.md:119,121`.

### I4. README, checked against code

`README.md:434-438` states the hierarchy as `defaults < user config < project config <
profile`. True for the in-workspace path; false for the alias / `--workspace` path (§E).

`README.md:467` — "Each profile is a self-contained directory (`.coi/profiles/<name>/`)
bundling a `config.toml` plus optional build script and context file" — matches
`loadProfileDirectories` + `resolveRelativePath` (`loader.go:394-401`).

`README.md:471-478` on `hardened` — matches `synthesizeHardenedProfile` exactly.

---

## J. Design-intent chronology (from `CHANGELOG.md`, newest last)

| Line | Entry |
|---|---|
| 973 | "Named profiles with environment override support" |
| 949 | "TOML-based configuration system with profile support" |
| 793 | "Per-profile domain allowlists for different security contexts" |
| 574 | Limits configurable "via TOML config file, profile directories, or CLI flags"; precedence then was *CLI flags > profile limits > config file* |
| 391 | (#114) "Self-contained profile directories" — profiles become directories under `profiles/`; "Precedence: user dirs < project dirs < CLI flags" |
| 466 | Scan locations narrowed to `~/.coi/profiles/` + `./.coi/profiles/`, single namespace, duplicate = refuse to start |
| 350 | Dropped `/etc/coi/` and `~/.config/coi/` |
| 389 | Profile inheritance (`inherits`), 10-level chains, cycle detection, INHERITS column |
| 449 | `profile create` / `edit` / `delete` |
| 460 | `profile show` → `profile info` (hidden alias kept) |
| 352-363 | 0.8.0: `[container]` section unifies profile and global shape; legacy layouts refused with migration errors |
| 385 | Embedded default profile as "single source of truth for all defaults"; built-in `default` synthesized; `ProfileConfig` gains all `Config` fields |
| 373 | Per-profile `storage_pool` — "project A on fast NVMe and project B on bulk spinning storage" |
| 371 | `[container] alias` |
| 426-427 | Bug fixes: default-profile pointer aliasing; `additional_protected_paths` replaced instead of merged during inheritance |
| 431 | Bug fix: 6 commands ignored `--profile` by reloading config independently |
| 329-335 | 0.9/0.10: config-shaped CLI flags removed in favour of config/profiles |
| head | 0.10: `--image`, `--persistent`, `--tmux`, `--tool`, `--compression`, `--timeout` removed; "everything config-shaped goes via config/profiles, not flags" |
| 42, 44 | `hardened` profile's secret-mask set broadened; `coi health` proves it at runtime |

---

## K. Reproduction notes

- Probes P1–P3 were temporary `*_test.go` files in `internal/config/`, run with
  `go test ./internal/config/ -run TestZZProbeN -v`, then deleted. They used `t.TempDir()`
  + `t.Setenv("HOME", ...)` + `os.Chdir` so the real `Load()` ran against a synthetic tree.
- `go test ./internal/config/... ./schema/...` passes clean on `afb96e8` (Go 1.25.0).
- The wiki was read from a clone of `https://github.com/mensfeld/code-on-incus.wiki.git`.
