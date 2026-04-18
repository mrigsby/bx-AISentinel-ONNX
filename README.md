# bx-AISentinel-ONNX

> Sibling module for [bx-AISentinel](https://github.com/mrigsby/bx-AISentinel). Adds Tier 1 NER-based PII detection using ONNX Runtime for Java and a GLiNER PII model.

**Status:** v0.2.0-pre · ONNX Runtime wired, no model yet · [Changelog](CHANGELOG.md)

## What this is

A standalone ColdBox module that implements the [`IDetector@1.0.0` contract](https://github.com/mrigsby/bx-AISentinel/blob/main/models/detectors/CONTRACT.md) from `bx-AISentinel`. It plugs into the sentinel via the `externalDetectors` policy setting and catches free-form PII that regex / entropy / registry miss: names, addresses, medical record numbers, institutional IDs, etc.

The detector is also usable outside the sentinel — any BoxLang code can call `getInstance("OnnxNerDetector@bx-AISentinel-ONNX").scan(text)` directly for log sanitization, form validation, or any other entity-extraction need.

## Status

Real ONNX Runtime + HuggingFace tokenizer wiring landed in v0.2.0-pre. What's here:

- `OnnxNerDetector` — satisfies the `IDetector@1.0.0` contract and drives the tokenize → inference → decode pipeline.
- `OnnxSession` — thread-safe singleton wrapping `OrtEnvironment` + `OrtSession` + tokenizer; lazy-loads on first use and degrades gracefully when JARs / model files aren't present.
- `EntityDecoder` — pure-BoxLang BIO-tag decoder turning ONNX logits into `bx-AISentinel` Hit structs.
- [cbjavaloader](https://forgebox.io/view/cbjavaloader) registers the module's `lib/` directory so ONNX Runtime Java classes resolve at runtime.
- 20+ TestBox specs covering the full pipeline with a `MockOnnxSession` fixture — no JARs, ONNX Runtime, or real model required to run `box testbox run`.

**What still requires manual setup to use at runtime:** place ONNX Runtime + HuggingFace tokenizer JARs in [`lib/`](lib/README.md), and place a compatible ONNX model + tokenizer at the configured `modelPath`. Phase 4 adds `ModelAssetManager` + a `box run-script download-model` task to automate both.

Without those assets, `scan()` returns empty gracefully — the `bx-AISentinel` plugin seam treats it as a no-op rather than a failure, and the operator sees the problem via the `bx-aisentinel-onnx` LogBox channel.

## Related

- [bx-AISentinel](https://github.com/mrigsby/bx-AISentinel) — the core middleware this module plugs into.
- [bx-AISentinel-demo](https://github.com/mrigsby/bx-AISentinel-demo) — ColdBox + CBWire demo; will gain an opt-in Tier 1 page covering the install + config for this module.

## License

Apache 2.0 (see [`LICENSE`](LICENSE)).
