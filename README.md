# tool-action

GitHub Actions for building, packaging, and publishing MCPB bundles.

## Actions

| Action | Description |
|--------|-------------|
| [`setup`](#setup) | Install tool-cli |
| [`validate`](#validate) | Validate manifest.json |
| [`pack`](#pack) | Pack bundle for a platform |
| [`publish`](#publish) | Publish multi-platform bundles |

## Quick Start

```yaml
name: Release
on:
  push:
    tags: ["v*"]

jobs:
  build:
    strategy:
      matrix:
        include:
          - target: darwin-arm64
            runner: macos-15
          - target: darwin-x86_64
            runner: macos-15-intel
          - target: linux-arm64
            runner: ubuntu-24.04-arm
          - target: linux-x86_64
            runner: ubuntu-24.04
          - target: win32-arm64
            runner: windows-11-arm
          - target: win32-x86_64
            runner: windows-2022

    runs-on: ${{ matrix.runner }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20.x"
      - run: npm ci --omit=dev

      - uses: zerocore-ai/tool-action/setup@v1
      - uses: zerocore-ai/tool-action/pack@v1
        with:
          target: ${{ matrix.target }}

      - uses: actions/upload-artifact@v4
        with:
          name: bundle-${{ matrix.target }}
          path: dist/*

  publish:
    needs: build
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - uses: actions/download-artifact@v4
        with:
          path: dist
          pattern: bundle-*
          merge-multiple: true

      - uses: zerocore-ai/tool-action/setup@v1
      - uses: zerocore-ai/tool-action/publish@v1
        env:
          TOOL_REGISTRY_TOKEN: ${{ secrets.TOOL_REGISTRY_TOKEN }}
```

---

## setup

Install tool-cli binary. Works on all platforms (macOS, Linux, Windows).

```yaml
- uses: zerocore-ai/tool-action/setup@v1
  with:
    version: latest              # tool-cli version (default: latest)
    fallback-to-source: "true"   # Build from source if prebuilt unavailable
```

### Inputs

| Input | Default | Description |
|-------|---------|-------------|
| `version` | `latest` | tool-cli version to install |
| `fallback-to-source` | `true` | Build from source if prebuilt binary unavailable |

### Outputs

| Output | Description |
|--------|-------------|
| `version` | Installed tool-cli version string |

---

## validate

Validate manifest.json before packing.

```yaml
- uses: zerocore-ai/tool-action/validate@v1
  with:
    strict: "false"              # Treat warnings as errors
    working-directory: "."       # Directory containing manifest.json
```

### Inputs

| Input | Default | Description |
|-------|---------|-------------|
| `strict` | `false` | Treat warnings as errors |
| `working-directory` | `.` | Directory containing manifest.json |

### Outputs

| Output | Description |
|--------|-------------|
| `valid` | `true` if validation passed |

---

## pack

Pack an MCPB bundle, optionally for a specific platform target.

```yaml
- uses: zerocore-ai/tool-action/pack@v1
  with:
    target: darwin-arm64         # Platform suffix (optional)
    output-dir: dist             # Output directory
    checksum: "true"             # Generate .sha256 file
    working-directory: "."       # Directory containing manifest.json
```

### Inputs

| Input | Default | Description |
|-------|---------|-------------|
| `target` | (none) | Platform target suffix (e.g., `darwin-arm64`). If omitted, creates bundle without platform suffix. |
| `output-dir` | `dist` | Directory to place the bundle |
| `checksum` | `true` | Generate `.sha256` checksum file |
| `working-directory` | `.` | Directory containing manifest.json |

### Outputs

| Output | Description |
|--------|-------------|
| `bundle-path` | Path to the created bundle file |
| `bundle-name` | Filename of the bundle |
| `checksum-path` | Path to checksum file (if generated) |

### Platform Targets

| Target | Runner |
|--------|--------|
| `darwin-arm64` | `macos-15` |
| `darwin-x86_64` | `macos-15-intel` |
| `linux-arm64` | `ubuntu-24.04-arm` |
| `linux-x86_64` | `ubuntu-24.04` |
| `win32-arm64` | `windows-11-arm` |
| `win32-x86_64` | `windows-2022` |

---

## publish

Publish multi-platform MCPB bundles to tool.store registry.

```yaml
- uses: zerocore-ai/tool-action/publish@v1
  with:
    bundles: "dist/*.mcpb dist/*.mcpbx"  # Glob patterns (space-separated)
    dry-run: "false"                      # Validate without uploading
    working-directory: "."                # Directory containing manifest.json
  env:
    TOOL_REGISTRY_TOKEN: ${{ secrets.TOOL_REGISTRY_TOKEN }}
```

### Inputs

| Input | Default | Description |
|-------|---------|-------------|
| `bundles` | `dist/*.mcpb dist/*.mcpbx` | Glob patterns for bundle files (space-separated) |
| `dry-run` | `false` | Validate without uploading |
| `working-directory` | `.` | Directory containing manifest.json |

### Outputs

| Output | Description |
|--------|-------------|
| `published` | `true` if publish succeeded |

### Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `TOOL_REGISTRY_TOKEN` | Yes (unless dry-run) | Authentication token for tool.store |

### Platform Detection

The publish action automatically maps bundle filenames to platform flags. Both `.mcpb` and `.mcpbx` extensions are supported:

| Filename Pattern | Flag |
|------------------|------|
| `*-darwin-arm64.mcpb[x]` | `--darwin-arm64` |
| `*-darwin-x86_64.mcpb[x]` | `--darwin-x64` |
| `*-linux-arm64.mcpb[x]` | `--linux-arm64` |
| `*-linux-x86_64.mcpb[x]` | `--linux-x64` |
| `*-win32-arm64.mcpb[x]` | `--win32-arm64` |
| `*-win32-x86_64.mcpb[x]` | `--win32-x64` |

> **Note:** `.mcpbx` is generated when the manifest uses MCPBX-specific features (HTTP transport, reference mode, system config, OAuth).

---

## Complete Example with GitHub Release

```yaml
name: Release
on:
  push:
    tags: ["v*"]

jobs:
  build:
    strategy:
      fail-fast: false
      matrix:
        include:
          - target: darwin-arm64
            runner: macos-15
          - target: darwin-x86_64
            runner: macos-15-intel
          - target: linux-arm64
            runner: ubuntu-24.04-arm
          - target: linux-x86_64
            runner: ubuntu-24.04
          - target: win32-arm64
            runner: windows-11-arm
          - target: win32-x86_64
            runner: windows-2022

    runs-on: ${{ matrix.runner }}
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: "20.x"

      - run: npm ci --omit=dev

      - uses: zerocore-ai/tool-action/setup@v1

      - uses: zerocore-ai/tool-action/pack@v1
        with:
          target: ${{ matrix.target }}

      - uses: actions/upload-artifact@v4
        with:
          name: bundle-${{ matrix.target }}
          path: |
            dist/*.mcpb
            dist/*.mcpbx
            dist/*.sha256

  release:
    needs: build
    runs-on: ubuntu-24.04
    permissions:
      contents: write
    steps:
      - uses: actions/download-artifact@v4
        with:
          path: dist
          pattern: bundle-*
          merge-multiple: true

      - uses: softprops/action-gh-release@v2
        with:
          files: |
            dist/*.mcpb
            dist/*.mcpbx
            dist/*.sha256

  publish:
    needs: release
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4

      - uses: actions/download-artifact@v4
        with:
          path: dist
          pattern: bundle-*
          merge-multiple: true

      - uses: zerocore-ai/tool-action/setup@v1

      - uses: zerocore-ai/tool-action/publish@v1
        env:
          TOOL_REGISTRY_TOKEN: ${{ secrets.TOOL_REGISTRY_TOKEN }}
```

## License

Apache-2.0
