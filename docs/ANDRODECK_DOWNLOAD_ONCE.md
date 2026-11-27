# AndroDeck — Download Once, Play Anytime

This document defines the AndroDeck product requirement for persistent runtime downloads.
It is a feature specification, not a claim that a new UI or runtime behavior has already shipped.

## Existing DroidDeck building blocks

The upstream code already contains important pieces that should be reused rather than duplicated:

- `core/Downloader.java`: file downloads, optional resume, progress callbacks, cancellation, and optional expected SHA-256 validation.
- `core/Hashes.kt`: SHA-256/SHA-512/MD5 helpers.
- `core/DownloadSource.kt`: source selection and mirror fallback.
- Existing runtime/component installation flows should remain the source of truth for which files are required and where they are installed.

Before changing installation flows, trace every runtime/component installer and its manifest format. Do not create a second download system or hard-code guessed URLs.

## User-facing promise

**Download Once – Play Anytime** means:

1. Download required runtime components once and retain verified installed files locally.
2. On later launches, reuse valid local components; do not download them again just because the app was reopened.
3. If a download is interrupted, resume only when the server supports byte ranges and the existing downloader's safety checks permit it.
4. Verify a trusted manifest checksum before marking a component ready.
5. If a checksum is missing, do not invent one or label the file cryptographically verified. Show an explicit unverified state and follow the existing upstream policy.
6. Repair only missing or invalid components; preserve already verified components.
7. Steam games remain managed by Steam and their own license/offline-mode rules. This promise applies to AndroDeck runtime files, not to every game's online requirements.

## Required component states

- `NOT_INSTALLED`: no usable local component.
- `DOWNLOADING`: transfer in progress.
- `PAUSED`: user paused or transfer interrupted; resumability must be rechecked.
- `VERIFYING`: checksum validation in progress.
- `READY`: installed and verified against a trusted expected digest.
- `READY_UNVERIFIED`: installed, but no trusted digest was supplied.
- `REPAIR_REQUIRED`: missing, truncated, or checksum-mismatched component.
- `ERROR`: failure with a recoverable explanation.

Persist state per component, not only as one global "download complete" flag.

## Safety and correctness requirements

- Use only official project manifests and trusted upstream download locations.
- Never bundle Steam client files or credentials in the APK.
- Never delete a working runtime before a replacement has downloaded and passed verification.
- Download to a temporary file, verify it, then atomically promote it to the installed path where the filesystem permits.
- Validate the expected SHA-256 format before starting a transfer.
- Treat HTTP 416, servers without Range support, changed ETags, and changed remote file sizes as reasons to restart safely rather than append potentially corrupt bytes.
- Re-check available storage before downloads and leave a safety margin.
- Keep downloads out of the gameplay foreground path; do not run unnecessary background jobs while a game is active.
- Respect Android storage permissions and scoped-storage constraints.
- Do not promise offline play for Steam games that require online authentication, servers, or DRM checks.

## Acceptance tests

1. First install downloads each required runtime component and verifies available trusted digests.
2. Closing and reopening AndroDeck reuses ready components without network transfer.
3. A supported interrupted transfer resumes and produces the exact expected digest.
4. A server without range support safely restarts the transfer.
5. A corrupted local component is marked `REPAIR_REQUIRED` and only that component is fetched again.
6. A missing checksum never produces a false `READY` verified state.
7. Insufficient storage blocks installation before partially replacing a working runtime.
8. A runtime update does not remove the previous working runtime until the new one is verified and installable.

## Branding and release status

AndroDeck is an independent community derivative based on DroidDeck. Preserve upstream copyright and license notices and comply with GPL-3.0 when distributing modified binaries/source. Do not imply endorsement by DroidDeck or Valve.

This specification does not itself implement the UI, wire existing installers to a persistent component registry, or prove an APK build. Those changes require code-level integration and automated/device testing.
