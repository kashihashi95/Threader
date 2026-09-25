# Google Play Policy & Platform Compliance Audit: Redder

## 1. Executive Summary
This document provides formal verification of compliance with Google Play Store Developer Program Policies, Android 14+ (API 34) and Android 15 (API 35) platform security standards, foreground service requirements, and user privacy protections.

---

## 2. Permission Declarations & Justifications

### A. `SYSTEM_ALERT_WINDOW` (Display Over Other Apps)
* **Declared In**: `AndroidManifest.xml`
* **Google Play Classification**: High-Risk / Special Access Permission.
* **Core Purpose**: Allows Redder to render a non-intrusive floating HUD overlay widget directly over third-party applications (e.g. Reddit official client, web browsers) so users can see the currently active speaker, scrub through dialogue, and control audio without switching away from their reading context.
* **User Consent Flow**:
  1. The app never requests or demands this permission during initial installation or splash screen.
  2. In `MainActivity`, an explanatory permission card describes the optional floating HUD feature.
  3. Tapping "Enable Floating HUD" directs the user to `Settings.ACTION_MANAGE_OVERLAY_PERMISSION` via explicit intent.
* **Graceful Degradation / Fallback Strategy**:
  - If `Settings.canDrawOverlays(context) == false`, the app **does not fail or crash**.
  - `FloatingOverlayService` is skipped entirely.
  - `RedderAudioService` launches in standard background mode, providing playback controls solely through the lockscreen and the notification drawer `MediaStyle` notification.

---

### B. `FOREGROUND_SERVICE_MEDIA_PLAYBACK` (Android 14+ API 34/35 Compliance)
* **Declared In**:
  ```xml
  <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
  <uses-permission android:name="android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK" />
  ```
* **Service Configuration**:
  ```xml
  <service
      android:name=".playback.RedderAudioService"
      android:foregroundServiceType="mediaPlayback"
      android:exported="false">
      <intent-filter>
          <action android:name="androidx.media3.session.MediaSessionService" />
      </intent-filter>
  </service>
  ```
* **Platform Invariants**:
  1. The service is started via `ContextCompat.startForegroundService()` from `ShareReceiverActivity` (an active Activity user interaction), satisfying Android 12+ background start restrictions (`ForegroundServiceStartNotAllowedException`).
  2. Inside `RedderAudioService.onCreate()`, `startForeground()` is invoked immediately within the required 5-second OS window.
  3. The ongoing notification is configured with `NotificationCompat.MediaStyle()` containing a valid `MediaSessionCompat` or AndroidX `MediaSession.token`.
  4. When playback is stopped and focus abandoned, `stopForeground(STOP_FOREGROUND_REMOVE)` and `stopSelf()` are called to prevent orphan foreground services.

---

## 3. Explicit Verification: Zero Accessibility Service Usage

> [!CAUTION]
> **Play Store Accessibility Policy Violation Prohibition**:
> Under Google Play's Accessibility Policy, `AccessibilityService` is strictly designated for apps assisting users with disabilities. Using Accessibility APIs to scrape UI text, intercept screens, or inspect content of third-party apps (such as the Reddit client) is a **Class 1 policy violation** resulting in immediate app removal and potential developer account termination.

* **Audit Finding**:
  - **Zero Accessibility Services Declared**: The Redder manifest contains **NO** `<service android:permission="android.permission.BIND_ACCESSIBILITY_SERVICE">`.
  - **Ingestion Mechanism**: Ingestion relies exclusively on standard Android public sharing intents:
    ```xml
    <activity
        android:name=".ui.ShareReceiverActivity"
        android:exported="true">
        <intent-filter>
            <action android:name="android.intent.action.SEND" />
            <category android:name="android.intent.category.DEFAULT" />
            <data android:mimeType="text/plain" />
        </intent-filter>
    </activity>
    ```
  - Content is retrieved cleanly over HTTPS using Reddit's public API endpoints, completely avoiding private screen scraping or DOM inspection.

---

## 4. Privacy, Data Safety & Store Compliance Disclosures

### A. Data Safety Section (Google Play Console Disclosures)
ThreadReader formally declares the following data safety profile in compliance with Google Play Developer Policies:

1. **On-Device Neural TTS (Zero Cloud AI Leakage)**
   * **Disclosure**: 100% on-device speech synthesis (Piper ONNX / Supertonic).
   * **Verification**: No voice recordings, input text, or audio tokens are transmitted to external AI servers (e.g. OpenAI, ElevenLabs, Google Cloud TTS).
   * **Data Category**: Audio Files / Voice Recordings $\to$ **NOT COLLECTED, NOT SHARED**.

2. **Network Ingestion (Public Decentralized Mirrors - Redlib)**
   * **Disclosure**: Ingestion queries public decentralized community mirrors (Redlib instances) and public Reddit endpoints.
   * **Verification**: Zero account harvesting. No user login credentials, passwords, OAuth tokens, or private browsing history are collected or stored.
   * **Boundaries**: Strict 5MB payload limit (`MAX_PAYLOAD_BYTES`) and SSRF protection (`UrlSecurityValidator`).
   * **Data Category**: Personal Info, Browsing History $\to$ **NOT COLLECTED, NOT SHARED**.

3. **Monetization & Advertising (AdMob & Pro Tier Suppression)**
   * **Disclosure**: Google AdMob uses advertising identifiers (Android Advertising ID / AAID) solely in the Free tier.
   * **Verification**: Completely suppressed when the user purchases the lifetime "Remove Ads" / Pro in-app purchase (`threadreader_remove_ads`).
   * **Data Category**: Device or other IDs $\to$ **COLLECTED & SHARED (Free Tier Only, Suppressed on Pro)**.

For full Play Console questionnaire answers and specifications, see [google-play-data-safety.md](file:///C:/Users/peouj/Redder/docs/specs/google-play-data-safety.md) and public policy [privacy-policy.md](file:///C:/Users/peouj/Redder/docs/privacy/privacy-policy.md).

---

## 5. Store Listing Assets Audit
- **App Icon (512x512)**: Native vector app launcher icon (`ic_launcher` Snoo headphones on Reddit orange) rendered to 512x512 PNG/JPG (`store_assets/app_icon_512.png` & `fastlane/metadata/android/en-US/images/icon.png`).
- **Feature Graphic (1024x500)**: 16:9 promotional banner (`store_assets/feature_graphic_1024x500.jpg` & `fastlane/metadata/android/en-US/images/featureGraphic.jpg`).
- **Companion UI Screenshot Blueprints**:
  1. *Screenshot 1: Ingestion Dashboard & Mirror Network*: Home view with top app bar, lifetime Pro status, Redlib decentralized mirror status, and direct thread ingestion input card.
  2. *Screenshot 2: Floating Audio Overlay (HUD) in Action*: Floating frosted-glass HUD active over a thread/browser displaying speaker badge, live spoken subtitle, audio controls, and acoustic waveform.
  3. *Screenshot 3: Supertonic On-Device Neural Voice Engine*: Voice engine settings highlighting 44.1 kHz Hi-Fi synthesis, quality step toggles, and 100% on-device privacy guarantee.
  4. *Screenshot 4 (Optional): Comment Hierarchy & Transcript*: Comment tree view showing parsed comments and jump navigation.


