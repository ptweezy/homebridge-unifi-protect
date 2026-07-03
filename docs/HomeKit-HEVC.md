<SPAN ALIGN="CENTER" STYLE="text-align:center">
<DIV ALIGN="CENTER" STYLE="text-align:center">

[![homebridge-unifi-protect: Native HomeKit support for UniFi Protect](https://raw.githubusercontent.com/hjdhjd/homebridge-unifi-protect/main/images/homebridge-unifi-protect.svg)](https://github.com/hjdhjd/homebridge-unifi-protect)

# Homebridge UniFi Protect

[![Downloads](https://img.shields.io/npm/dt/homebridge-unifi-protect?color=%230559C9&logo=icloud&logoColor=%23FFFFFF&style=for-the-badge)](https://www.npmjs.com/package/homebridge-unifi-protect)
[![Version](https://img.shields.io/npm/v/homebridge-unifi-protect?color=%230559C9&label=Homebridge%20UniFi%20Protect&logo=ubiquiti&logoColor=%23FFFFFF&style=for-the-badge)](https://www.npmjs.com/package/homebridge-unifi-protect)
[![UniFi Protect@Homebridge Discord](https://img.shields.io/discord/432663330281226270?color=0559C9&label=Discord&logo=discord&logoColor=%23FFFFFF&style=for-the-badge)](https://discord.gg/QXqfHEW)
[![verified-by-homebridge](https://img.shields.io/badge/homebridge-verified-blueviolet?color=%23491F59&style=for-the-badge&logoColor=%23FFFFFF&logo=homebridge)](https://github.com/homebridge/homebridge/wiki/Verified-Plugins)

## Complete HomeKit support for the UniFi Protect ecosystem using [Homebridge](https://homebridge.io).
</DIV>
</SPAN>

### HEVC (H.265) and 4K Streaming for iOS 27 / tvOS 27

Beginning with iOS 27 and tvOS 27, HomeKit can negotiate **HEVC (H.265)** and **higher-resolution (up to 4K)** camera streams — a capability described in Apple's [HomeKit Secure Video Open Source Compatibility Guide](https://developer.apple.com/download/files/HomeKit-Secure-Video-Open-Source-Compatibility-Guide.pdf). This lets HBUP deliver a Protect camera's **native H.265 stream directly to HomeKit without transcoding**, which is the enhancement requested in [issue #1322](https://github.com/hjdhjd/homebridge-unifi-protect/issues/1322).

Most modern UniFi Protect cameras (G4, G5, G6 families) encode in H.265 by default. Historically, HomeKit only accepted H.264, so HBUP had to *transcode* those H.265 streams to H.264 for live viewing — burning CPU (or a hardware encoder), adding latency, and losing quality. With HEVC support, capable HomeKit clients receive the camera's native stream **copied, not re-encoded**, which is faster, lighter, and higher fidelity.

> [!IMPORTANT]
> **This feature requires a patched `hap-nodejs`.** HomeKit codec negotiation is implemented in `hap-nodejs` (the HomeKit Accessory Protocol library that Homebridge bundles), not in this plugin. As of this writing, released `hap-nodejs` (e.g. v2.1.7) still models H.264 only — its `VideoCodecType` enum has `H265` commented out, and there are no `H265Profile` / `H265Level` definitions. **A plugin cannot advertise a codec that the underlying HAP library has no way to express.** This repository therefore ships both halves of the work: the plugin-side changes *and* a patch that adds HEVC support to `hap-nodejs`. Until an official `hap-nodejs` release adds HEVC, you must apply the patch yourself (see [Installing the hap-nodejs patch](#installing-the-hap-nodejs-patch)).

- - -

### What is (and isn't) implemented

| Capability | Status | Notes |
|---|---|---|
| **HEVC live streaming (passthrough)** | ✅ Implemented | A native H.265 camera stream is copied directly to iOS 27 / tvOS 27 clients — no transcoding. |
| **4K live streaming** | ✅ Already supported | HBUP already advertises resolutions up to 3840×2160. iOS 27 clients can now request them; the copy path serves the native 4K stream. No change was needed here. |
| **H.264 fallback** | ✅ Guaranteed | H.264 is always advertised first. Pre‑iOS‑27 clients (and any client that declines HEVC) transparently continue to use H.264 exactly as before. |
| **HEVC HomeKit Secure Video *recording*** | ⏳ Follow-up | The HAP patch *adds the API* to advertise HEVC recording, but the plugin does not use it yet — true recording *passthrough* additionally requires a copy path in [`homebridge-plugin-utils`](https://github.com/hjdhjd/homebridge-plugin-utils), whose recording pipeline currently always re-encodes to H.264. See [HomeKit Secure Video recording](#homekit-secure-video-recording). HKSV continues to record normally (H.264) in the meantime. |

- - -

### How it works

The implementation spans two layers:

**1. `hap-nodejs` (patched — [`patches/hap-nodejs-hevc.patch`](../patches/hap-nodejs-hevc.patch))**

  - `VideoCodecType.H265 = 0x01` is enabled, and `H265Profile` / `H265Level` enums are added (`src/lib/camera/RTPStreamManagement.ts`).
  - `VideoStreamingOptions` gains an optional `h265` field. `_supportedVideoStreamConfiguration()` now advertises **multiple** `Video Codec Configuration` TLVs — the mandatory H.264 configuration plus an H.265 one when supplied. HomeKit clients select whichever they support.
  - The recording manager (`RecordingManagement.ts`) is symmetrically extended so `CameraRecordingOptions.video` may be an array of codec configurations.
  - The changes are **fully backward-compatible**: a single-codec (H.264-only) configuration serializes to byte-identical TLV, so existing accessories and the HKSV configuration hash are unaffected.

**2. `homebridge-unifi-protect` (this plugin — [`src/protect-stream.ts`](../src/protect-stream.ts))**

  - **Capability detection.** At startup the streaming delegate resolves `VideoCodecType.H265` from the *running* HAP. On an unpatched Homebridge this is `undefined`, so the plugin quietly falls back to today's H.264-only behavior. The plugin never imports the HEVC enums directly, so it continues to build and run against any supported Homebridge.
  - **Advertising.** HEVC is advertised only when the camera is **currently encoding in H.265**, cropping is disabled, and the `Video.Stream.HEVC` [feature option](https://github.com/hjdhjd/homebridge-unifi-protect/blob/main/docs/FeatureOptions.md) is enabled (it is, by default). We never advertise HEVC for an H.264 or AV1 camera, because HEVC is delivered by *copying* the native stream — there is no HEVC encoder in the FFmpeg pipeline to transcode *to*. The feature option gives you a per-camera (or global) off switch should the provisional profile/level values misbehave with a particular client, without having to revert the `hap-nodejs` patch.
  - **Negotiation.** When a stream starts, HomeKit tells us which codec it selected (`VideoInfo.codec`). If it chose HEVC, the plugin forces a stream **copy** (`-codec:v copy`), bypassing all transcoding preferences, and selects the codec-appropriate bitstream filter (`hevc_mp4toannexb` for the timeshift-buffer path, versus `h264_mp4toannexb`).

- - -

> [!WARNING]
> ### Provisional H.265 profile / level values
>
> Apple's tvOS 27 / iOS 27 compatibility guide is a **developer preview**, and the exact numeric identifiers HomeKit expects for the H.265 **profile** and **level** fields are not yet part of any public, finalized specification. The patch advertises **Main profile** at levels through **5.1** — reasonable, spec-consistent defaults that cover Protect's H.265 streams through 4K — but they may need adjustment once Apple finalizes the guide.
>
> These values live in exactly two places so they are trivial to tune:
> - `hap-nodejs`: the `H265Profile` / `H265Level` enums in `patches/hap-nodejs-hevc.patch`.
> - this plugin: `HOMEKIT_HEVC_PROFILES` / `HOMEKIT_HEVC_LEVELS` at the top of [`src/protect-stream.ts`](../src/protect-stream.ts) (they must match the enum values above).
>
> Because HEVC is only ever advertised for H.265 cameras (and never when cropping is enabled), a mismatch here is **contained**: H.264 cameras are entirely unaffected, and a client that rejects the H.265 configuration simply falls back to H.264.

- - -

### Installing the hap-nodejs patch

The patch modifies `hap-nodejs` source. Which package and version you patch depends on your Homebridge major version:

- **Homebridge 2.x** bundles `@homebridge/hap-nodejs` (2.x).
- **Homebridge 1.x** bundles `hap-nodejs` (0.14.x).

The HEVC changes are structurally identical across these releases; [`patches/hap-nodejs-hevc.patch`](../patches/hap-nodejs-hevc.patch) is generated against tag `v2.1.7`. Apply it against the tag matching your Homebridge's bundled version.

#### Option A — build a patched hap-nodejs and pin it with an npm override (recommended)

```sh
# 1. Clone hap-nodejs at the tag matching your Homebridge's bundled version.
git clone https://github.com/homebridge/HAP-NodeJS.git
cd HAP-NodeJS
git checkout v2.1.7          # or the version your Homebridge bundles

# 2. Apply the patch and build.
git apply /path/to/homebridge-unifi-protect/patches/hap-nodejs-hevc.patch
npm ci
npm run build
npm pack                     # produces e.g. homebridge-hap-nodejs-2.1.7.tgz
```

Then point your Homebridge installation at the patched build using an [npm `overrides`](https://docs.npmjs.com/cli/v10/configuring-npm/package-json#overrides) entry in your Homebridge install's `package.json`, and reinstall:

```jsonc
{
  "overrides": {
    "hap-nodejs": "file:/absolute/path/to/homebridge-hap-nodejs-2.1.7.tgz"
    // For Homebridge 2.x, override "@homebridge/hap-nodejs" instead.
  }
}
```

#### Option B — patch-package against the installed build

If you use [`patch-package`](https://www.npmjs.com/package/patch-package), edit the compiled files under `node_modules/<hap-nodejs>/dist/lib/camera/` to mirror the source patch, then run `npx patch-package <hap-nodejs>` to capture a `dist`-level patch that reapplies on every `npm install`. This keeps the change local to one Homebridge install.

> After patching, restart Homebridge. Then rebuild this plugin (`npm run build`) so it is compiled against the HEVC-capable type definitions if you develop against the patched `hap-nodejs`. The published plugin does **not** require the patched types to build — it detects HEVC at runtime.

- - -

### Verifying HEVC passthrough

1. Confirm the camera is encoding in H.265 (UniFi Protect → camera → Settings → Advanced → codec, or check the HBUP startup log — the codec is shown in each camera's streaming/HKSV log line as `HEVC`).
2. From an **iOS 27 / tvOS 27** device (e.g. an Apple TV running tvOS 27, per the tester in [issue #1322](https://github.com/hjdhjd/homebridge-unifi-protect/issues/1322)), open the camera's live view in the Home app.
3. In the HBUP log, the streaming request line should show the `HEVC` codec **without** the transcoding gear indicator (⚙), meaning the stream is being copied rather than transcoded.
4. For byte-level confirmation, enable the `Debug.Video.FFmpeg` [feature option](https://github.com/hjdhjd/homebridge-unifi-protect/blob/main/docs/FeatureOptions.md) and confirm the FFmpeg command line contains `-codec:v copy` (and, on the timeshift-buffer path, `-bsf:v hevc_mp4toannexb`) rather than an encoder such as `libx264` / `h264_videotoolbox`.

If HomeKit falls back to H.264 (transcoding gear present), the H.265 configuration was declined — most likely the profile/level values need adjustment for your client's OS build. See the [provisional values warning](#provisional-h265-profile--level-values) above.

- - -

### HomeKit Secure Video recording

The HAP patch also enables *advertising* HEVC for HomeKit Secure Video recording, but this plugin does **not** yet advertise it, and HKSV recording continues to use H.264 as before. The reason is deliberate: `homebridge-plugin-utils`' recording pipeline always **re-encodes** video to H.264 (via its `recordEncoder`) — it has no stream-copy path for recording. Advertising HEVC recording without a matching copy path would tell HomeKit "I will record H.265" while actually delivering H.264, corrupting recordings.

Enabling HEVC recording passthrough is a clean follow-up that requires a `-codec:v copy` recording path (with `hevc_mp4toannexb`) in `homebridge-plugin-utils`' `FfmpegRecordingProcess`. Once that exists, this plugin can advertise an H.265 recording configuration (the HAP side already supports it) and select it for H.265 cameras.

- - -

### Safety and fallback guarantees

- On an **unpatched** Homebridge, everything behaves exactly as it does today — the plugin detects the absence of `VideoCodecType.H265` and never advertises or attempts HEVC.
- For **H.264 and AV1 cameras**, nothing changes — HEVC is only ever advertised for cameras currently encoding in H.265.
- **Older HomeKit clients** (pre‑iOS‑27) ignore the additional H.265 codec configuration and continue to negotiate H.264.
- **Cropping** and HEVC passthrough are mutually exclusive (cropping requires a transcode), so enabling cropping cleanly disables HEVC advertising for that camera.
