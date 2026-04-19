# bx-AISentinel-ONNX

> Tier 1 NER-based PII detection for [bx-AISentinel](https://github.com/mrigsby/bx-AISentinel). Plugs into the sentinel's external-detector plugin seam and catches free-form PII that regex / entropy / registry miss.

**Status:** v0.4.0-pre · end-to-end verified against real Piiranha model · [Changelog](CHANGELOG.md)

## What this is

A standalone ColdBox module that implements the [`IDetector@1.0.0` contract](https://github.com/mrigsby/bx-AISentinel/blob/main/models/detectors/CONTRACT.md). Install both this module and `bx-AISentinel`, place the ONNX Runtime JARs and a supported model on disk, and the sentinel automatically picks up contextual PII — names, addresses, phone numbers, medical record numbers, institutional IDs — alongside the formatted secrets its built-in regex catalog already catches.

The detector is also useful outside the sentinel. Any BoxLang code can call `getInstance( "OnnxNerDetector@bx-AISentinel-ONNX" ).scan( text )` (or the convenience `extractEntities` / `containsPii` / `redactInline` helpers) for log sanitization, form validation, or chat-input moderation — no middleware required.

## Install

### Install the module

```sh
box install git+https://github.com/mrigsby/bx-AISentinel-ONNX.git
```

CommandBox installs into `modules_app/bx-AISentinel-ONNX/`. The module declares [`cbjavaloader`](https://forgebox.io/view/cbjavaloader) as a dependency — CommandBox pulls it automatically.

### Place the Java libraries

The module does NOT bundle the ONNX Runtime / tokenizer JARs (they're 30–50 MB each and platform-complex). Place them in the module's `lib/` directory. Three documented paths:

```sh
# Option 1 — direct download from Maven Central (recommended)
cd modules_app/bx-AISentinel-ONNX/
mkdir -p lib
curl -L -o lib/onnxruntime-1.22.0.jar \
  https://repo1.maven.org/maven2/com/microsoft/onnxruntime/onnxruntime/1.22.0/onnxruntime-1.22.0.jar
curl -L -o lib/tokenizers-0.36.0.jar \
  https://repo1.maven.org/maven2/ai/djl/huggingface/tokenizers/0.36.0/tokenizers-0.36.0.jar
curl -L -o lib/api-0.36.0.jar \
  https://repo1.maven.org/maven2/ai/djl/api/0.36.0/api-0.36.0.jar

# Option 2 — manual browser download from Maven Central
# Drop all THREE JARs (onnxruntime-*.jar, tokenizers-*.jar, api-*.jar) into lib/
# by hand. The api JAR is a transitive dep of tokenizers — required.

# Option 3 — shared system location
# Place JARs anywhere and override ModuleConfig's lib loader path.
```

See [`lib/README.md`](lib/README.md) for details, links, and versions.

### Place the model

The default model is [GLiNER PII v1](https://huggingface.co/urchade/gliner_multi_pii-v1) — Apache 2.0, multi-language, 384-token context. Three delivery modes, chosen via the `assetMode` setting:

- **`manual`** (default) — download `model.onnx` + `tokenizer.json` yourself, place them at the configured `modelPath`. Validated on boot; clear error if anything is off.
- **`auto-download`** — module fetches missing files on first use from the registered source URL, verifies SHA-256 against a pinned value, and caches them. Explicit opt-in — no surprise network I/O at startup.
- **`shared-cache`** — same as auto-download, but `modelPath` points OUTSIDE the module directory (e.g. `~/.bx-aisentinel/models/gliner-pii-v1/` or `/opt/bx-ai/models/gliner-pii-v1/`), so weights survive `box install` reinstalls and multiple CommandBox projects on the same host share one copy.

Or download the model directly from HuggingFace with `curl` (the default `modelPath` is `./modules_app/bx-AISentinel-ONNX/assets/`):

```sh
cd modules_app/bx-AISentinel-ONNX/
mkdir -p assets
curl -L -o assets/model.onnx \
  https://huggingface.co/onnx-community/gliner_multi_pii-v1/resolve/main/onnx/model.onnx
curl -L -o assets/tokenizer.json \
  https://huggingface.co/onnx-community/gliner_multi_pii-v1/resolve/main/tokenizer.json
```

Or set `assetMode: "auto-download"` + `acceptUnverified: true` in module settings and let the detector fetch the files on its first `scan()` call.

> ⚠️ **Default `modelPath` is module-local** (`./modules_app/bx-AISentinel-ONNX/assets/`). A CommandBox reinstall of the module wipes the weights and forces a re-download (or re-place in manual mode). For production, override `modelPath` to a host-level directory like `~/.bx-aisentinel/models/gliner-pii-v1/` or `/opt/bx-ai/models/gliner-pii-v1/` so weights persist across updates.

## Wire into bx-AISentinel

Once both modules are installed and assets are in place, opt the detector into the sentinel's pipeline via policy:

```javascript
// In your host app's boxlang.json → modules.bx-aisentinel.settings
{
    "modules": {
        "bx-aisentinel": {
            "externalDetectors": [
                "OnnxNerDetector@bx-AISentinel-ONNX"
            ]
        },
        "bx-aisentinel-onnx": {
            "modelName" : "gliner-pii-v1",
            "modelPath" : "~/.bx-aisentinel/models/gliner-pii-v1/",
            "assetMode" : "manual"
        }
    }
}
```

That's it. From now on every `aiAgent()` with `bx-AISentinel` wired in the middleware chain gets NER-backed PII coverage on top of the Tier 0 detectors.

## Use directly (no sentinel)

```javascript
var detector = getInstance( "OnnxNerDetector@bx-AISentinel-ONNX" );

var hits = detector.scan( "Alice Martinez, born 1982-03-14, lives at 742 Evergreen Terrace." );
// → [
//     { label: "PII_PERSON_NAME", value: "Alice Martinez", start: 1,  end: 14, ... },
//     { label: "PII_DOB",         value: "1982-03-14",     start: 22, end: 31, ... },
//     { label: "PII_ADDRESS",     value: "742 Evergreen Terrace", ... }
//   ]

var emails = detector.extractEntities( text, [ "PII_EMAIL" ] );    // filter

if ( detector.containsPii( formInput ) ) { block(); }              // fast boolean

var safe = detector.redactInline( logLine, "[REDACTED]" );         // inline scrub
```

## Configuration

Defaults live in [`ModuleConfig.bx`](ModuleConfig.bx). Override via the host app's `boxlang.json → modules.bx-aisentinel-onnx.settings`.

| Setting | Default | Purpose |
| --- | --- | --- |
| `modelName` | `"gliner-pii-v1"` | Selector into the `SupportedModels` registry. See `list` below. |
| `modelPath` | `./modules_app/bx-AISentinel-ONNX/assets/` | Directory holding `model.onnx` + `tokenizer.json`. **Override to a host-level path for production** (see warning above). |
| `modelSource` | looked up from registry | Download URL. Only consulted in `auto-download` / `shared-cache` modes. |
| `tokenizerSource` | looked up from registry | Tokenizer download URL. |
| `modelChecksum` | looked up from registry | Expected SHA-256. Used in every mode to detect corruption. |
| `tokenizerChecksum` | looked up from registry | Expected SHA-256 for the tokenizer file. |
| `assetMode` | `"manual"` | `"manual"` / `"auto-download"` / `"shared-cache"`. |
| `eagerInit` | `false` | If `true`, load the ONNX session at module boot. Default false so non-NER workloads pay nothing. |
| `downloadTimeoutMs` | `120000` | Max time for a first-run download before giving up. |
| `acceptUnverified` | `false` | Allow auto-download of registry entries without a pinned SHA-256. |
| `acceptUnverifiedLicense` | `false` | Allow auto-download of registry entries whose license isn't yet verified. |
| `confidenceFloor` | `0.5` | Drop entity hits below this confidence. Higher is stricter. |

## Supported models

The authoritative registry lives in [`models/SupportedModels.bx`](models/SupportedModels.bx) — inspect it directly for the full list of registered models and their metadata. Current entries:

| `modelName` | Description | License | Decoder | Status |
| --- | --- | --- | --- | --- |
| `gliner-pii-v1` | GLiNER multi-PII v1. 6 languages, 384-token context. | Apache 2.0 (verified) | `gliner-span` (needs adapter — see DEV-NOTES) | Checksum unverified |
| `piiranha-pii-v1` | Piiranha v1. Fine-tuned mdeberta-v3, 98.5% F1, 17 PII types, 256-token context. | Unknown (unverified) | Standard BIO | Checksum + license unverified |

Both ship as `verified: false` — the `ModelAssetManager` will refuse auto-download by default. Operators either use `manual` mode (place files yourself) or pass `acceptUnverified: true` (and `acceptUnverifiedLicense: true` for Piiranha) to bypass. See [`models/SupportedModels.bx`](models/SupportedModels.bx) for the registry source.

## Safety rails

The module refuses to do anything surprising. It will:

- ✗ Refuse to auto-download a model whose SHA-256 isn't pinned in the registry (unless you pass `acceptUnverified: true`).
- ✗ Refuse to auto-download a model whose license we haven't personally verified (unless you pass `acceptUnverifiedLicense: true`).
- ✗ Refuse a file whose checksum doesn't match — always. Never silently proceeds with a wrong model.
- ✗ Refuse to write over an existing file during download.

It will:

- ✓ Fall back to `[]` hits when JARs / model are missing so the sentinel's pipeline stays up.
- ✓ Log every failure mode to the `bx-aisentinel-onnx` LogBox channel with an actionable message.
- ✓ Expose the full state via `detector.validateAssets()` for health endpoints.

## Entity schema

Each model ships with its own `labelToSentinel` mapping in the registry. Common outputs:

- `PII_PERSON_NAME` — given name, surname, or full name
- `PII_ADDRESS` — street, city, zipcode, or full address
- `PII_EMAIL` — email addresses
- `PII_PHONE` — phone numbers in any format
- `PII_DOB` — dates of birth
- `PII_SSN` — US social security numbers (or equivalent when supported)
- `PII_CREDIT_CARD` — card numbers
- `PII_PASSPORT` — passport numbers
- `PII_MEDICAL_RECORD` — medical record identifiers (model-dependent)
- `PII_NATIONAL_ID` — national ID numbers (CPF, identity cards, etc.)
- `PII_IP_ADDRESS` — IPv4/v6 addresses

These land as `token.label` when the sentinel mints a replacement token `⟦SECRET:PII_XXX:hmac8⟧`.

## Performance

Target: **sub-250 ms warmed inference on 1 KB of text** with GLiNER-small on a developer laptop CPU. Cold start adds ~500 ms for native-lib extraction + JIT. Resident memory runs 200–400 MB per loaded model.

The detector lazy-loads by default (`eagerInit = false`) so a host app that never calls `scan()` pays nothing.

## Health check

```javascript
var status = getInstance( "OnnxNerDetector@bx-AISentinel-ONNX" ).validateAssets();
// → {
//     ok            : false,           // assets present + label map populated
//     sessionLoaded : false,           // current session lifecycle state (lazy by default)
//     lastScanError : "",              // last error caught by scan(); "" if none
//     assetManager  : {
//         ok           : false,
//         mode         : "manual",
//         modelName    : "gliner-pii-v1",
//         modelFile    : "...",
//         missingFiles : [ "..." ],
//         licenseStatus: "verified",
//         message      : "Missing asset file(s) at ..."
//     },
//     session       : {
//         loaded       : false,
//         modelPath    : "...",
//         loadError    : ""
//     }
//   }
```

Surface this from a host health endpoint. `ok: false` with a populated `assetManager.missingFiles` or `assetManager.checksumIssues` tells ops exactly what's wrong.

**v0.4.0+ contract change:** `ok` no longer requires `session.loaded` to be `true`. It now means "the detector has everything it needs to attempt loading on first scan" — assets present, label map configured. Use the new `sessionLoaded` field to read current session lifecycle state, and `lastScanError` to see whether the most recent `scan()` failed silently. `validateAssets()` itself has NO side effects — it never triggers a model load. To force load at construction, pass `eagerInit: true` in the module settings.

## Troubleshooting

| Symptom | Fix |
| --- | --- |
| `scan()` always returns `[]` | Call `getLastScanError()` first — non-empty means a real error was caught. Empty + still no hits → run `validateAssets()`; probably missing JARs, missing model files, or empty `id2label`. Check the `bx-aisentinel-onnx` LogBox channel. |
| `NoClassDefFoundError: ai/djl/modality/nlp/preprocess/Tokenizer` | Missing the DJL `api-*.jar` in `lib/`. The `tokenizers` extension JAR depends on it. See the install steps above. |
| `UnverifiedModel` error during download | The registry entry doesn't have a pinned SHA-256. Pass `acceptUnverified: true` if you trust the source, or switch to `manual` mode. |
| `UnverifiedLicense` error during download | The model's license hasn't been vetted by the module maintainer. Pass `acceptUnverifiedLicense: true` after confirming terms yourself, or pick a different model. |
| `PostDownloadValidationFailed` | Download succeeded but the file is still missing or wrong-checksum. Network intermittency or a 30x redirect that CFHTTP didn't follow. Retry with `manual` mode. |
| `Checksum mismatch` | File was tampered, corrupted, or the registry entry is stale. Delete the file and re-download. Do NOT disable verification. |
| `cbjavaloader not found` | `box install` didn't pull the dep. Re-run or check network access to ForgeBox. |
| `validateAssets().ok` always `true` but `scan()` still empty | Asset readiness is now distinct from session-loaded state (v0.4.0+). Check `validateAssets().sessionLoaded` and `validateAssets().lastScanError`. Set `eagerInit: true` in settings to force-load at construction. |

## Development

```sh
git clone https://github.com/mrigsby/bx-AISentinel-ONNX.git
cd bx-AISentinel-ONNX/tests
box install               # pulls TestBox 7
box server start          # boots an isolated BoxLang engine on port 8083
# Runner URL: http://localhost:8083/
# or run headlessly:
box testbox run
```

Test harness is completely isolated from the module's runtime (own port, own `.engine/`, own TestBox install). Specs run without any JARs or real model — the `MockOnnxSession` and `TestableAssetManager` fixtures substitute for the real thing.

To exercise the real-model integration test:

```sh
# After placing JARs in lib/ and downloading the model:
box testbox run labels=real-model
```

## Related

- [bx-AISentinel](https://github.com/mrigsby/bx-AISentinel) — the core middleware this module plugs into.
- [bx-AISentinel-demo](https://github.com/mrigsby/bx-AISentinel-demo) — ColdBox + CBWire demo. The Tier 1 NER help page documents how to enable this module there.
- [`IDetector@1.0.0` contract](https://github.com/mrigsby/bx-AISentinel/blob/main/models/detectors/CONTRACT.md) — the interface this module implements.

## License

Apache 2.0 (see [`LICENSE`](LICENSE)).

Model licensing is separate from the module's license and model-specific. See `SupportedModels.bx` and the linked model cards for each entry's terms. The module only auto-downloads models whose license status is `verified`; everything else is operator opt-in.
