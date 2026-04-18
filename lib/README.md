# `lib/` — Java dependency JARs

This directory holds the Java libraries the module loads via [cbjavaloader](https://forgebox.io/view/cbjavaloader) at `ModuleConfig.onLoad()`. The JARs themselves are **not committed** to the repository (`.gitignore` excludes everything here except this README). Install them once per deployment and they stay put until you replace them.

## What goes here

| File | Source | Purpose |
|---|---|---|
| `onnxruntime-<version>.jar` | [Maven — com.microsoft.onnxruntime:onnxruntime](https://mvnrepository.com/artifact/com.microsoft.onnxruntime/onnxruntime) | ONNX Runtime Java binding. Single monolithic JAR bundles native libs for linux-x64, macos-arm64, macos-x64, windows-x64; the JVM extracts the right one at first use. |
| `tokenizers-<version>.jar` | [Maven — ai.djl.huggingface:tokenizers](https://mvnrepository.com/artifact/ai.djl.huggingface/tokenizers) | DJL's HuggingFace tokenizer wrapper. Also a single JAR with bundled native binaries. |

**Current pinned versions** (tracked in `models/SupportedModels.bx` once Phase 4 lands):

- `onnxruntime`: **1.24.0** or newer (required for the macOS Apple Silicon native-lib-loader fix).
- `tokenizers`: **0.33.0** or newer.

## How to place them

### Option 1 — CommandBox Maven install (recommended)

From the module's root directory:

```sh
box install jar:com.microsoft.onnxruntime:onnxruntime:1.24.0 --installPath=lib
box install jar:ai.djl.huggingface:tokenizers:0.33.0 --installPath=lib
```

### Option 2 — manual download

1. Download the JARs from Maven Central (links in the table above).
2. Drop them into this directory.

### Option 3 — from a shared system location

If several projects on the same host need these libs, place them in a shared directory (`/opt/bx-ai/lib/`, `~/.bx-aisentinel/lib/`, etc.) and override the module's lib-path setting in `boxlang.json → modules.bx-aisentinel-onnx.settings.libPath`. Phase 4 formalizes this via `JarAssetManager` — for now, the default is this module-local directory.

## Verifying the install

After placing the JARs, start the server. The module's `onLoad()` logs to the `bx-aisentinel-onnx` LogBox channel:

```
cbjavaloader appended …/lib (2 JAR(s) found)
```

If that line says `0 JAR(s) found` or the log complains about `lib/` being missing, the JARs haven't been placed correctly.

## Why not checked in

`onnxruntime.jar` is 30–50 MB (per platform-bundled-native reasons), and keeping binaries out of git history keeps module clones fast and publication packages small. Phase 4 adds a `box run-script download-jars` task and `JarAssetManager` so ops can automate placement as part of Docker builds / CI warmup.
