# bx-AISentinel-ONNX

> Sibling module for [bx-AISentinel](https://github.com/mrigsby/bx-AISentinel). Adds Tier 1 NER-based PII detection using ONNX Runtime for Java and a GLiNER PII model.

**Status:** v0.1.0-pre · scaffolding — not yet functional · [Changelog](CHANGELOG.md)

## What this is

A standalone ColdBox module that implements the [`IDetector@1.0.0` contract](https://github.com/mrigsby/bx-AISentinel/blob/main/models/detectors/CONTRACT.md) from `bx-AISentinel`. It plugs into the sentinel via the `externalDetectors` policy setting and catches free-form PII that regex / entropy / registry miss: names, addresses, medical record numbers, institutional IDs, etc.

The detector is also usable outside the sentinel — any BoxLang code can call `getInstance("OnnxNerDetector@bx-AISentinel-ONNX").scan(text)` directly for log sanitization, form validation, or any other entity-extraction need.

## Status

This release is pre-functional scaffolding. The detector resolves, satisfies the contract, and returns empty results — enough to verify the `bx-AISentinel` plugin seam loads the module cross-repo. Real ONNX Runtime wiring and GLiNER PII integration land in upcoming phases:

- **Phase 3** (in progress): ONNX session + HuggingFace tokenizer wiring; fixture-model integration tests.
- **Phase 4**: `ModelAssetManager` (manual / auto-download / shared-cache modes), GLiNER PII v1 entity schema, `box run-script download-model` task, real-weight specs.

Check back at the first `v0.1.0` tag for a working module.

## Related

- [bx-AISentinel](https://github.com/mrigsby/bx-AISentinel) — the core middleware this module plugs into.
- [bx-AISentinel-demo](https://github.com/mrigsby/bx-AISentinel-demo) — ColdBox + CBWire demo; will gain an opt-in Tier 1 page covering the install + config for this module.

## License

Apache 2.0 (see [`LICENSE`](LICENSE)).
