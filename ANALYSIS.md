# Code Analysis

Date: 2026-08-17

Scope: first-pass review of the platform lifecycle, media/HKSV pipeline,
configuration handling, security/privacy surfaces, dependency health, tests,
and release metadata. This review is analysis-only; runtime source files were
not changed.

## Findings

### High - Derived media limits can exceed the documented maxima

**Evidence:** `media-config.ts` derives `maxBitrateKbps` as twice the selected
`bitrateKbps`, then derives `bufferSizeKbps` as twice that value. The fallback
path does not re-apply the upper bounds after deriving those values.

For example, this valid-looking configuration:

```json
{
  "media": {
    "recording": {
      "bitrateKbps": 12000
    }
  }
}
```

normalizes to `bitrateKbps=12000`, `maxBitrateKbps=24000`, and
`bufferSizeKbps=48000`, even though the documented maxima are 20,000 and
40,000 respectively. The generated FFmpeg arguments can therefore violate
the configuration contract and consume more memory/bandwidth than the safety
limits imply.

**Recommendation:** Clamp derived defaults to their maxima, or reject a
bitrate whose derived values cannot fit. Add boundary tests for the maximum
bitrate and for explicit/derived combinations.

### Medium - Package lock metadata is inconsistent with the package manifest

**Evidence:** `package.json` declares version `15.0.1`, while the root entry in
`package-lock.json` declares `15.0.0-nightly.202607092250`. The lockfile's
dependency graph is usable, but this drift makes release provenance and
reproducibility harder to audit and can produce misleading package metadata in
automation.

**Recommendation:** Update the lockfile root package metadata as part of the
release/versioning workflow and add a CI check that the two versions match.

### Medium - Production dependency audit reports unresolved vulnerabilities

**Evidence:** `npm audit --omit=dev` reported 18 production-tree findings
(4 moderate, 13 high, 1 critical) in the current checkout. The affected tree
includes transitive packages such as `axios`, `ip`, `fast-uri`, and Homebridge
UI/Nest-related packages.

**Impact:** The plugin runs inside a Homebridge process and includes a local
configuration UI, so vulnerable transitive HTTP/UI dependencies increase the
attack surface even where the plugin does not directly call the affected API.

**Recommendation:** Triage the audit output by reachable code path, update
direct dependencies where fixes are available, and document accepted
transitive exceptions with expiry/owner dates. Avoid `npm audit fix --force`
without testing Homebridge compatibility because it may introduce major
dependency changes.

### Medium - Test coverage is concentrated on recently added media helpers

**Evidence:** The repository's test command executes one Node test file with 9
tests. Those tests cover media normalization, queue behavior, parser bounds,
and selected lifecycle cases, but there are no automated tests for platform
startup/config migration, refresh-token persistence, the custom UI handlers,
camera accessory construction, or real FFmpeg/RTP integration.

**Impact:** Regressions in authentication/config updates, HomeKit service
registration, and process/network integration can pass CI unnoticed.

**Recommendation:** Add focused tests around configuration migration and
refresh-token replacement, UI request validation/error handling, camera
recording teardown, and at least one fixture-based FFmpeg/RTP integration path.
Keep external Ring calls mocked and deterministic.

### Low - Invalid Homebridge config fallback can replace the wrong token

**Evidence:** `ring-platform.ts` parses and structurally updates the
Homebridge config, but on any parse/update error it falls back to
`configContents.replace(oldRefreshToken, newRefreshToken)`.

**Impact:** If the file is malformed or the expected platform shape changes,
the fallback can replace an unrelated occurrence of the old token and write a
configuration that was never structurally validated. The error is also
converted into a success-shaped update result.

**Recommendation:** Do not perform an unconstrained string replacement.
Surface the update failure and require a valid platform entry, or use a
targeted parser/recovery path that verifies exactly one replacement before
writing.

## Positive observations

- The media ingress and FFmpeg wrapper have explicit shutdown, abort, and
  escalation paths, including synchronous child cleanup on process exit.
- The recording resource governor reserves capacity before resolving waiters
  and prevents two active recording encoders for the same camera.
- Nested media settings reject unknown keys and invalid types rather than
  silently accepting arbitrary values.
- The baseline build, lint, and existing media tests pass in the fork checkout.

## Validation

Commands run in the fork:

- `npm ci`
- `npm run lint` - passed
- `npm test` - passed; build completed and 9 tests passed
- `npm audit --omit=dev` - reported the production dependency findings above
- Direct boundary probe of `normalizeMediaConfig` confirmed the derived-limit
  overflow described in the first finding.

## HKSV reliability re-evaluation

### Permanent support boundary

This project permanently excludes battery-powered, solar-powered, and
low-performance deployments from HKSV support. The target is always-powered,
premium Ring cameras such as Floodlight Cam Pro/2 and equivalent models, hosted
on a capable computer. Raspberry Pi-class and other low-power Homebridge hosts
are also excluded from the HKSV reliability target.

This is a product scope decision, not an unresolved compatibility problem.
Battery/low-power behavior must not be reopened as a future reliability goal.
Legacy configuration options and implementation branches may remain for
backward compatibility, but they are unsupported and must not influence
supported-deployment estimates, test matrices, or roadmap priorities.

### Revised estimate

The earlier estimate of 60-70% for controlled wired deployments was
optimistic. Based on the complete control flow, the current implementation is
better described as:

- **50-65% dependable for wired cameras on a known-good host**
- **Not production-grade for unattended security coverage**

This is an operational reliability estimate, not a claim that exactly this
percentage of clips will succeed. The code has substantially improved cleanup
and resource bounding, but successful recording still requires several
independent conditions: HomeKit must enable recording, Ring notifications or
history must produce a motion/doorbell event, the Ring live call must negotiate,
FFmpeg must accept the SDP and selected codec, and HomeKit must consume the
fragmented MP4 stream before a timeout or cancellation.

### Most important distinction: live view is not an HKSV health check

`CameraSource` has separate live-stream and recording paths. A camera can
stream successfully while recording never starts. Recording requests are
ignored unless HomeKit has first enabled recording and supplied a recording
configuration. After that, the plugin depends on motion/doorbell events to
cause HomeKit to request a recording. This explains reports where live view
works but no clips are saved.

The public issue history contains reports of:

- motion events not arriving until Ring notification credentials were refreshed;
- return-audio UDP binding failures on otherwise working live streams.

The battery-camera and Raspberry Pi reports are intentionally excluded from the
supported reliability assessment. The v15 refactor directly addresses some
stale-process behavior, but trigger delivery, negotiation, and codec behavior
remain first-class risks even on supported always-powered premium deployments.

## Prioritized improvement roadmap

### P0 - Make recording observability explicit

Add a per-camera HKSV state machine and counters for:

- recording enabled/configured;
- motion/doorbell event received;
- HomeKit recording request received;
- queue wait, Ring call acquisition, SDP answer, FFmpeg start;
- first MP4 fragment emitted;
- normal completion, cancellation, stall, timeout, and FFmpeg exit code.

Expose a rate-limited diagnostic summary in debug logs and a Homebridge UI
status view. Without this, users cannot distinguish HomeKit configuration,
Ring notification, network, codec, and parser failures.

### P0 - Add a deterministic health/self-test path

Provide a diagnostic command or UI action that validates, without requiring a
real motion event:

1. effective normalized media configuration;
2. FFmpeg executable and required encoders;
3. UDP port allocation and local bind address;
4. Ring media call and SDP negotiation;
5. one short fragmented-MP4 recording fixture.

Report a structured failure reason. A successful live-view test alone is not
sufficient.

### P1 - Test the actual end-to-end recording contract

Expand beyond the current 9 helper/lifecycle tests with deterministic fixtures
for:

- HomeKit recording enable/configuration callbacks;
- motion and doorbell events leading to a recording request;
- delayed/missing notification behavior;
- CRLF and variant Ring SDP inputs;
- FFmpeg startup failure, malformed output, no-output stall, and non-zero exit;
- cancellation at every stage, including while queued and while awaiting SDP;
- multiple cameras under memory and concurrency limits;
- fragments delivered in partial and oversized chunks.

The highest-value test is a fake Ring RTP source feeding a real or fixture
FFmpeg process and asserting that HomeKit receives valid initialization and
media fragments.

### P1 - Separate capability selection from capability verification

`auto` probes encoder names, but explicit codec settings are accepted without
verification and encoder presence does not prove that the encoder can process
the camera's input/profile. Validate the selected codec at startup, fail with
an actionable message, and optionally fall back from hardware encoding to
`libx264` or `copy` according to a documented policy.

### P1 - Improve media negotiation resilience

Treat SDP parsing as a protocol boundary rather than string rewriting:

- parse and rewrite media sections while preserving required session-level
  attributes;
- support CRLF and formatting variations;
- validate that the expected audio/video sections and ports were rewritten;
- distinguish audio-only from video recording explicitly;
- log the selected payload types, codec, address, and ports without secrets.

This would reduce failures caused by Ring-side SDP changes and make FFmpeg
errors diagnosable.

### P1 - Make resource protection system-aware for supported hosts

The queue bounds recording concurrency and buffered bytes, but it does not
bound total process memory, CPU, or aggregate live-stream/transcoder load.
Add configurable global byte/process budgets, reject or defer work before
starting FFmpeg, and record high-water marks. Do not add low-power tuning
profiles; supported hosts should have sufficient headroom for the selected
premium cameras.

### P2 - Treat notification health as a dependency

Add notification freshness and event-source diagnostics. When history fallback
is enabled, report whether events are coming from push or polling and whether
the event was deduplicated. Surface a clear remediation message when no Ring
events arrive for a configured interval, including token/push-credential
refresh guidance.

### P2 - Harden configuration and rollout

Clamp derived bitrate/buffer values, validate codec/profile combinations, and
include the effective configuration in diagnostics. Use staged releases for
media changes, retain a stable fallback profile, and publish a tested matrix
for always-powered premium camera models and capable hosts.

## Bottom line

The v15 work materially improves bounded failure handling and is a credible
experimental foundation. It should not yet be marketed as equivalent to
Scrypted or native Ring recording. The next major reliability gain will come
from observability and end-to-end tests, not another isolated FFmpeg cleanup
patch: users currently lack proof of where the recording chain stopped.
