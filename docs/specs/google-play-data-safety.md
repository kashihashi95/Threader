# Google Play Store Data Safety & Privacy Disclosures Specification: ThreadReader

**Application Identifier:** `com.redder`  
**Application Title:** ThreadReader  
**Target Platform:** Android 14+ (API 34) & Android 15 (API 35)  
**Document Version:** 1.0.0  
**Effective Date:** September 2026  

---

## 1. Executive Privacy Architecture

ThreadReader is architected under the principle of **Zero Cloud Data Leakage** and **Radical Minimization**. The application transforms public discussion threads from Reddit and decentralized Redlib mirrors into conversational multi-speaker audio dialogue.

The architecture enforces three foundational privacy guarantees:
1. **100% On-Device Neural TTS:** Speech synthesis occurs purely on-device via local ONNX Runtime sessions (Piper ONNX / Supertonic). No user input, thread content, voice characteristics, or synthesized audio streams are ever transmitted to cloud AI servers.
2. **Decentralized Public Ingestion:** Thread ingestion queries public decentralized community mirrors (Redlib instances) and public unauthenticated REST endpoints. The app requires zero user login credentials, zero OAuth tokens, and harvests no account or personal data.
3. **Monetized Privacy via Pro Suppression:** AdMob advertising identifiers are strictly isolated and completely suppressed upon purchasing the lifetime "Remove Ads" / Pro entitlement.

---

## 2. Google Play Console Data Safety Questionnaire Mapping

The following table provides the exact declarations required in the Google Play Console **App Content $\to$ Data Safety** section.

### A. Data Collection & Sharing Overview

| Play Console Question | Response | Technical Architecture Justification |
| :--- | :--- | :--- |
| **Does your app collect or share any of the required user data types?** | **Yes** | Advertising ID is accessed solely by the Google Mobile Ads SDK in the free tier for ad serving. |
| **Is all of the user data collected by your app encrypted in transit?** | **Yes** | All network requests (ad requests and public mirror queries) strictly use TLS 1.3 / HTTPS. Cleartext HTTP is prohibited by Network Security Config. |
| **Do you provide a way for users to request that their data is deleted?** | **Yes** | The app stores zero user personal data on any remote server. Users can delete all local caches immediately via Android System Settings $\to$ Clear Data / Uninstall. |
| **Is your app subject to the EU General Data Protection Regulation (GDPR)?** | **Yes** | Full GDPR compliance: consent management via Google UMP SDK, zero backend tracking, local-first storage. |
| **Does your app collect data from children under 13?** | **No** | ThreadReader does not target children; age rating is 13+ (or 17+ depending on forum content settings). |

---

### B. Detailed Data Category Declarations

#### 1. Audio & Voice Data
* **Data Type:** Voice or sound recordings
* **Collected?**: **NO**
* **Shared?**: **NO**
* **Technical Justification:**
  - `PiperOnnxTtsEngine` synthesizes speech directly in native memory using local ONNX model weights (`model.onnx`) and acoustic configurations.
  - Zero audio buffers or speech tokens are uploaded to third-party AI APIs (such as ElevenLabs, OpenAI, Google Cloud Text-to-Speech, or Azure Speech).
  - The Android permission `android.permission.RECORD_AUDIO` is **NOT declared** in `AndroidManifest.xml` and is blocked by CI compliance tests.

#### 2. Personal Info & User Accounts
* **Data Type:** Name, Email address, User IDs, Address, Phone number, Financial info
* **Collected?**: **NO**
* **Shared?**: **NO**
* **Technical Justification:**
  - ThreadReader does not implement user registration, account sign-in, or social login.
  - In-app purchases for Pro are handled entirely by Google Play Billing via encrypted OS IPC (`BillingClient`). ThreadReader never inspects or touches credit card or billing details.

#### 3. Browsing History & Web Activity
* **Data Type:** Web browsing history, In-app search queries
* **Collected?**: **NO**
* **Shared?**: **NO**
* **Technical Justification:**
  - Forum thread URLs provided via Android `SEND` sharing intents or manual URL input are processed transiently in memory to fetch the public discussion payload.
  - No browsing logs, URL history, or reading telemetry are uploaded or synced to external servers.

#### 4. Device or Other Identifiers
* **Data Type:** Device or other IDs (specifically Android Advertising ID / AAID)
* **Collected?**: **YES (Conditional)**
* **Shared?**: **YES (Conditional - AdMob)**
* **Processed Ephemerally?**: **NO**
* **Is this data required or optional?**: **Required for Free Tier; Suppressed for Pro Tier**
* **Purposes:**
  1. Advertising or marketing (displaying banner and frequency-capped interstitial ads via Google AdMob).
  2. Analytics & fraud prevention (Google Mobile Ads SDK telemetry).
* **Pro Suppression Guarantee:**
  - When the user purchases the lifetime Pro / Ad-Free in-app purchase (`threadreader_remove_ads`), `BillingRepository.isAdFree` emits `true`.
  - In response, `GoogleAdManager` immediately cancels ad loading, zeroes out ad references, and drops all ad requests. Banner views are unmounted from the Compose UI hierarchy.
  - For Pro users, advertising identifiers are never requested or shared.

---

## 3. Detailed Component Privacy Audits

### 3.1 On-Device Neural TTS (Piper ONNX & Supertonic)
- **Inference Boundary:** Purely within the process memory space of `com.redder`.
- **Zero Third-Party AI Handshake:** The app does not initialize HTTP clients or WebSocket connections during phonemization, token mapping, or acoustic model inference.
- **Model Storage:** Model weights and CMUDict assets reside in local application storage (`context.filesDir` / `assets`).

### 3.2 Network Ingestion: Public Decentralized Community Mirrors (Redlib)
- **Public Mirror Ingestion:** Users can fetch discussion threads through public community mirrors (Redlib instances) without needing Reddit developer API keys or user account authorization.
- **Zero Credential Exposure:** Requests contain only standard read-only HTTP headers (`User-Agent: ThreadReader/1.0`). No cookies, session tokens, or personal identifiers are sent.
- **SSRF & Intranet Protection:** `UrlSecurityValidator` rejects all private RFC 1918 IPv4 ranges (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`), loopback (`127.0.0.1`), IPv6 local addresses (`[::1]`), and cloud metadata IP endpoints (`169.254.169.254`).
- **Memory Denial-of-Service Defense:** `RedditApiClient` enforces a strict 5MB payload ceiling (`MAX_PAYLOAD_BYTES`), terminating incoming streams with `PayloadTooLargeException` if an oversized payload is received.

### 3.3 Monetization, Google AdMob, and Pro Tier Suppression
- **Free Tier Operations:**
  - Google AdMob SDK initializes and requests standard banner (`BANNER_TEST_AD_UNIT_ID`) and interstitial (`INTERSTITIAL_TEST_AD_UNIT_ID`) units.
  - Interstitial presentations are frequency-capped with a mandatory 5-minute cooldown (`INTERSTITIAL_COOLDOWN_MS = 300_000L`).
  - Overlay Isolation: Ads are strictly confined to `MainActivity` and explicitly prohibited from `FloatingOverlayService`.
- **Pro Tier Operations:**
  - `BillingRepository` monitors Google Play Billing purchases via `StateFlow<Boolean> isAdFree`.
  - When `isAdFree == true`:
    1. `GoogleAdManager.initialize()` exits immediately without calling `MobileAds.initialize()`.
    2. Any existing `interstitialAd` reference is nullified.
    3. `canShowInterstitial()` returns `false`.
    4. `showInterstitialIfAllowed()` executes `onDismiss()` and returns `false`.
    5. `MainDashboardState.isBannerVisible` evaluates to `false`, removing the banner view from composition.
  - Zero advertising identifiers are transmitted for Pro users.

---

## 4. Google Play Policy Checklists

- [x] **Zero Accessibility Services Policy:** Confirmed. No `BIND_ACCESSIBILITY_SERVICE` declared or used.
- [x] **Foreground Service Type Declaration:** Confirmed. `FOREGROUND_SERVICE_MEDIA_PLAYBACK` declared and tied to active playback notification with `MediaSessionService`.
- [x] **System Alert Window Permission:** Explanatory consent card shown in `MainActivity`; graceful fallback to standard background audio notification when denied.
- [x] **Encrypted Data in Transit:** Confirmed. 100% TLS/HTTPS across all network communication.
- [x] **Data Safety Form Readiness:** Complete parameter mapping verified above.
