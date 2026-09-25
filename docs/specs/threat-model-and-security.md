# Threat Model & Security Sheet: Redder

## 1. Threat Modeling Overview (STRIDE)
Redder processes untrusted external input (URLs and user-generated Reddit comments) and passes that data into an on-device neural synthesis engine. This document outlines the attack surface, identified threat vectors, and active hardening measures.

---

## 2. Ingestion Security: SSRF & Malicious Redirect Prevention

### A. Attack Vector: Arbitrary URL & SSRF Injection
* **Threat**: An attacker tricks Redder into sending HTTP requests to internal intranet endpoints (`http://192.168.1.1/admin`, `http://169.254.169.254/metadata`) or tracking endpoints via crafted share intents.
* **Threat**: A malicious shortlink issues an infinite redirect loop or redirects to a high-bandwidth binary payload ($>1\text{ GB}$) to cause mobile data overage or memory exhaustion.

### B. Defensive Countermeasures
1. **Strict Hostname Whitelisting**:
   Redder validates that the destination URL matches verified Reddit domains before opening network sockets:
   ```kotlin
   object UrlSecurityValidator {
       private val ALLOWED_HOSTS = setOf(
           "reddit.com",
           "www.reddit.com",
           "old.reddit.com",
           "redd.it",
           "v.redd.it"
       )

       fun isPermittedUrl(urlStr: String): Boolean {
           val uri = Uri.parse(urlStr)
           val host = uri.host?.lowercase() ?: return false
           val scheme = uri.scheme?.lowercase() ?: return false
           
           // Enforce HTTPS
           if (scheme != "https") return false
           
           // Check domain or valid subdomain
           return ALLOWED_HOSTS.contains(host) || host.endsWith(".reddit.com")
       }
   }
   ```
2. **Redirect Limits & Host Verification**:
   The `OkHttpClient` enforces a maximum of 5 redirects and validates that each redirect target remains within the permitted Reddit domain whitelist.
3. **HTTP Response Size Ceiling**:
   Reddit JSON responses are clamped to a hard ceiling of $5\text{ MB}$. Any payload exceeding $5\text{ MB}$ terminates the network stream immediately to prevent Out-Of-Memory (OOM) attacks from artificially bloated JSON listings.

---

## 3. Neural Engine & G2P Sanitization (Denial of Service Prevention)

### A. Attack Vector: Pathological Text & Tensor Memory Exhaustion
* **Threat**: An adversary posts a comment consisting of $50,000$ consecutive unspaced characters (e.g. `aaaaaaaa...`), nested recursive unicode diacritics (Zalgo text), or repeated mathematical symbols designed to expand into quadratic phoneme sequences.
* **Impact**:
  - G2P phonemizer memory runaway ($>500\text{ MB}$ allocation).
  - ONNX runtime tensor allocation failure (`OrtException: Failed to allocate memory for tensor`).
  - Total process crash or frozen UI thread.

### B. Defensive Countermeasures
1. **Character Length Clamping**:
   - Every comment body is truncated to a maximum of **1,200 characters** before processing.
   - Any single comment exceeding 1,200 characters is appended with a verbal truncation marker (`"... comment truncated for brevity."`).
2. **Zalgo & Non-Printable Character Stripping**:
   - Remove zero-width joiners, non-printable control characters, and excessive combining diacritical marks using unicode regex filtering:
   ```kotlin
   fun sanitizeUntrustedText(rawText: String): String {
       return rawText
           .take(1200)
           // Strip non-printable / control chars except standard whitespace
           .replace("[\\p{Cntrl}&&[^\r\n\t]]".toRegex(), "")
           // Strip combining diacritical marks (Zalgo text defense)
           .replace("\\p{M}{3,}".toRegex(), "")
           // Replace excessive repetitive characters ("aaaaaaa..." -> "aaa")
           .replace("([a-zA-Z0-9])\\1{5,}".toRegex(), "$1$1$1")
           .trim()
   }
   ```
3. **ONNX Tensor Size Clamping**:
   The input token buffer passed to the ONNX session is strictly bounded:
   - Maximum phoneme sequence length: $512$ tokens.
   - If an utterance yields $>512$ phonemes, it is split into two smaller sequential chunks, preventing native tensor buffer overflow.

---

## 4. Android IPC & Overlay Security

### A. Overlay Hijacking (`SYSTEM_ALERT_WINDOW`)
* **Threat**: Tapjacking or overlay click interception where a rogue app attempts to obscure the HUD or abuse the overlay to capture touches intended for another application.
* **Defense**:
  - Redder sets `WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL` and bounds touch handling strictly to the visual boundaries of its floating HUD widgets.
  - Transparent regions do not intercept touches, allowing underlying apps to receive gestures unimpeded.

### B. Component Export Hardening
* All internal services (`RedderAudioService`, `FloatingOverlayService`) are declared with `android:exported="false"`.
* Only `ShareReceiverActivity` is exported (`android:exported="true"`) to allow third-party apps to target it with `ACTION_SEND`.
* Inputs received from the share intent are treated as untrusted strings and subjected to the full security validation pipeline.
