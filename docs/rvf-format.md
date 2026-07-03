# RVF Container Format

RuView uses **RVF** (RuVector Format) containers to package models, runtime
state and edge WASM modules. Two related layouts exist in this repository.

## 1. Segmented model container (`wifi-densepose-sensing-server`)

Source: `crates/wifi-densepose-sensing-server/src/rvf_container.rs` and
`rvf_pipeline.rs`.

A container is a sequence of 64-byte-aligned segments. Each segment starts with
a 64-byte header:

| Field | Notes |
|-------|-------|
| `magic` | `0x52564653` (`SEGMENT_MAGIC`) |
| `version` | `1` (`SEGMENT_VERSION`) |
| `seg_type` | segment kind (see below) |
| `payload_length` | payload bytes following the header |
| `content_hash` | CRC32 of the payload — integrity only, **not** cryptographic |

Segment kinds include: `SEG_VEC` (0x01), `SEG_MANIFEST` (0x05), `SEG_QUANT`
(0x06), `SEG_META` (0x07), `SEG_WITNESS` (0x0A), `SEG_PROFILE` (0x0B),
`SEG_LORA` (0x0D), plus pipeline segments `SEG_INDEX`, `SEG_OVERLAY`,
`SEG_AGGREGATE_WEIGHTS`, `SEG_CRYPTO` (0x0C), `SEG_WASM`, `SEG_DASHBOARD`.

`RvfReader::from_bytes` validates each segment's magic, version and CRC32
content hash, then exposes segments via `find_segment(seg_type)`.

## 2. Edge WASM module (`wifi-densepose-wasm-edge`)

Source: `crates/wifi-densepose-wasm-edge/src/rvf.rs`.

Layout: `RVF_HEADER_SIZE` (32) header + `RVF_MANIFEST_SIZE` (96) manifest + WASM
payload + `RVF_SIGNATURE_LEN` (64) trailing signature. Magic `0x01465652`,
format version `1`. The manifest declares required host capabilities via
`CAP_*` flags (`CAP_READ_PHASE`, `CAP_READ_AMPLITUDE`, `CAP_READ_VARIANCE`,
`CAP_READ_VITALS`, `CAP_READ_HISTORY`, `CAP_EMIT_EVENTS`, `CAP_LOG`;
`CAP_ALL = 0x7F`).

## Security status — signed-module verification is UNENFORCED

> **Frozen by closeout 2026-07-03.** RVF is currently a **write-only** signed
> format: builders declare capability flags and reserve a signature field, but
> **no verifier exists**.

Concretely:

- The edge-module builder (`rvf.rs::build_rvf`) writes an **all-zero**
  signature (`&[0u8; RVF_SIGNATURE_LEN]`) and expects modules to be "signed
  externally". `patch_signature` can insert a signature, but nothing checks it.
- The model-container pipeline (`rvf_pipeline.rs`) writes the `SEG_CRYPTO`
  segment **empty**.
- The loader (`RvfReader::from_bytes`) verifies **CRC32 structural integrity
  only**. It performs **no cryptographic signature verification** and **does not
  gate the `CAP_*` capability flags**. Modules are loaded on trust.
- On the first RVF load per process the sensing server emits an unmissable
  `WARN`: *"RuView: RVF signed-module verification is UNENFORCED …"*.

**Do not load RVF modules by trust.** Treat unsigned or zero-signature modules
as untrusted. Unfreezing this capability requires a verifying loader that (a)
checks an Ed25519 signature over the header + manifest + payload and (b) enforces
the declared `CAP_*` flags against a host capability grant, before any module is
executed.
