# Patches

### `hap-nodejs-hevc.patch`

Adds **H.265 / HEVC** video codec support to [`hap-nodejs`](https://github.com/homebridge/HAP-NodeJS), which Homebridge bundles as its HomeKit Accessory Protocol implementation. Released `hap-nodejs` models H.264 only — its `VideoCodecType` enum has `H265` commented out and it has no `H265Profile` / `H265Level` definitions — so a plugin has no way to advertise HEVC to HomeKit until this gap is closed upstream.

This patch:

- Enables `VideoCodecType.H265` and adds `H265Profile` / `H265Level` / `H265CodecParameters` (`src/lib/camera/RTPStreamManagement.ts`).
- Advertises multiple `Video Codec Configuration` TLVs for both streaming (`VideoStreamingOptions.h265`) and recording (`CameraRecordingOptions.video` may be an array), so capable HomeKit clients (iOS 27 / tvOS 27+) can negotiate HEVC while everything else continues to use H.264.
- Is backward-compatible: an H.264-only configuration serializes to byte-identical TLV, leaving existing accessories — and the HomeKit Secure Video configuration hash — unchanged.

It is generated against `hap-nodejs` tag **`v2.1.7`**. The changes are structurally identical across recent releases; apply it against the tag your Homebridge bundles (Homebridge 2.x → `@homebridge/hap-nodejs` 2.x, Homebridge 1.x → `hap-nodejs` 0.14.x).

The patch also adds `src/lib/camera/HevcTlv.spec.ts`, a self-contained Jest test that verifies the multi-codec TLV serialization (H.264-only, H.264+H.265, and byte-identical back-compat). After applying the patch you can run it with `npx jest src/lib/camera/HevcTlv.spec.ts`. The existing camera specs (`RTPStreamManagement.spec.ts`, `RecordingManagement.spec.ts`) continue to pass unchanged, which confirms the H.264-only output is byte-for-byte identical to before.

See **[docs/HomeKit-HEVC.md](../docs/HomeKit-HEVC.md)** for the full rationale, installation instructions, the provisional H.265 profile/level caveat, and verification steps.

> [!NOTE]
> This patch is required only for the HEVC feature. Without it, the plugin runs normally and streams H.264 exactly as before — HEVC advertising is detected and enabled at runtime, and stays dormant when the running `hap-nodejs` predates HEVC support.
