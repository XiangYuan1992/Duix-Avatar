# Realtime Avatar WebRTC Control Service Design

Date: 2026-06-03

## 1. Purpose

This spec defines a new realtime avatar control and playback service for the Duix-Avatar direction. The goal is to provide a browser-accessible control page that connects to a Python realtime inference service, starts text-driven avatar synthesis, and plays synchronized audio/video through WebRTC.

This spec intentionally does not define OBS integration behavior. If a user wants to capture the browser page in OBS, that is outside this system's scope.

## 2. Confirmed Scope

### In scope for MVP

- A Python service using `aiortc`.
- The WebRTC service and realtime inference module run in the same Python process.
- Access over intranet IP and port, without authentication.
- One page only: `GET /avatar/control`.
- A single active client/session at a time.
- A control page with:
  - Connect button.
  - Text input.
  - Start synthesis button.
  - Stop synthesis button.
  - Status display.
  - In-page WebRTC playback of synchronized audio and video.
- WebSocket-based signaling and session control.
- WebRTC output must include both video and audio tracks.
- Text synthesis is designed around segment-based streaming architecture.
- MVP may treat the submitted full text as one segment.
- Microphone audio input is reserved for future extension, not implemented in MVP.
- When no inference session is running, no inference audio/video should be emitted.
- Idle UI displays a placeholder/test pattern in the page layer.

### Out of scope for MVP

- OBS-specific URL parameters or capture behavior.
- Authentication and token validation.
- Public internet hardening.
- Multi-client viewing.
- Multi-session support.
- Microphone realtime driving.
- True per-keystroke incremental synthesis.
- Graceful stop that waits for the current segment to finish.
- Separate inference and WebRTC service processes.
- Direct local access to cloud `/dev/videoX` devices.

## 3. System Context

Current Duix-Avatar is task-oriented: it submits complete audio/video inputs to a backend, polls status, and receives generated video files. It is not a realtime frame-streaming architecture.

The new service is separate in responsibility: it should model realtime inference as a session that emits synchronized audio/video frames into WebRTC tracks.

## 4. High-Level Architecture

```text
Browser
  GET /avatar/control
  WS  /api/session
        |
        v
Python Realtime Avatar Service
  ├─ HTTP page server
  ├─ WebSocket session/signaling controller
  ├─ aiortc PeerConnection manager
  ├─ Text segmenter and segment queue
  ├─ Realtime inference session
  └─ WebRTC audio/video tracks
```

The service owns one active session at a time. The page first connects to the service. After the connection succeeds, the user submits text and starts synthesis. The inference session then emits synchronized audio/video data, which is sent through the established WebRTC connection.

## 5. HTTP Surface

### `GET /avatar/control`

Returns the control page.

The control page must include:

- Connection status.
- Media playback area.
- Text input area.
- Connect button.
- Start synthesis button.
- Stop synthesis button.
- Synthesis/session status.

No URL parameters are required or designed for MVP.

## 6. WebSocket Surface

### `WS /api/session`

The WebSocket is responsible for:

- Claiming the single active control session.
- Exchanging WebRTC offer/answer messages.
- Receiving start/stop commands.
- Sending session status events.
- Releasing the session when the page closes, refreshes, or disconnects unexpectedly.

### Single-session behavior

If there is already an active WebSocket session, a second client must not become active. The server should either reject the connection or accept it only long enough to send an error event such as `session.busy`, then close it.

## 7. WebSocket Message Types

All messages are JSON objects with a `type` field.

### Client to server

#### `webrtc.offer`

Sent after the user clicks Connect and the browser creates an SDP offer.

```json
{
  "type": "webrtc.offer",
  "sdp": "...",
  "sdpType": "offer"
}
```

#### `session.start`

Starts synthesis from submitted text.

```json
{
  "type": "session.start",
  "text": "要合成的整段文本"
}
```

The server must validate that:

- The WebRTC connection exists or is being established.
- The text is not empty.
- No synthesis session is already running.

#### `session.stop`

Stops the current synthesis session immediately.

```json
{
  "type": "session.stop"
}
```

### Server to client

#### `webrtc.answer`

Returned after the server accepts the browser offer.

```json
{
  "type": "webrtc.answer",
  "sdp": "...",
  "sdpType": "answer"
}
```

#### `session.status`

Reports state transitions.

```json
{
  "type": "session.status",
  "status": "connected"
}
```

Allowed MVP statuses:

- `idle`
- `connecting`
- `connected`
- `starting`
- `streaming`
- `stopping`
- `error`

#### `session.error`

Reports recoverable or fatal errors.

```json
{
  "type": "session.error",
  "code": "TEXT_EMPTY",
  "message": "Text must not be empty"
}
```

Recommended MVP error codes:

- `SESSION_BUSY`
- `INVALID_MESSAGE`
- `TEXT_EMPTY`
- `WEBRTC_NOT_READY`
- `SYNTHESIS_ALREADY_RUNNING`
- `INFERENCE_FAILED`
- `INTERNAL_ERROR`

#### `segment.status`

Reports segment queue progress. Even if MVP treats the full text as one segment, the status shape should be segment-based.

```json
{
  "type": "segment.status",
  "segmentId": "seg-1",
  "status": "streaming"
}
```

Allowed segment statuses:

- `queued`
- `starting`
- `streaming`
- `completed`
- `cancelled`
- `failed`

## 8. Control Page State Machine

```text
idle
  ↓ user clicks Connect
connecting
  ↓ WebSocket + WebRTC connected
connected
  ↓ user enters text and clicks Start
starting
  ↓ inference emits synchronized audio/video
streaming
  ↓ user clicks Stop
stopping
  ↓ inference cancelled and queue cleared
connected
  ↓ page closes, refreshes, or connection fails
idle
```

There is no manual disconnect button. Closing or refreshing the page releases the server-side active session.

## 9. Media Semantics

### Connected but not synthesizing

When the page is connected but synthesis has not started:

- The page shows a placeholder/test-pattern UI in the playback area.
- The inference module must not emit fake inference audio/video.
- The service may keep WebRTC tracks alive internally if needed, but these tracks must not be presented as successful inference output.

### Synthesizing

When synthesis starts:

- The service creates an inference session.
- The inference session emits synchronized audio/video frames.
- WebRTC audio and video tracks carry those frames to the browser.
- The control page plays the resulting stream.

### Audio requirement

MVP output must include audio and video. Video-only output does not satisfy this spec.

## 10. Text Synthesis Design

The architecture must be segment-based even if MVP uses a single segment.

### MVP behavior

```text
Submitted full text
  ↓
TextSegmenter emits one segment
  ↓
SegmentQueue runs that segment
  ↓
Inference session outputs synchronized audio/video
```

### Future behavior

The same interfaces should support:

- Splitting text by Chinese and English punctuation.
- Splitting text by maximum length.
- Queueing multiple segments.
- Preparing later segments while earlier segments stream.
- Reporting per-segment status.
- Graceful stop after the current segment finishes.

### Not planned in this spec

Per-keystroke incremental synthesis is not part of this design. The system should not restart synthesis on every text input change.

## 11. Stop Semantics

### MVP immediate stop

When the user clicks Stop:

- Cancel the active inference session immediately.
- Clear the segment queue.
- Stop audio output for the current synthesis.
- Return the UI to `connected` state.
- Keep the WebSocket and WebRTC connection available for another Start command.

### Future graceful stop

A later version may add graceful stop:

- Reject new segments.
- Let the current segment complete.
- Then return to `connected`.

## 12. Inference Module Boundary

Although inference and WebRTC run in the same Python process for MVP, the module boundary should remain explicit.

Recommended internal interfaces:

```text
TextSegmenter
  input: text
  output: ordered segments

SegmentQueue
  input: segments
  output: segment lifecycle events

InferenceSession
  input: segment text
  output: synchronized audio/video frames
  controls: start, cancel

MediaTrackBridge
  input: audio/video frames from inference
  output: aiortc AudioStreamTrack and VideoStreamTrack
```

The inference module must not block the asyncio event loop. Heavy work should run in an executor, worker thread, or process as needed.

## 13. Future Microphone Extension

Microphone input is not implemented in MVP, but the architecture should not block it.

Future microphone flow:

```text
Browser microphone
  ↓ WebRTC audio track or WebSocket audio chunks
Python service
  ↓ realtime inference session
Synchronized avatar audio/video output
  ↓ WebRTC playback
```

The MVP control page may show no microphone controls. If any placeholder UI is present, it must be clearly disabled or hidden to avoid implying support.

## 14. Deployment Assumptions

- The service runs on a cloud or intranet host.
- Users access it with `http://<ip>:<port>/avatar/control` during MVP.
- No TLS or authentication is required for MVP.
- Public internet deployment requires a later security review and likely HTTPS.

## 15. Acceptance Criteria

MVP is accepted when:

1. A user can open `http://<ip>:<port>/avatar/control`.
2. The page initially shows an idle state and placeholder playback area.
3. Clicking Connect establishes the single active session and WebRTC connection.
4. A second concurrent client cannot take over the active session.
5. Entering non-empty text and clicking Start creates a synthesis session.
6. The service sends synchronized audio and video over WebRTC while synthesis runs.
7. The page plays both audio and video.
8. Clicking Stop immediately cancels synthesis and returns the page to connected state.
9. Closing or refreshing the page releases the active session.
10. The internal text flow uses segment abstractions, even if MVP emits one segment for the full text.

## 16. Open Questions Not Resolved by This Spec

These are intentionally left for implementation planning or future specs:

- Exact realtime inference model API.
- Exact audio/video frame formats produced by the inference module.
- Target latency budget.
- Browser autoplay handling details.
- Production authentication.
- TLS/domain deployment.
- Multi-client viewing.
- OBS-specific usage guidance.
