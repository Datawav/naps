NAP-CAPTURE
===========

Shell-Mediated Microphone Recording
-----------------------------------

`draft`

**NAP ID:** NAP-CAPTURE
**Domain:** `capture`
**Web binding (NIP-5D):** `window.napplet.capture` · `shell.supports("capture")`

## Description

NAP-CAPTURE lets a napplet ask its runtime to create a finite, user-approved
microphone recording. The runtime owns consent, device selection, platform
permission, capture, encoding, limits, indication, retention, and teardown.

The napplet receives encoded audio bytes, not microphone authority. It never
receives a device handle, stable device identifier, raw stream, recorder object,
or platform permission state.

NAP-CAPTURE has no dependency on another NAP. Passing a completed artifact to
`upload`, `storage`, `relay`, or another domain is a separate napplet action.

## Discovery

A napplet detects the capability with:

```text
shell.supports("capture")
```

A runtime that reports support MUST implement every baseline operation in this
document.

`info()` reports coarse output formats and policy limits. It MUST NOT reveal
stable device identifiers, device labels, device counts, or platform permission
state.

## API Surface

| Operation | Parameters | Result | Wire |
|---|---|---|---|
| `info` | — | `CaptureInfo` | `capture.info` → `capture.info.result` |
| `start` | `CaptureRequest` | `CaptureSession` | `capture.start` → `capture.start.result` |
| `status` | `captureId` | `CaptureStatus` | `capture.status` → `capture.status.result` |
| `stop` | `captureId` | `CaptureArtifact` | `capture.stop` → `capture.stop.result` |
| `cancel` | `captureId` | `CaptureStatus` | `capture.cancel` → `capture.cancel.result` |
| `release` | `captureId` | acknowledgement | `capture.release` → `capture.release.result` |
| `onEvent` | handler | subscription cancellation handle | receives `capture.changed` |

`status` is authoritative. `onEvent` is advisory; a missed event never loses a
terminal artifact or terminal error.

### `CaptureInfo`

| Field | Required | Type | Notes |
|---|---|---|---|
| `sources` | yes | list of text | MUST contain only `"microphone"` in this version. |
| `mimeTypes` | yes | list of text | MIME types the runtime can produce, in runtime preference order. |
| `maxDurationMs` | yes | integer or null | Coarse duration limit; `null` means undisclosed. |
| `maxBytes` | yes | integer or null | Coarse encoded-byte limit; `null` means undisclosed. |

`CaptureInfo` is advisory. Policy, device availability, and platform permission
can still make a later `start` fail.

### `CaptureRequest`

| Field | Required | Type | Notes |
|---|---|---|---|
| `source` | yes | text | MUST be `"microphone"`. |
| `mimeTypes` | no | list of text | Acceptable output MIME types in caller preference order. |

An omitted or empty `mimeTypes` list lets the runtime choose any advertised
format. When a non-empty list is supplied, the runtime MUST choose an item from
that list or fail with `unsupported-format`.

The request has no device selector or low-level audio constraints. A runtime MAY
offer device and quality choices in trusted shell UI.

### `CaptureSession`

| Field | Required | Type | Notes |
|---|---|---|---|
| `captureId` | yes | text | Opaque, runtime-generated identifier scoped to the requesting napplet identity. |
| `source` | yes | text | `"microphone"`. |
| `mimeType` | yes | text | Actual output MIME type selected by the runtime. |

A successful `start` result means consent succeeded and recording is active. A
runtime MUST NOT return a `CaptureSession` while consent is pending.

### `CaptureArtifact`

| Field | Required | Type | Notes |
|---|---|---|---|
| `captureId` | yes | text | Identifier of the completed capture. |
| `data` | yes | any | Projection-specific binary value containing complete encoded audio bytes; never text or base64. |
| `mimeType` | yes | text | Actual MIME type of `data`. |
| `size` | yes | integer | Exact byte length of `data`. |
| `durationMs` | yes | integer | Recorded duration measured by the runtime. |
| `truncated` | yes | boolean | `true` when a limit or interruption ended capture. |
| `reason` | yes | `"requested"`, `"user"`, `"limit"`, or `"interrupted"` | Terminal cause. |

`size` and `durationMs` MUST be non-negative. `size` MUST equal the byte length of
`data`. The runtime MUST report the actual container and codec in `mimeType`; it
MUST NOT merely echo the request.

### `CaptureStatus`

| Field | Required | Type | Notes |
|---|---|---|---|
| `captureId` | yes | text | Affected capture. |
| `state` | yes | `"recording"`, `"completed"`, `"cancelled"`, or `"failed"` | Current monotonic state. |
| `artifact` | completed only | `CaptureArtifact` | Retained terminal artifact. |
| `reason` | cancelled only | `"user"`, `"policy"`, or `"revoked"` | Stable cancellation reason. |
| `error` | failed only | `CaptureError` | Retained normalized failure. |

A terminal `CaptureStatus` remains available until `release`, napplet unload, or
identity revocation. A runtime MUST reject a new `start` for that identity with
`busy` while any capture remains recording or unreleased. This bounds retained
sensitive data to one capture per identity.

### `CaptureEvent`

| Field | Required | Type | Notes |
|---|---|---|---|
| `captureId` | yes | text | Affected capture. |
| `state` | yes | `"completed"`, `"cancelled"`, or `"failed"` | New terminal state. |
| `reason` | cancelled only | `"user"`, `"policy"`, or `"revoked"` | Stable cancellation reason. |
| `error` | failed only | `CaptureError` | Normalized failure. |

`capture.changed` carries `CaptureEvent`. It is a wake-up hint only. The napplet
uses `status` to retrieve the authoritative terminal state and artifact.

### `CaptureError`

| Field | Required | Type | Notes |
|---|---|---|---|
| `code` | yes | text | Stable error code from the table below. |
| `message` | no | text | Safe diagnostic text; MUST NOT expose device identifiers or platform internals. |

## Lifecycle

Each runtime MUST reserve at most one pending, recording, or unreleased capture
slot per requesting napplet identity. It MAY enforce a stricter runtime-wide
limit. The slot is reserved before trusted consent UI opens. A concurrent
`start` for that identity fails with `busy`. Denial, cancellation, or start
failure releases a pending reservation.

```text
                start approved
     absent  -------------------->  recording
                                        |
                           stop/runtime | cancel
                               finalize | discard
                                        v
                    completed / cancelled / failed
                                        |
                                     release
                                        v
                                      absent
```

Rules:

1. `start` begins consent and permission work. It creates a session only after
   consent succeeds and the microphone is active.
2. `status` returns the current state. For `completed`, it returns the retained
   artifact. For `cancelled` or `failed`, it returns the terminal reason.
3. `stop` on `recording` finalizes audio, retains the artifact, and returns it.
   `stop` on `completed` is idempotent and returns the same retained artifact.
   `stop` on `cancelled` or `failed` fails with `invalid-state`; the caller uses
   `status` to retrieve the terminal reason.
4. `cancel` on `recording` stops capture, destroys audio bytes, retains a
   `cancelled` status, and returns that status. `cancel` on a terminal capture is
   idempotent: it returns the retained status and does not alter or erase it.
5. `release` is valid only for a terminal capture. It erases retained status and
   audio. Later operations on that identifier fail with `capture-not-found`.
6. A runtime-enforced duration or byte limit SHOULD finalize recoverable audio,
   retain a `completed` status with `truncated: true` and `reason: "limit"`, and
   emit `capture.changed`.
7. Device loss or permission revocation MAY retain recoverable audio as
   `completed` with `reason: "interrupted"`; otherwise it retains `failed`.
8. Runtime/user/policy cancellation destroys audio and retains `cancelled`.
9. A caller request and a runtime action can race. The first terminal transition
   processed by the runtime wins. Later terminal operations follow rules 3–5 and
   MUST NOT create a second artifact or change terminal state.
10. On napplet unload or identity revocation, the runtime MUST stop capture and
    erase every retained status and artifact owned by that identity. It MUST NOT
    continue recording in the background.

Every transition to `completed`, `cancelled`, or `failed` emits exactly one
`capture.changed` event while the napplet endpoint remains present. Event loss
is harmless because `status` is authoritative.

A napplet cannot programmatically cancel consent before `start` returns because
no `captureId` exists yet. The runtime MUST cancel pending consent when the
napplet unloads; trusted shell UI MUST let the user cancel it directly.

Pause/resume is not a baseline v1 operation.

## Wire Protocol

All request messages carry a caller-generated correlation `id`. Result or error
messages echo that `id`. `captureId` is generated by the runtime and scoped to
the requesting `(dTag, aggregateHash)` identity.

| Type | Direction | Payload fields |
|---|---|---|
| `capture.info` | napplet → runtime | `id` |
| `capture.info.result` | runtime → napplet | `id`, `info` |
| `capture.start` | napplet → runtime | `id`, `request` |
| `capture.start.result` | runtime → napplet | `id`, `session` |
| `capture.status` | napplet → runtime | `id`, `captureId` |
| `capture.status.result` | runtime → napplet | `id`, `status` |
| `capture.stop` | napplet → runtime | `id`, `captureId` |
| `capture.stop.result` | runtime → napplet | `id`, `artifact` |
| `capture.cancel` | napplet → runtime | `id`, `captureId` |
| `capture.cancel.result` | runtime → napplet | `id`, `status` |
| `capture.release` | napplet → runtime | `id`, `captureId` |
| `capture.release.result` | runtime → napplet | `id`, `captureId` |
| `capture.changed` | runtime → napplet | `event` |
| `capture.error` | runtime → napplet | `id`, `error` |

### Examples

Discover formats and limits:

```json
{ "type": "capture.info", "id": "cap-info-1" }
```

```json
{
  "type": "capture.info.result",
  "id": "cap-info-1",
  "info": {
    "sources": ["microphone"],
    "mimeTypes": ["audio/webm;codecs=opus", "audio/mp4"],
    "maxDurationMs": 600000,
    "maxBytes": 52428800
  }
}
```

Start recording:

```json
{
  "type": "capture.start",
  "id": "cap-start-1",
  "request": {
    "source": "microphone",
    "mimeTypes": ["audio/webm;codecs=opus", "audio/mp4"]
  }
}
```

```json
{
  "type": "capture.start.result",
  "id": "cap-start-1",
  "session": {
    "captureId": "capture_7f2a",
    "source": "microphone",
    "mimeType": "audio/webm;codecs=opus"
  }
}
```

Stop and retain the artifact:

```json
{
  "type": "capture.stop",
  "id": "cap-stop-1",
  "captureId": "capture_7f2a"
}
```

```text
{
  "type": "capture.stop.result",
  "id": "cap-stop-1",
  "artifact": {
    "captureId": "capture_7f2a",
    "data": <binary audio/webm 18234 bytes>,
    "mimeType": "audio/webm;codecs=opus",
    "size": 18234,
    "durationMs": 2410,
    "truncated": false,
    "reason": "requested"
  }
}
```

Runtime-driven completion emits a hint:

```json
{
  "type": "capture.changed",
  "event": {
    "captureId": "capture_7f2a",
    "state": "completed"
  }
}
```

The napplet then calls `capture.status`; the completed result contains the
retained `artifact`.

## Runtime Behavior

A conforming runtime:

- MUST own microphone access, device selection, encoding, retention, and teardown.
- MUST attribute every request and `captureId` to the requesting napplet identity.
- MUST obtain explicit user approval for every capture session through an action
  attributable to the user. Manifest requirements and prior platform permission
  are not themselves consent to record.
- MUST render trusted shell-controlled indication while microphone capture is
  active, including the requesting napplet identity and a shell-level stop or
  cancel control.
- MUST keep the indicator active while the platform microphone remains active.
- MUST normalize platform errors and keep low-level device diagnostics local.
- MUST enforce duration, byte, prompt-rate, and concurrency policy independently
  of caller input.
- MUST stop capture promptly when the user, platform, shell policy, or napplet
  lifecycle revokes authority.
- MUST erase cancelled, released, failed, and unloaded capture bytes.
- MUST NOT upload, publish, sign, relay, transcribe, or persist beyond this
  lifecycle as a side effect of capture.

The runtime MAY provide trusted device selection, quality controls, previews, or
retention warnings in shell UI. Those controls do not alter this contract.

## Errors

| Code | Meaning |
|---|---|
| `policy-denied` | Runtime policy denied capture before consent. |
| `user-cancelled` | The user cancelled or declined trusted capture UI. |
| `permission-denied` | Platform microphone permission denied or unavailable to the runtime. |
| `device-unavailable` | No usable microphone exists, or the selected device became unavailable. |
| `unsupported-source` | `source` is not `"microphone"`. |
| `unsupported-format` | The runtime cannot produce an acceptable requested MIME type. |
| `busy` | A pending, recording, or unreleased capture already reserves the identity slot, or the runtime is at its limit. |
| `capture-not-found` | The identifier is unknown or belongs to another napplet. |
| `invalid-state` | The requested operation is invalid for the retained state. |
| `quota-exceeded` | The runtime cannot retain enough bytes to complete capture. |
| `capture-failed` | Capture or encoding failed for another normalized reason. |

These correspond respectively to policy denial, user cancelled, permission
denied, device unavailable, unsupported input, unsupported output, concurrency,
identifier, invalid state, storage, and capture failures. A runtime MUST avoid
exposing whether a foreign `captureId` exists; use `capture-not-found` for both
unknown and foreign identifiers.

## Security Considerations

Microphone access can expose private speech and ambient audio. The runtime is the
security boundary and MUST mediate it as a visible, revocable capability.

- Platform permission is not napplet consent. A prior platform grant MUST NOT
  authorize silent capture by a newly loaded napplet.
- Consent UI and active-capture indication MUST be rendered by trusted shell UI,
  not supplied solely by the napplet.
- Device identifiers, labels, counts, and permission state MUST NOT cross this API.
- Capture identifiers MUST be unguessable, identity-scoped, and unusable by
  other napplets.
- Runtime limits MUST prevent unbounded recording, memory growth, repeated
  prompt abuse, and concurrent microphone monopolization.
- Completed bytes are sensitive. They MUST be delivered only to the requesting
  identity and erased on `release`, unload, or identity revocation.
- Live audio, waveform, and level streaming are excluded. They add continuous
  listening and high-frequency transport semantics not needed here.

This NAP does not alter the web projection. Web runtimes remain governed by
[NIP-5D](https://github.com/nostr-protocol/nips/pull/2303), including its sandbox
and identity rules. Projection guidance: web runtimes should carry the artifact's
binary value by structured clone, not base64, and should use an idiomatic binary
value such as `Blob` or `ArrayBuffer`.

## Non-Goals

NAP-CAPTURE v1 does not define:

- camera, screen, window, tab, system-audio, or mixed-source capture;
- device enumeration, device selection APIs, or persistent device preferences;
- raw audio streams, chunks, waveform data, level meters, or WebRTC tracks;
- pause/resume;
- editing, transcoding, transcription, upload, signing, publishing, or relay use;
- durable recovery after napplet unload;
- background recording after napplet unload; or
- any change to a projection's sandbox policy.

## Implementations

None yet. A runtime prototype demonstrating consent, trusted indication,
binary delivery, terminal retention, limit handling, and unload cleanup is
required before this draft should be considered mergeable.

## Changelog

- Initial draft pending commit identifier.
