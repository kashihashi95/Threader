# ADR-002: Process/Service Decoupling of Floating HUD from MediaSessionService

## Status
Accepted

## Date
2026-09-20

## Context
Redder provides two primary user-facing touchpoints during audio playback:
1. **Interactive Floating HUD**: An on-screen overlay view (`SYSTEM_ALERT_WINDOW`) displaying current speaker avatars, dialogue text scrubbing, play/pause controls, and skip buttons directly above third-party applications (e.g. Reddit, browser).
2. **Audio Playback & Media Session**: A background audio player handling continuous playback, lockscreen controls, notification drawer media controls, hardware volume keys, Bluetooth headphones/Android Auto commands, and hardware audio focus.

An initial monolithic proposal combined the `WindowManager` overlay and the `ExoPlayer` audio loop into a single Android `Service` declared as `FOREGROUND_SERVICE_TYPE_MEDIA_PLAYBACK`.

This monolithic approach introduced critical platform bugs and UX failure modes:
- **Coupled Lifecycles**: If a user swipes away or closes the floating HUD overlay to declutter their screen, the entire Service is destroyed, immediately terminating audio playback. In standard media apps (e.g., YouTube Music, Pocket Casts), audio is expected to persist in the background with lockscreen/notification controls even when visual UI is closed.
- **Android 14+ (API 34/35) Foreground Service Enforcement**: Android 14 mandates strict runtime type validations for foreground services. A service declared with `FOREGROUND_SERVICE_TYPE_MEDIA_PLAYBACK` must link directly to an active `MediaSession` with an ongoing media notification. Embedding non-media UI loops (`WindowManager.addView()`, gesture detectors, Compose view hierarchies) into this service increases memory leak risks and causes unexpected lifecycle terminations.
- **Permission Asymmetry**: `android.permission.SYSTEM_ALERT_WINDOW` is a "special permission" requiring explicit user navigation to system settings, whereas `POST_NOTIFICATIONS` is an ordinary runtime permission. If the user denies or revokes overlay permissions, a monolithic service fails to start entirely, preventing the user from using audio playback.

## Decision
We decouple the presentation overlay and audio engine into **two distinct, independently managed services** communicating via standard AndroidX Media3 IPC primitives:

1. **`RedderAudioService` (`androidx.media3.session.MediaSessionService`)**:
   - Manages `ExoPlayer`, `MediaSession`, hardware audio focus, and the foreground `MediaStyle` notification.
   - Declared as `FOREGROUND_SERVICE_TYPE_MEDIA_PLAYBACK`.
   - Exposes a `MediaSession` token that any local or remote controller can bind to.
   - Completely headless; has zero dependencies on `WindowManager` or visual UI.

2. **`FloatingOverlayService` (`android.app.Service`)**:
   - Manages the `ComposeView` floating HUD attached to `WindowManager` via `WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY`.
   - Binds to `RedderAudioService` using a standard `MediaController` (`MediaController.Builder(context, sessionToken)`).
   - Translates HUD touch events (play, pause, skip, scrub) into standard `MediaController` commands.
   - Observes `Player.Listener` state updates (playback state, metadata, current item) emitted by the `MediaSession`.
   - Can be shown, dismissed, or killed at will without affecting background audio playback.

## Alternatives Considered

### Monolithic Single Service
- **Pros**: Fewer service declarations in `AndroidManifest.xml`; single lifecycle callback.
- **Cons**: Dismissing the HUD kills audio; denies playback if overlay permission is missing; violates Android 14 FGS compliance guidelines; high memory footprint in media process.
- **Rejected**: Incompatible with standard Android media player expectations and Android 14 platform rules.

### Multi-Process Separation (`:overlay_process` vs `:media_process`)
- **Pros**: Crash in Compose overlay does not crash audio playback engine; separate memory heaps.
- **Cons**: Overhead of multi-process Android apps (duplicate ART runtime initialization overhead, $\sim30\text{ MB}$ extra baseline RAM, complex custom AIDL or IPC channels for non-media data).
- **Rejected**: Unnecessary complexity for Phase 1. Decoupled services running in the same process communicate seamlessly via Media3 `MediaController` without cross-process IPC serialization penalties.

## Consequences

### Positive
- **Independent Lifecycle**: Audio persists cleanly when the user minimizes or closes the floating HUD.
- **Graceful Permission Degradation**: If `SYSTEM_ALERT_WINDOW` permission is denied, `RedderAudioService` still launches normally, providing full playback via lockscreen and notification drawer controls.
- **Strict Android 14 Compliance**: `RedderAudioService` satisfies all `FOREGROUND_SERVICE_TYPE_MEDIA_PLAYBACK` requirements without leaking window manager references.
- **Testability**: The audio engine can be unit-tested and instrumentation-tested without creating mock display windows or overlay permissions.

### Negative / Trade-offs
- Requires coordinating two services from `ShareReceiverActivity` (trampoline activity starts `RedderAudioService` first, and optionally `FloatingOverlayService` if overlay permission is granted).
- State synchronization between HUD and Player must flow through asynchronous `MediaController` callbacks.
