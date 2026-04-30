# AGENTS.md

This file gives coding agents the project context and implementation rules for this repository.

The detailed product and architecture design lives in:

- `encrypted_video_player_design.md`

When in doubt, follow that design document first.

## Project Summary

This project is an encrypted video player system.

The first target platform is Windows desktop. The backend should be written in Rust and exposed as REST APIs. The client should be written in C++/Qt and use libmpv for playback.

The first version is not intended to be full commercial DRM. It should implement a practical protected video playback flow:

- Upload video.
- Transcode/package video into HLS AES-128.
- Store encrypted HLS assets in object storage.
- Authenticate users before playback.
- Issue short-lived play tokens.
- Dynamically return m3u8 playlists.
- Protect the HLS key endpoint.
- Let libmpv request HLS segments and AES keys and perform playback.
- Overlay dynamic user watermark in the Qt client.
- Record key requests and playback events.

## Core Architecture

Recommended MVP stack:

```text
Backend language: Rust
Backend framework: axum + tokio
Database: PostgreSQL + sqlx
Object storage: MinIO using S3-compatible APIs
Packaging/transcoding: FFmpeg
Playback protocol: HLS AES-128
Desktop client: C++ / Qt
Playback engine: libmpv
Auth/token: PASETO or JWT
Watermark: Qt transparent overlay
```

Optional later components:

```text
Redis
CDN
Vault or cloud KMS
NATS JetStream
DASH/CENC
Widevine / PlayReady / FairPlay
Stronger client hardening
```

Do not add optional later components unless the user explicitly asks or the current implementation stage clearly requires them.

## Component Responsibilities

### Rust Backend API

Responsible for:

- User login and authentication.
- Video metadata APIs.
- Playback authorization.
- Short-lived play token issuance.
- Dynamic m3u8 response.
- HLS key endpoint authorization.
- Playback event ingestion.
- Device registration.
- Upload task creation.
- Transcode job status APIs.

The backend should not permanently serve high-volume video segment traffic in production. For MVP or local development it may proxy simple assets, but the intended production path is object storage/CDN for segments.

### Rust Worker

Responsible for:

- Processing video packaging/transcoding jobs.
- Calling FFmpeg.
- Generating per-video AES content keys.
- Producing HLS AES-128 playlists and segments.
- Uploading generated assets to MinIO.
- Updating job and video status in PostgreSQL.

Video packaging must be asynchronous. Do not block an upload HTTP request waiting for FFmpeg to finish.

### PostgreSQL

Stores:

- Users.
- Videos.
- Playback permissions.
- Video key records.
- Device records.
- Transcode jobs.
- Playback sessions.
- Playback events.
- Key request logs.

Do not store video binary data in PostgreSQL.

### MinIO / Object Storage

Stores:

- Original uploaded video.
- HLS m3u8 template or generated playlist files.
- Encrypted HLS `.ts` or `.m4s` segments.
- Covers.
- Subtitles.

Raw source videos must not be public.

HLS keys must not be stored as public objects.

### Qt Client

Responsible for:

- Login UI.
- Video list UI.
- Calling backend business REST APIs via Qt networking.
- Requesting playback authorization.
- Passing the returned play URL to libmpv.
- Showing dynamic watermark over the playback area.
- Reporting playback events.
- Handling playback errors and token expiry.

### libmpv

Responsible for:

- Loading the HLS m3u8 URL.
- Parsing standard HLS playlists.
- Requesting HLS key URLs declared by `#EXT-X-KEY`.
- Downloading encrypted segments.
- Performing HLS AES-128 decryption internally.
- Decoding and rendering audio/video.

Do not implement a custom HLS parser or AES playback pipeline in the first version.

## Important Playback Facts

For standard HLS AES-128, libmpv/FFmpeg can parse this:

```m3u8
#EXT-X-KEY:METHOD=AES-128,URI="https://api.example.com/api/v1/videos/v_123/key?play_token=xxx"
```

Then it will:

- Request the key URL.
- Expect the response body to be raw 16-byte AES key data.
- Download encrypted segments.
- Decrypt and play internally.

The key endpoint must not return JSON.

Correct key response characteristics:

```http
Content-Type: application/octet-stream
Cache-Control: no-store
Pragma: no-cache
```

The first version should put `play_token` in the m3u8 and key URL query string for compatibility with libmpv. Business APIs should still use normal authorization headers.

## Security Boundary

HLS AES-128 is not commercial DRM.

The client must obtain a usable key or usable decrypted data to play video, so the system cannot fully prevent professional memory key extraction.

The system should instead:

- Keep play tokens short-lived.
- Bind play tokens to `user_id`, `video_id`, `device_id`, `play_session_id`, and expiry.
- Use separate content keys per video.
- Dynamically generate or rewrite m3u8 responses.
- Protect the key endpoint with authorization.
- Record key requests.
- Add rate limits and anomaly detection.
- Overlay dynamic user watermark.
- Use HTTPS.
- Avoid caching key responses.

Later security enhancements may include:

- Redis-backed session state.
- Key request sliding window by playback position.
- Multiple keys per video, for example one key per N segments or N minutes.
- CDN signed URLs.
- Vault/KMS for content key protection.
- Client certificate pinning.
- Client integrity checking.
- Basic anti-debugging.
- Commercial DRM for high-value content.

Do not claim that the MVP prevents all key extraction or all screen recording.

## MVP Scope

MVP should include:

- Rust axum REST API.
- PostgreSQL schema/migrations.
- MinIO integration.
- Video upload.
- FFmpeg HLS AES-128 packaging.
- Video status management.
- User login.
- Video list.
- Playback authorization.
- Short-lived play token.
- Dynamic m3u8 response.
- Authorized key endpoint.
- Qt + libmpv playback on Windows.
- Qt dynamic visible watermark.
- Playback event reporting.
- Key request logging.

MVP should not include unless explicitly requested:

- CDN.
- Commercial DRM.
- Offline encrypted cache.
- Strong device fingerprinting.
- Heavy client packer/virtualization.
- Large distributed transcode scheduler.
- Multi-platform clients.
- DASH/CENC.

## Recommended API Shape

Use REST APIs.

Suggested routes:

```text
POST /api/v1/auth/login
POST /api/v1/auth/logout
POST /api/v1/auth/refresh
GET  /api/v1/me

GET  /api/v1/videos
GET  /api/v1/videos/{video_id}
POST /api/v1/videos/{video_id}/play

GET  /api/v1/videos/{video_id}/hls/index.m3u8?play_token=...
GET  /api/v1/videos/{video_id}/key?play_token=...

POST /api/v1/playback/events

POST /api/v1/devices/register
GET  /api/v1/devices
DELETE /api/v1/devices/{device_id}
```

Playback authorization should return:

```json
{
  "play_url": "https://api.example.com/api/v1/videos/v_123/hls/index.m3u8?play_token=xxx",
  "play_token": "xxx",
  "expires_at": "2026-04-30T12:30:00Z",
  "watermark": {
    "text": "u_123 138****1234",
    "opacity": 0.22,
    "move_interval_sec": 12
  }
}
```

## Suggested Development Order

1. Create Rust axum backend skeleton.
2. Add PostgreSQL and migrations.
3. Add users, videos, permissions, key records, jobs, playback sessions, playback events.
4. Add MinIO integration.
5. Implement video upload.
6. Implement worker that calls FFmpeg.
7. Generate HLS AES-128 output.
8. Implement video status transitions.
9. Implement login and access token.
10. Implement video list.
11. Implement playback authorization and play token.
12. Implement dynamic m3u8 response.
13. Implement authorized key endpoint returning raw key bytes.
14. Implement Qt login and video list.
15. Integrate libmpv playback.
16. Implement Qt dynamic watermark overlay.
17. Implement playback event reporting.
18. Add key request logging, no-store response headers, and basic rate limiting.
19. Add detection for token reuse across IPs/devices.

## Implementation Rules For Agents

- Prefer the existing design in `encrypted_video_player_design.md`.
- Keep the first version simple and shippable.
- Do not replace libmpv with a custom FFmpeg playback pipeline unless explicitly requested.
- Do not implement fake DRM claims.
- Do not expose HLS keys as public static files.
- Do not return HLS keys as JSON.
- Do not store raw video binary in PostgreSQL.
- Do not make FFmpeg packaging synchronous inside the upload request.
- Use structured APIs and clear service boundaries.
- Keep security-sensitive token and key logic centralized in backend services.
- Add tests around token validation, permission checks, and key endpoint behavior when code exists.

## Terminology

- `content key`: AES key used to encrypt HLS media segments.
- `play token`: short-lived token authorizing one playback session.
- `play_session_id`: identifier for one playback authorization/session.
- `m3u8`: HLS playlist file.
- `segment`: encrypted HLS media chunk.
- `key endpoint`: backend endpoint returning raw HLS AES key bytes after authorization.
- `dynamic m3u8`: m3u8 generated or rewritten by backend to include session-specific key URLs.
