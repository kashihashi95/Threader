# Audio Focus & Interruption Matrix: Redder Playback Engine

## 1. Overview
On Android, multiple applications can simultaneously request to play audio. To prevent jarring acoustic collisions (e.g., reading Reddit comments over a phone call or blasting sound when Bluetooth disconnects), Redder implements a strict **Audio Focus & Interruption Policy** aligned with AndroidX Media3 standards and Android 12+ audio routing rules.

---

## 2. Audio Attributes Specification

Redder requests audio focus using speech-optimized attributes configured on `ExoPlayer`:

```kotlin
val audioAttributes = AudioAttributes.Builder()
    .setContentType(C.AUDIO_CONTENT_TYPE_SPEECH)
    .setUsage(C.USAGE_MEDIA)
    .build()

exoPlayer.setAudioAttributes(audioAttributes, /* handleAudioFocus = */ true)
```

- **Content Type**: `AUDIO_CONTENT_TYPE_SPEECH` informs the OS and digital signal processors (DSPs) to optimize for vocal clarity and speech intelligibility rather than wide dynamic music ranges.
- **Usage**: `USAGE_MEDIA` binds volume adjustments to the hardware media volume stream.

---

## 3. System Interruption Decision Matrix

| System Event | OS Signal / Callback | Player Reaction | Volume / State Change | Post-Interruption Recovery |
| :--- | :--- | :--- | :--- | :--- |
| **Incoming Turn-by-Turn GPS Navigation Prompt** | `AUDIOFOCUS_LOSS_TRANSIENT_CAN_DUCK` | **Duck Playback** | Duck volume by **80%** (`player.volume = 0.2f`). Playback continues at low volume. | On `AUDIOFOCUS_GAIN`: Smoothly ramp volume back to `1.0f` over 300ms. |
| **Incoming Phone Call (Ringing / Ongoing)** | `AUDIOFOCUS_LOSS_TRANSIENT` | **Pause Player** | Immediate call to `player.pause()`. Record `resumeOnFocusGain = true`. Retain synthesis buffer. | On `AUDIOFOCUS_GAIN` (call ended): Automatically resume playback (`player.play()`). Reset `resumeOnFocusGain = false`. |
| **Third-Party Media App Starts (Spotify / YouTube)** | `AUDIOFOCUS_LOSS` (Permanent) | **Abandon & Stop** | Call `player.stop()`. Persist current thread URL and comment index to local storage. Release audio focus. Stop foreground service after 30s. | Do **not** auto-resume. User must explicitly tap Play in Redder HUD or notification. |
| **Hardware Disconnect (Bluetooth unpairs / Wired headphones unplugged)** | `AudioManager.ACTION_AUDIO_BECOMING_NOISY` broadcast | **Immediate Pause** | BroadcastReceiver intercepts intent and immediately pauses `ExoPlayer` (`player.pause()`). | Do **not** auto-resume on re-connection. User must explicitly press play to avoid unwanted sound projection. |
| **Google Assistant / Siri / Voice Input Activated** | `AUDIOFOCUS_LOSS_TRANSIENT` | **Pause Player** | Pause speech playback and pause microphone/TTS interactions. | On `AUDIOFOCUS_GAIN`: Resume playback automatically. |
| **Alarm Clock Ringing** | `AUDIOFOCUS_LOSS_TRANSIENT` | **Pause Player** | Alarm has higher priority (`USAGE_ALARM`). Redder pauses immediately. | On `AUDIOFOCUS_GAIN`: Resume playback when alarm is dismissed. |
| **Notification Ping (Short alert sound)** | Handled by OS AudioPolicy | **Ignored / Momentary Duck** | For standard pings (<1.5s), OS ducks media stream briefly at hardware mixer level without pausing player. | Handled transparently by hardware HAL. |

---

## 4. BroadcastReceiver Implementation: `BecomingNoisyReceiver`

```kotlin
class BecomingNoisyReceiver(
    private val onNoiseDetected: () -> Unit
) : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        if (intent.action == AudioManager.ACTION_AUDIO_BECOMING_NOISY) {
            onNoiseDetected()
        }
    }

    companion object {
        fun createFilter() = IntentFilter(AudioManager.ACTION_AUDIO_BECOMING_NOISY)
    }
}
```

Registered dynamically in `RedderAudioService.onCreate()` when playback begins, and unregistered in `onDestroy()` or when playback stops, preventing battery drain and intent leakage.
