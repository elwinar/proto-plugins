# Proto Plugin TOML Cheatsheet

Complete reference for all available options in proto plugin TOML files with examples and edge cases.

## Root Level

### `name` (Required)
The human-readable name of the plugin.

```toml
name = "Ruff"                    # Display name, can contain spaces
name = "postgresql"              # Typically lowercase for file names
```

**Edge cases:**
- Name is used for logging and display purposes only
- Does not need to match the filename
- Can contain spaces and special characters

### `type` (Required)
Specifies the type of plugin. Currently only `"cli"` is supported.

```toml
type = "cli"                     # Only valid value
```

---

## `[resolve]` Section

Controls how proto discovers and resolves available versions.

### `git-url`
Git repository URL for resolving versions from git tags.

```toml
[resolve]
git-url = "https://github.com/mvdan/gofumpt"
git-url = "https://github.com/astral-sh/ruff"
```

**Edge cases:**
- Must be a valid Git URL
- Cannot be used together with `manifest-url`
- Proto automatically fetches tags from this repository
- Performance: First time fetch may be slow, results are cached

### `manifest-url`
Alternative to `git-url`: URL pointing to a JSON version manifest.

```toml
[resolve]
manifest-url = "https://raw.githubusercontent.com/elwinar/proto-plugins/main/gcloud-versions.json"
```

**Use when:**
- Project doesn't use git tags for releases
- Need custom version discovery logic
- Releases are published through non-standard channels

**Edge cases:**
- Manifest URL must return valid JSON with version list
- Cannot be used together with `git-url`
- You are responsible for maintaining the manifest

### `version-pattern`
Regex pattern to extract version from git tags (only used with `git-url`).

```toml
[resolve]
git-url = "https://github.com/jqlang/jq"
version-pattern = "^jq-((?<major>[0-9]+)\\.(?<minor>[0-9]+)\\.(?<patch>[0-9]+))"

[resolve]
git-url = "https://github.com/biomejs/biome/"
version-pattern = "^@biomejs/biome@((?<major>[0-9]+)\\.(?<minor>[0-9]+)\\.(?<patch>[0-9]+)(?<pre>-[^+]+)?)"
```

**Format:**
- Must be a valid regex with named capture groups: `(?<major>...)`, `(?<minor>...)`, `(?<patch>...)`
- Optional groups: `(?<pre>...)` for pre-release identifiers
- Default pattern (if omitted): `^v?((?<major>[0-9]+)\\.(?<minor>[0-9]+)\\.(?<patch>[0-9]+))`

**Edge cases:**
- Complex version strings (like `@biomejs/biome@1.0.0`) need explicit patterns
- Pre-release versions are optional but useful for filtering
- If pattern doesn't match any tags, version discovery fails
- Patterns are case-sensitive

---

## `[platform.X]` Sections

Platform-specific download configuration. Supports: `linux`, `macos`, `windows`.

### `download-file`
Filename pattern of the release asset to download.

```toml
[platform.linux]
download-file = "gofumpt_v{version}_linux_{arch}"
download-file = "ruff-{arch}-unknown-linux-{libc}.tar.gz"
download-file = "postgresql-{version}-{arch}-unknown-linux-{libc}.tar.gz"

[platform.macos]
download-file = "jq-macos-{arch}"

[platform.windows]
download-file = "biome-win32-{arch}.exe"
download-file = "rclone-v{version}-windows-{arch}.zip"
```

**Template variables:**
- `{version}` - Release version (e.g., "1.2.3")
- `{arch}` - Architecture after mapping (see `[install.arch]`)
- `{libc}` - Libc variant after mapping (see `[install.libc]`)
- `{download_file}` - Can be referenced in `download-url` (not in `download-file` itself)

**Edge cases:**
- File must exist in the release
- Leading directory prefixes in names must match exactly
- Order of variables doesn't matter, but double-check against actual filenames

### `exe-path`
Path to the executable within the downloaded archive (relative to archive root).

```toml
[platform.linux]
exe-path = "ruff-{arch}-unknown-linux-{libc}/ruff"
exe-path = "bin/psql"

[platform.windows]
exe-path = "google-cloud-sdk/bin/gcloud.cmd"
exe-path = "migrate.exe"

[platform.macos]
exe-path = "google-cloud-sdk/bin/gcloud"
```

**Common patterns:**
- Single binary in archive root: `exe-path = "binary-name"`
- Binary in subdirectory: `exe-path = "dir/subdir/binary"`
- Windows executables: `.exe` extension required
- Can use `{arch}` and `{libc}` template variables

**Edge cases:**
- Not required if binary is at archive root with exact name from `download-file`
- If download-file already includes `.exe` extension, don't repeat it in `exe-path`
- For bare binaries (unpack=false), exe-path must match `download-file` exactly

### `checksum-file`
Filename of the checksum file within the release.

```toml
[platform.linux]
checksum-file = "sha256sum.txt"
checksum-file = "ruff-{arch}-unknown-linux-{libc}.tar.gz.sha256"
```

**Common formats:**
- `sha256sum.txt` - Single file with all checksums
- `{download_file}.sha256` - Individual checksum per file
- `checksums.txt` - Same as sha256sum.txt

**Edge cases:**
- Optional: if omitted, checksum verification is skipped
- Format varies by project (grep release page to find correct name)
- Some projects use `.sha256`, others `.sha256sum`

### `archive-prefix`
Subdirectory within the archive that contains the files (strips outer directory).

```toml
[platform.linux]
archive-prefix = "postgresql-{version}-{arch}-unknown-linux-{libc}"

[platform.macos]
archive-prefix = "rclone-v{version}-osx-{arch}"
```

**When to use:**
- Archives have a root folder (e.g., `rclone-v1.5.0-linux-amd64/`) that needs stripping
- Without this, `exe-path` would need to include the prefix

**Edge cases:**
- Only affects unpacked archives (unpack=true)
- Can use `{version}`, `{arch}`, `{libc}` variables
- If archive has inconsistent root structure, this will fail

---

## `[install]` Section

Download and installation configuration.

### `download-url`
Template URL for downloading the release asset.

```toml
[install]
download-url = "https://github.com/mvdan/gofumpt/releases/download/v{version}/{download_file}"
download-url = "https://github.com/jqlang/jq/releases/download/jq-{version}/{download_file}"
download-url = "https://dl.google.com/dl/cloudsdk/channels/rapid/downloads/{download_file}"
download-url = "https://github.com/biomejs/biome/releases/download/%40biomejs/biome%40{version}/{download_file}"
```

**Template variables:**
- `{version}` - Release version
- `{download_file}` - From platform-specific `download-file`

**Edge cases:**
- Some projects use URL encoding: `%40` = `@`, `%2F` = `/`
- Version prefix varies: some use `v{version}`, others `jq-{version}`
- Check the actual GitHub release page to get the pattern right
- Don't assume standard format; many projects have custom naming

### `checksum-url`
Template URL for downloading the checksum file.

```toml
[install]
checksum-url = "https://github.com/astral-sh/ruff/releases/download/{version}/{checksum_file}"
checksum-url = "https://github.com/golang-migrate/migrate/releases/download/v{version}/{checksum_file}"
```

**Template variables:**
- `{version}` - Release version
- `{checksum_file}` - From platform-specific `checksum-file`

**Edge cases:**
- Optional: if omitted, checksum verification is skipped
- Uses `{checksum_file}` not `{download_file}`
- Must match the same base URL structure as `download-url`

### `unpack`
Whether to extract/unpack downloaded archives. Defaults to `true`.

```toml
[install]
unpack = false                   # Binary is pre-compiled, not archived
unpack = true                    # Archive with tar.gz or zip

# Implicit (defaults to true if omitted)
```

**When to use:**
- `unpack = false`: Downloaded file is the executable itself (e.g., pre-compiled binary)
- `unpack = true` (or omitted): Downloaded file is a tar.gz or zip archive

**Edge cases:**
- If `unpack = false`, `exe-path` must match `download-file` exactly
- If archive contains unexpected structure, unpack may fail

---

## `[install.arch]` Section

Maps system architecture to release asset architecture strings.

```toml
[install.arch]
aarch64 = "arm64"               # ARM 64-bit
aarch64 = "arm"                 # Some projects use this
x86_64 = "amd64"                # Intel/AMD 64-bit
x86_64 = "x64"                  # Alternative naming
x86_64 = "x86_64"               # Direct passthrough
x86 = "i686"                    # 32-bit (rarely used)
```

**System → Release mapping:**
- Your system reports: `uname -m` (e.g., `aarch64`, `x86_64`)
- Project uses: their own naming in releases (e.g., `arm64`, `amd64`)

**Common mappings:**
- `aarch64 → arm64` (most common for ARM 64-bit)
- `x86_64 → amd64` (Go projects)
- `x86_64 → x64` (JavaScript projects)

**Edge cases:**
- If architecture not listed, fallback is the system name as-is
- Some projects don't use mapping at all (omit this section)
- Missing mapping causes download failure with unclear error
- Verify against actual release filenames

---

## `[install.libc]` Section

Maps system C library to release asset variants (Linux only).

```toml
[install.libc]
gnu = ""                         # glibc systems (most common)
gnu = "gnu"                      # Alternative: explicit "gnu" suffix
musl = "-musl"                   # musl-based systems (Alpine)
musl = "musl"                    # Alternative: "musl" suffix
```

**When needed:**
- Some projects build separate binaries for glibc vs musl
- Linux systems report libc type; proto detects and maps it

**Common mappings:**
- GNU/glibc (Ubuntu, Debian, Fedora): `gnu = ""`
- Alpine (musl): `musl = "-musl"`
- Some use `gnu = "gnu"` and `musl = "musl"` without special chars

**Edge cases:**
- If omitted, libc is not used in `download-file`
- `gnu = ""` means empty string (common for default variant)
- Wrong mapping causes download failure on non-matching libc
- Some projects only ship glibc binaries (musl users need to build from source)

---

## `[install.exes.*]` Sections

For tools that install multiple executables (e.g., PostgreSQL with psql, createdb, etc.).

```toml
[install.exes.psql]
primary = true                   # Primary/default executable
exe-path = "bin/psql"

[install.exes.createdb]
exe-path = "bin/createdb"

[install.exes.pg_dump]
exe-path = "bin/pg_dump"
```

**Common use cases:**
- PostgreSQL (psql, createdb, pg_dump, etc.)
- Sets of related CLI tools (multiple binaries)

**`primary` field:**
- Only one `exes.*` entry should have `primary = true`
- The primary executable is available as the tool's main command
- Other executables are available as separate commands

**Edge cases:**
- If only one executable, can use root-level `exe-path` instead (simpler)
- All entries must have valid `exe-path`
- Names (after `exes.`) become the command names in the shell

---

## `[detect]` Section

Configuration for version auto-detection from project files.

### `version-files`
List of filenames that may contain version constraints.

```toml
[detect]
version-files = [".tool-versions", ".proto-version"]
version-files = [".shfmt-version"]
version-files = [".nvmrc", ".node-version"]
```

**Format:**
- Array of filenames to check in order
- First matching file is used
- Files are searched in the project directory

**Edge cases:**
- Files must contain a version string (e.g., `1.2.3` or `v1.2.3`)
- Format varies; proto tries to parse common patterns
- If multiple files exist, first one wins (order matters)
- Requires correct `version-pattern` in `[resolve]` to work properly

---

## Complete Examples

### Minimal Example (Simple Release)
```toml
name = "jq"
type = "cli"

[resolve]
git-url = "https://github.com/jqlang/jq"

[platform.linux]
download-file = "jq-linux-{arch}"

[platform.macos]
download-file = "jq-macos-{arch}"

[platform.windows]
download-file = "jq-windows-{arch}.exe"

[install]
download-url = "https://github.com/jqlang/jq/releases/download/jq-{version}/{download_file}"
unpack = false

[install.arch]
aarch64 = "arm64"
x86_64 = "amd64"
```

### Complex Example (Custom Version Pattern, Multiple Architectures)
```toml
name = "Ruff"
type = "cli"

[resolve]
git-url = "https://github.com/astral-sh/ruff"

[platform.linux]
exe-path = "ruff-{arch}-unknown-linux-{libc}/ruff"
download-file = "ruff-{arch}-unknown-linux-{libc}.tar.gz"
checksum-file = "ruff-{arch}-unknown-linux-{libc}.tar.gz.sha256"

[platform.macos]
exe-path = "ruff-{arch}-apple-darwin/ruff"
download-file = "ruff-{arch}-apple-darwin.tar.gz"
checksum-file = "ruff-{arch}-apple-darwin.tar.gz.sha256"

[platform.windows]
exe-path = "ruff-{arch}-pc-windows-msvc/ruff.exe"
download-file = "ruff-{arch}-pc-windows-msvc.zip"
checksum-file = "ruff-{arch}-pc-windows-msvc.zip.sha256"

[install]
download-url = "https://github.com/astral-sh/ruff/releases/download/{version}/{download_file}"
checksum-url = "https://github.com/astral-sh/ruff/releases/download/{version}/{checksum_file}"

[install.arch]
x86 = "i686"
# aarch64 and x86_64 use defaults

[install.libc]
gnu = ""
# musl uses default (not provided)
```

### Multiple Executables Example
```toml
name = "postgresql"
type = "cli"

[resolve]
git-url = "https://github.com/theseus-rs/postgresql-binaries"

[platform.linux]
download-file = "postgresql-{version}-{arch}-unknown-linux-{libc}.tar.gz"
checksum-file = "postgresql-{version}-{arch}-unknown-linux-{libc}.tar.gz.sha256"
archive-prefix = "postgresql-{version}-{arch}-unknown-linux-{libc}"

[platform.macos]
download-file = "postgresql-{version}-{arch}-apple-darwin.tar.gz"
checksum-file = "postgresql-{version}-{arch}-apple-darwin.tar.gz.sha256"
archive-prefix = "postgresql-{version}-{arch}-apple-darwin"

[platform.windows]
download-file = "postgresql-{version}-{arch}-pc-windows-msvc.zip"
checksum-file = "postgresql-{version}-{arch}-pc-windows-msvc.zip.sha256"
archive-prefix = "postgresql-{version}-{arch}-pc-windows-msvc"

[install]
download-url = "https://github.com/theseus-rs/postgresql-binaries/releases/download/{version}/{download_file}"
checksum-url = "https://github.com/theseus-rs/postgresql-binaries/releases/download/{version}/{checksum_file}"

[install.arch]
aarch64 = "arm64"
x86_64 = "amd64"

[install.libc]
gnu = ""
musl = "-musl"

[install.exes.psql]
primary = true
exe-path = "bin/psql"

[install.exes.createdb]
exe-path = "bin/createdb"

[install.exes.pg_dump]
exe-path = "bin/pg_dump"
```

---

## Common Troubleshooting

### "File not found" during download
- **Cause:** `download-file` name doesn't match actual release filename
- **Fix:** Check GitHub releases page, verify exact naming and extensions

### "Version pattern failed to match"
- **Cause:** Git tags don't match `version-pattern` regex
- **Fix:** Check git tags (`git ls-remote --tags`), adjust pattern accordingly

### Wrong binary architecture downloaded
- **Cause:** `[install.arch]` mapping is incorrect
- **Fix:** Check what arch name the project uses in filenames, update mapping

### "Cannot unpack archive"
- **Cause:** `archive-prefix` doesn't match actual archive root directory
- **Fix:** Extract archive manually, check root directory name, update prefix

### Binary not found after install
- **Cause:** `exe-path` doesn't match actual location in archive
- **Fix:** Extract archive, find binary, update `exe-path` to match

---

## Release Tool & Language Templates

Quick templates for projects using popular release tools and language conventions.

### GoReleaser (Go Projects)

GoReleaser is extremely popular in Go projects. Typical naming pattern:

```toml
name = "tool-name"
type = "cli"

[resolve]
git-url = "https://github.com/owner/repo"

[platform.linux]
download-file = "tool-name_{version}_linux_{arch}.tar.gz"
# OR for different naming:
download-file = "tool-name-{version}-linux-{arch}.tar.gz"

[platform.macos]
download-file = "tool-name_{version}_darwin_{arch}.tar.gz"

[platform.windows]
download-file = "tool-name_{version}_windows_{arch}.zip"

[install]
download-url = "https://github.com/owner/repo/releases/download/v{version}/{download_file}"

[install.arch]
aarch64 = "arm64"
x86_64 = "amd64"
```

**Characteristics:**
- Uses `v{version}` prefix in release tags (e.g., `v1.2.3`)
- Consistent underscore separators: `tool_os_arch`
- Checksums usually in separate `.sha256sum` file
- Often publishes checksums file: add `checksums.txt`

**Examples:** gofumpt, migrate, rclone (via GoReleaser)

---

### Rust Projects (cargo-release or cross)

Rust projects often cross-compile with consistent naming:

```toml
name = "tool-name"
type = "cli"

[resolve]
git-url = "https://github.com/owner/repo"

[platform.linux]
download-file = "tool-name-{arch}-unknown-linux-{libc}.tar.gz"
checksum-file = "tool-name-{arch}-unknown-linux-{libc}.tar.gz.sha256"

[platform.macos]
download-file = "tool-name-{arch}-apple-darwin.tar.gz"
checksum-file = "tool-name-{arch}-apple-darwin.tar.gz.sha256"

[platform.windows]
download-file = "tool-name-{arch}-pc-windows-msvc.zip"
checksum-file = "tool-name-{arch}-pc-windows-msvc.zip.sha256"

[install]
download-url = "https://github.com/owner/repo/releases/download/v{version}/{download_file}"
checksum-url = "https://github.com/owner/repo/releases/download/v{version}/{checksum_file}"

[install.arch]
aarch64 = "aarch64"
x86_64 = "x86_64"
x86 = "i686"

[install.libc]
gnu = ""
musl = "-musl"
```

**Characteristics:**
- Verbose triple naming: `{arch}-{vendor}-{os}-{abi}`
- Separate checksum per binary (not one master file)
- Uses `v{version}` tags
- Explicit libc variants

**Examples:** ruff, biome, ripgrep

---

### Node.js/JavaScript Projects

JavaScript tools often use different naming, especially on npm:

```toml
name = "tool-name"
type = "cli"

[resolve]
git-url = "https://github.com/owner/repo"
# Some JS projects use git tags without 'v' prefix
# Adjust version-pattern if needed:
# version-pattern = "^((?<major>[0-9]+)\\.(?<minor>[0-9]+)\\.(?<patch>[0-9]+))"

[platform.linux]
download-file = "tool-name-{version}-linux-{arch}.tar.gz"

[platform.macos]
download-file = "tool-name-{version}-macos-{arch}.tar.gz"

[platform.windows]
download-file = "tool-name-{version}-windows-{arch}.zip"

[install]
download-url = "https://github.com/owner/repo/releases/download/{version}/{download_file}"

[install.arch]
aarch64 = "arm64"
x86_64 = "x64"
```

**Characteristics:**
- May not use `v` prefix (e.g., `1.2.3` instead of `v1.2.3`)
- Uses `macos` not `darwin` in filenames
- Uses `x64` not `amd64` for x86_64
- Often simpler naming, fewer variants

**Examples:** biome, esbuild

---

### Python Projects

Python CLI tools are less standardized but follow some patterns:

```toml
name = "tool-name"
type = "cli"

[resolve]
git-url = "https://github.com/owner/repo"

[platform.linux]
download-file = "tool-name-{version}-linux-{arch}"
# OR: tool-name-{version}-Linux-{arch}

[platform.macos]
download-file = "tool-name-{version}-macos-{arch}"

[platform.windows]
download-file = "tool-name-{version}-windows-{arch}.exe"

[install]
download-url = "https://github.com/owner/repo/releases/download/v{version}/{download_file}"
unpack = false

[install.arch]
aarch64 = "aarch64"
x86_64 = "x86_64"
```

**Characteristics:**
- Often ships bare executables (unpack=false) not archives
- Architecture names less standardized
- May use `{version}` without `v` prefix

**Examples:** ruff (also Rust), black (if compiled)

---

### Generic GitHub Release Convention

Fallback template for projects without a standard release tool:

```toml
name = "tool-name"
type = "cli"

[resolve]
git-url = "https://github.com/owner/repo"

[platform.linux]
download-file = "name-{version}-linux-{arch}.tar.gz"

[platform.macos]
download-file = "name-{version}-macos-{arch}.tar.gz"

[platform.windows]
download-file = "name-{version}-windows-{arch}.zip"

[install]
download-url = "https://github.com/owner/repo/releases/download/v{version}/{download_file}"

[install.arch]
aarch64 = "arm64"
x86_64 = "amd64"
```

---

### Projects with Non-Standard Releases

Some projects don't follow conventions and require custom configurations:

#### Projects using scoped npm packages (e.g., @owner/tool)
```toml
[resolve]
version-pattern = "^@owner/tool@((?<major>[0-9]+)\\.(?<minor>[0-9]+)\\.(?<patch>[0-9]+))"

[install]
# URL encoding required: %40 = @
download-url = "https://github.com/owner/repo/releases/download/%40owner/tool%40{version}/{download_file}"
```

**Examples:** biome (@biomejs/biome)

#### Projects using prebuilt binaries from CDN
```toml
[resolve]
# Use manifest instead of git tags
manifest-url = "https://example.com/releases/versions.json"

[install]
# Direct CDN download, not GitHub releases
download-url = "https://cdn.example.com/releases/{version}/{download_file}"
```

**Examples:** Google Cloud SDK, Node.js

#### Projects with version tags without 'v' prefix
```toml
[resolve]
git-url = "https://github.com/owner/repo"
version-pattern = "^((?<major>[0-9]+)\\.(?<minor>[0-9]+)\\.(?<patch>[0-9]+))"

[install]
# No 'v' prefix in download URL
download-url = "https://github.com/owner/repo/releases/download/{version}/{download_file}"
```

---

## Quick Decision Tree

When creating a new plugin configuration, follow this decision tree:

1. **How are versions published?**
   - Git tags (most common) → Use `[resolve] git-url`
   - Custom manifest → Use `[resolve] manifest-url`

2. **What's the version tag format?**
   - Standard `v1.2.3` → Use default or omit `version-pattern`
   - Non-standard (e.g., `@scope/name@1.2.3`) → Provide custom `version-pattern`

3. **What's the release asset naming?**
   - Check 2-3 recent releases on GitHub
   - Look for patterns in: `{arch}`, `{os}`, `{version}` placement
   - Use template variables to normalize

4. **Is it a single binary or archive?**
   - Single binary → `unpack = false`
   - Tar/zip archive → `unpack = true` (default)

5. **Which architectures are supported?**
   - Check available binaries in latest release
   - Create `[install.arch]` mappings for mismatch only
   - If no mismatch, omit the section

6. **Are checksums available?**
   - Check for `.sha256`, `checksums.txt`, etc.
   - Add `checksum-file` and `checksum-url` if available
   - Optional but recommended for security

---

## Design Principles

1. **Template variables are power tools:** Use `{version}`, `{arch}`, `{libc}` to handle variations
2. **Validate against reality:** Always check actual GitHub releases and archive contents
3. **Platform matters:** Linux, macOS, Windows often have different naming conventions
4. **Checksum is optional but recommended:** Adds security and catch corrupted downloads
5. **Version detection:** Custom patterns needed when projects don't follow semantic versioning
