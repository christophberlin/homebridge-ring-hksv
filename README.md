<p align="center">
  <a href="https://github.com/homebridge/verified/blob/master/verified-plugins.json"><img alt="Homebridge Verified" src="https://raw.githubusercontent.com/dgreif/ring/main/packages/homebridge-ring/branding/Homebridge_x_Ring.svg?sanitize=true" width="500px"></a>
</p>

# homebridge-ring-hksv

[![npm](https://badgen.net/npm/v/homebridge-ring-hksv)](https://www.npmjs.com/package/homebridge-ring-hksv)
[![npm](https://badgen.net/npm/dt/homebridge-ring-hksv)](https://www.npmjs.com/package/homebridge-ring-hksv)
[![verified-by-homebridge](https://badgen.net/badge/homebridge/unverified)](https://github.com/homebridge/homebridge/wiki/Verified-Plugins)
[![certified-hoobs-plugin](https://badgen.net/badge/HOOBS/Uncertified/yellow)](https://plugins.hoobs.org/plugin/homebridge-ring-hksv)
[![Donate](https://badgen.net/badge/Donate/BuyMeACoffee/FFDD00)](https://buymeacoffee.com/trinityhades)


`homebridge-ring-hksv` is a Homebridge platform plugin for Ring devices, with HomeKit Secure Video (HKSV) support.

## Permanent HKSV support boundary

HKSV support is intentionally limited to **always-powered, premium Ring
cameras** on a sufficiently capable Homebridge host. Floodlight Cam Pro/2 and
equivalent always-powered premium models are the target class.

Battery-powered, solar-powered, or low-performance camera deployments are
**out of scope and unsupported**. Small or low-power Homebridge hosts,
including Raspberry Pi-class systems, are also out of scope for HKSV. This is a
deliberate permanent product decision, not a compatibility gap on the roadmap;
do not use battery/low-power support as a future reliability or feature
criterion.

The plugin may continue to expose legacy compatibility settings related to
those deployments, but those settings do not imply support or a reliability
commitment.

## Origin and Attribution

This project is based on Dustin Greif's original Ring Homebridge ecosystem:

- Upstream plugin: [dgreif/homebridge-ring](https://github.com/dgreif/homebridge-ring)
- Upstream API library: [dgreif/ring-client-api](https://github.com/dgreif/ring-client-api)

Big thanks to Dustin and all upstream contributors. This fork reuses and extends that foundation.

## Important Plugin Options

| Option | Purpose |
| --- | --- |
| `enableHksv` | Enables experimental HKSV support for eligible cameras |
| `enableCameraMotionHistory` | Allows camera motion to fall back to recent Ring event history if live notifications are stale; disable to only trust push-triggered motion |
| `disableHksvOnBattery` | Legacy compatibility setting; battery-camera HKSV is out of scope and unsupported |
| `hksvPrebufferLengthMs` | HKSV prebuffer duration (minimum 4000ms) |
| `hksvFragmentLengthMs` | HKSV fragment duration target |
| `hksvMaxRecordingSeconds` | Optional safety cap for a recording session |
| `hksvPerformanceMode` | HKSV tuning profile for supported always-powered deployments (`balanced` or `quality`) |
| `hksvMaxConcurrentRecordings` / `hksvMaxQueuedBytes` | HKSV safeguards for supported systems under load |
| `hksvVideoBitrateKbps` / `hksvVideoMaxBitrateKbps` / `hksvVideoBufferSizeKbps` | HKSV recording bitrate controls for improving fast-motion quality |
| `hksvVideoCrf` / `hksvVideoPreset` | Optional libx264 quality and CPU tuning controls |
| `hksvVideoKeyframeInterval` | HKSV recording keyframe interval |
| `homeKitAccessoryTag` | Appends a tag to accessory names and HomeKit IDs so the same Ring device can be exposed as a distinct HomeKit accessory for debugging/testing |
| `cameraVideoCodec` | Preferred H.264 handling (`auto`, `copy`, `h264_v4l2m2m`, `h264_videotoolbox`, or `libx264`) |
| `hideDoorbellSwitch` / `hideCameraMotionSensor` / `hideCameraSirenSwitch` | Hides specific HomeKit-exposed services |
| `showPanicButtons` | Adds panic switches (use with caution) |
| `ffmpegPath` | Override FFmpeg binary path |
| `debug` | Enables additional logging |
| `disableLogs` | Disables plugin logging |

## v15 media configuration

v15 uses an adaptive media engine that keeps Ring call ownership separate from
each live-view and recording FFmpeg process. Configure `media.profile` as
`adaptive` (default) or `quality`. `lowPower` is retained only for legacy
configuration compatibility and is not a supported HKSV deployment target.
Advanced overrides live in
`media.recording`; they are strictly validated and take precedence over legacy
`hksv*` fields, which remain compatibility aliases during migration. Set
`media.recording.maxDurationSeconds` to `0` to disable its safety cap.

## Installation

If Homebridge is installed globally:

```bash
npm i -g --unsafe-perm homebridge-ring-hksv
```

If you want an opt-in prerelease channel from npm, install one of the dist-tags instead:

```bash
npm i -g --unsafe-perm homebridge-ring-hksv@dev
npm i -g --unsafe-perm homebridge-ring-hksv@nightly
```

The `latest` tag remains the stable release line. `dev` is intended for hand-picked prerelease builds, and `nightly` is intended for newer automated or near-mainline snapshots.

If running from source:

```bash
npm install
npm run build
```

To publish an opt-in prerelease channel to npm:

```bash
npm run publish:beta
npm run publish:nightly
```

Both commands create a prerelease semver version and publish it under the matching npm dist-tag, so users can install `homebridge-ring-hksv@dev` or `homebridge-ring-hksv@nightly` without affecting `latest`. Add `-- --dry-run` to preview the computed version without changing files or publishing.

## Basic Configuration

Use Homebridge UI (`homebridge-config-ui-x`) when possible.

Add a platform block with your Ring refresh token:

```json
{
  "platform": "Homebridge Ring HKSV",
  "refreshToken": "your-refresh-token"
}
```

If you need Home app to treat the same physical Ring device as a different HomeKit accessory, add a `homeKitAccessoryTag`:

```json
{
  "platform": "Homebridge Ring HKSV",
  "refreshToken": "your-refresh-token",
  "homeKitAccessoryTag": "Debug Home A"
}
```

Changing `homeKitAccessoryTag` updates both the exposed accessory name and the generated HomeKit identity, which changes the advertised MAC-style identifier shown during manual camera pairing.

## HKSV Status

HKSV support is experimental and actively evolving within the permanent support
boundary above. Behavior may still vary by supported camera model, Ring API
changes, and FFmpeg environment.

I currently am able to run 3 cameras with HKSV enabled on a Homebridge instance ran on a M4 Mac Mini 32GB of RAM.
Please report your experience and setup details to help improve support.

For fast-motion pixelation or stuttering in HKSV recordings, try increasing
`hksvVideoBitrateKbps` first if you are transcoding. On Apple Silicon Macs,
`cameraVideoCodec: "h264_videotoolbox"` can reduce CPU load by using hardware
encoding.

### Minimum specifications for HKSV

Only always-powered premium Ring cameras and a capable host are supported.
There is no supported low-power profile or battery-camera minimum
specification.

## Troubleshooting

For support and debugging:

- This repository issues: <https://github.com/trinityhades/homebridge-ring-hksv/issues>

- Upstream Ring wiki (token/auth/camera references): <https://github.com/dgreif/ring/wiki>

## Disclaimer

This plugin is not affiliated with Ring or Amazon.

Use emergency/panic-related automations at your own risk.

## License

MIT
