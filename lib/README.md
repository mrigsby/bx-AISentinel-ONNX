# `lib/` — Java dependency JARs

This directory holds the Java libraries the module loads via [cbjavaloader](https://forgebox.io/view/cbjavaloader) at `ModuleConfig.onLoad()`. The JARs themselves are **not committed** to the repository (`.gitignore` excludes everything here except this README). Install them once per deployment and they stay put until you replace them.

## What goes here

| File | Source | Purpose |
|---|---|---|
| `onnxruntime-<version>.jar` | [Maven — com.microsoft.onnxruntime:onnxruntime](https://mvnrepository.com/artifact/com.microsoft.onnxruntime/onnxruntime) | ONNX Runtime Java binding. Single monolithic JAR bundles native libs for linux-x64, macos-arm64, macos-x64, windows-x64; the JVM extracts the right one at first use. |
| `tokenizers-<version>.jar` | [Maven — ai.djl.huggingface:tokenizers](https://mvnrepository.com/artifact/ai.djl.huggingface/tokenizers) | DJL's HuggingFace tokenizer wrapper. Also a single JAR with bundled native binaries. |
| `api-<version>.jar` | [Maven — ai.djl:api](https://mvnrepository.com/artifact/ai.djl/api) | DJL core API. Required transitive dependency of `tokenizers`. Without it, the JVM fails with `NoClassDefFoundError: ai/djl/modality/nlp/preprocess/Tokenizer` at first tokenizer construction. Match the version to the `tokenizers` version. |

**Known-working versions** (current as of publication):

- `onnxruntime`: **1.22.0** (latest stable non-android release). Newer releases should work; if you hit a macOS Apple Silicon native-lib-loader issue on an older release, bump to the latest.
- `tokenizers`: **0.36.0**.
- `api`: **0.36.0** (must match the `tokenizers` version).

Pick the current release on Maven Central — the links in the table above go to the version listings. If a newer version exists by the time you read this, use it.

## How to place them

### Option 1 — direct download from Maven Central (recommended)

From the module's root directory:

```sh
mkdir -p lib
curl -L -o lib/onnxruntime-1.22.0.jar \
  https://repo1.maven.org/maven2/com/microsoft/onnxruntime/onnxruntime/1.22.0/onnxruntime-1.22.0.jar
curl -L -o lib/tokenizers-0.36.0.jar \
  https://repo1.maven.org/maven2/ai/djl/huggingface/tokenizers/0.36.0/tokenizers-0.36.0.jar
curl -L -o lib/api-0.36.0.jar \
  https://repo1.maven.org/maven2/ai/djl/api/0.36.0/api-0.36.0.jar
```

Swap the version numbers for whatever the current Maven Central releases are when you run this. Keep the `api` and `tokenizers` versions in sync.

### Option 2 — manual browser download

1. Visit the Maven pages linked in the table above.
2. Click the version you want; download the `.jar` artifact.
3. Drop it into this directory.

### Option 3 — from a shared system location

If several projects on the same host need these libs, place them in a shared directory (`/opt/bx-ai/lib/`, `~/.bx-aisentinel/lib/`, etc.) and point `ModuleConfig.onLoad()`'s loader at that path instead. A future `JarAssetManager` (parallel to `ModelAssetManager`) can automate this; for now, the default is this module-local directory.

## Verifying the install

After placing the JARs, start the server. The module's `onLoad()` logs to the `bx-aisentinel-onnx` LogBox channel:

```text
cbjavaloader appended …/lib (3 JAR(s) found)
```

If that line says `0 JAR(s) found` or the log complains about `lib/` being missing, the JARs haven't been placed correctly. If JARs are present but you see `NoClassDefFoundError: ai/djl/modality/nlp/preprocess/Tokenizer` at first scan, you're missing `api-*.jar`.

## Why not checked in

`onnxruntime.jar` is 30–50 MB (per platform-bundled-native reasons), and keeping binaries out of git history keeps module clones fast and publication packages small. Phase 4 adds a `box run-script download-jars` task and `JarAssetManager` so ops can automate placement as part of Docker builds / CI warmup.
