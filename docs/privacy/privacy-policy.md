# Privacy Policy for ThreadReader

**Effective Date:** September 25, 2026  
**Last Updated:** September 25, 2026  
**Application Identifier:** `com.redder`  
**Publisher:** Redder Developer / ThreadReader Team  
**Contact Email:** `threader.support@gmail.com`  

---

## 1. Introduction

ThreadReader ("we", "our", or "the application") is committed to protecting your privacy. This Privacy Policy describes how ThreadReader handles information when you use our Android application. 

ThreadReader is built on a **local-first, radical privacy minimization architecture**. We do not operate remote user databases, do not require user accounts or logins, and execute all voice and speech synthesis entirely on your device.

---

## 2. Core Architectural Privacy Guarantees

### 2.1 100% On-Device Neural Speech Synthesis (Zero Cloud AI)
- **Local Neural Execution:** All text-to-speech synthesis (utilizing Piper ONNX and local neural acoustic models) is executed purely on your Android device using local CPU/GPU/NPU compute hardware.
- **No Remote AI Transmission:** Thread content, discussions, voice characteristics, speech tokens, and generated audio streams are **never** transmitted to remote third-party AI or cloud speech providers (such as OpenAI, ElevenLabs, Google Cloud Text-to-Speech, or Microsoft Azure).
- **No Microphone Access:** The application does **not** declare or request the `RECORD_AUDIO` permission. ThreadReader is strictly an audio player and synthesizer; it does not capture or listen to microphone audio.

### 2.2 Public Forum Ingestion & Decentralized Mirrors
- **Public Forum Threads:** ThreadReader retrieves public discussion threads from Reddit or decentralized community mirrors (such as Redlib instances) based strictly on links you provide manually or share into the app via Android sharing intents.
- **Zero Account Harvesting:** ThreadReader does not require or offer user account registration. You do not log in with Reddit, provide credentials, or generate user accounts. We do not harvest, store, or transmit usernames, passwords, or personal profiles.
- **Network Boundaries & Security:** All network requests enforce strict TLS 1.3 / HTTPS encryption. Cleartext HTTP is prohibited. The app includes active Server-Side Request Forgery (SSRF) and intranet filters that automatically block access to private local network ranges (RFC 1918) and cloud metadata endpoints.

---

## 3. Data Collection, Monetization & Advertising

### 3.1 Free Tier (Ad-Supported)
- **Google AdMob:** The free version of ThreadReader displays banner and interstitial advertisements served by Google AdMob (Google LLC).
- **Identifiers Collected:** The Google Mobile Ads SDK may collect and process device identifiers, specifically the **Android Advertising ID (AAID)**, IP addresses, and diagnostic crash/performance metrics for ad delivery, personalization (where consented), frequency capping, and fraud prevention in accordance with [Google's Privacy Policy](https://policies.google.com/privacy).
- **Consent Management:** In regions requiring consent (such as the European Economic Area and the UK), ThreadReader integrates Google's User Messaging Platform (UMP) SDK to collect and respect user consent choices for personalized versus non-personalized ads before ad requests are initialized.

### 3.2 Pro Tier (Ad-Free Suppression Guarantee)
- **Complete Ad Component Deactivation:** If you purchase the optional lifetime "Remove Ads" / Pro in-app purchase (`threadreader_remove_ads`), all advertising components within the app are permanently disabled.
- **Zero Advertising Telemetry for Pro Users:** When Pro status is active, ad loading routines are halted, banner views are unmounted, and advertising identifiers are never requested, collected, or shared with Google AdMob.

### 3.3 Payment Processing
- **Google Play Billing:** All in-app purchases are handled entirely by Google Play Billing through secure operating system IPC (`BillingClient`).
- **No Financial Data Handled:** ThreadReader never collects, processes, or stores credit card numbers, bank account numbers, or billing addresses. Your payment information is governed by the [Google Play Terms of Service](https://play.google.com/intl/en_us/about/play-terms/).

---

## 4. Android Device Permissions & Usage

ThreadReader requests only the minimal permissions required for media playback functionality:

| Permission | Purpose | User Control |
| :--- | :--- | :--- |
| `INTERNET` | Retrieve public discussion content and serve ads (free tier only). | Granted automatically by Android upon installation. |
| `FOREGROUND_SERVICE` & `FOREGROUND_SERVICE_MEDIA_PLAYBACK` | Enable background audio playback and maintain MediaSession integration. | Active only while audio playback is in progress. |
| `POST_NOTIFICATIONS` | Display standard media playback controls (Play, Pause, Skip) in the notification shade. | Can be permitted or silenced at any time in system settings. |
| `SYSTEM_ALERT_WINDOW` *(Optional)* | Render the floating Head-Up Display (HUD) overlay over other apps. | **Optional:** Explicitly requested only if you toggle the floating HUD feature. Can be revoked at any time. |

---

## 5. Data Storage, Retention & Deletion

- **Local-Only Storage:** All application data—including local preferences, cached public discussion threads, and downloaded neural voice packages—resides exclusively on your device's internal storage (`context.filesDir` / `SharedPreferences`).
- **Zero Remote Storage:** We do not operate remote databases or cloud servers holding your personal data.
- **User Data Deletion:** Because all data is stored on-device, you maintain total control over your data:
  1. **Instant In-App / System Purge:** You can delete all cached data, voice models, and preferences instantly by navigating to your Android device's **Settings $\to$ Apps $\to$ ThreadReader $\to$ Storage $\to$ Clear Data**.
  2. **Uninstallation:** Completely uninstalling the app permanently purges all associated local files and settings.
  3. **Data Inquiries:** To request assistance or verify data handling, contact us at `threader.support@gmail.com`.

---

## 6. Regulatory Disclosures

### 6.1 European Users (GDPR)
Under the General Data Protection Regulation (GDPR), European Economic Area (EEA) and UK users have specific data protection rights:
- **Legal Basis for Processing:** Processing of network requests for thread ingestion is based on legitimate interest in delivering user-requested content. Advertising identifiers (free tier only) are processed based on your freely given consent collected via the Google UMP consent banner.
- **Your Rights:** You have the right to access, rectify, port, object to processing, and erase your data. Because all application data resides locally on your device, clearing application storage or uninstalling the app executes complete data erasure.

### 6.2 California Users (CCPA / CPRA)
Under the California Consumer Privacy Act as amended by the CPRA:
- **Categories of Data Collected:** Identifiers (Android Advertising ID in free tier), internet/network activity (ad telemetry in free tier).
- **No Sale of Personal Data:** ThreadReader does not sell personal information for monetary consideration.
- **Opt-Out Mechanism:** You can opt out of personalized advertising by resetting or deleting your Advertising ID in Android Settings (**Settings $\to$ Privacy / Google $\to$ Ads**) or by upgrading to the Pro (Ad-Free) tier.

---

## 7. Children's Privacy (COPPA Compliance)

ThreadReader is not directed toward children under the age of 13. The application does not knowingly collect, solicit, or maintain personal information from children under 13. Because the application ingests public internet discussion content, the application is rated for users aged 13 and older. If you believe a child under 13 has provided personal information to us, please contact us immediately at `threader.support@gmail.com`, and we will take immediate steps to address the matter.

---

## 8. Third-Party Links & Services

The application may ingest content containing links to third-party websites or services. We are not responsible for the privacy practices, content, or policies of any third-party websites or platforms. We encourage you to review the privacy policies of any third-party sites you visit.

---

## 9. Changes to This Privacy Policy

We may update this Privacy Policy from time to time. Any changes will be posted with an updated "Last Updated" date at the top of this document and updated on the Google Play Store listing. Continued use of ThreadReader after any revisions indicates acceptance of the updated policy.

---

## 10. Contact Us

If you have questions, feedback, or data privacy requests concerning this Privacy Policy, please contact:

- **Developer:** Redder Developer / ThreadReader Team  
- **Email:** `threader.support@gmail.com`  
- **Issue Tracker:** Via our public repository or the Google Play developer contact page.
