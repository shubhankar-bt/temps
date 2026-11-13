# Proposed Updates to the FinCore Phase 3 MFA Solution Document

Hi [Manager's Name],

While implementing and testing the Multi-Factor Authentication (MFA) flow in the Login Service, we identified a few practical scenarios and security refinements that were not explicitly detailed in the initial version of the Solution Document. To ensure our documentation matches the actual system behavior before we hand the build over to the SIT, UAT, and ISD teams, we should include the following updates in the Solution Document.

---

## 1. Updates to Existing Sections (Section 2.2)

### Section 2.2.2 - Code Generation and Storage Security
* **What we should add:** In addition to hashing the 6-digit code with BCrypt before storing it in Redis, when the Login Service sends the code to the notification service over Kafka, the internal payload is encrypted using AES-256-GCM. This ensures the plain-text code is never exposed in memory or on the message broker.

### Section 2.2.3 - Code Input and Payload Encryption
* **What we should add:** Along with validating that the input is strictly a 6-digit numeric value, the verification endpoint now supports RSA-OAEP encryption from the frontend (using the same RSA public key mechanism used for password encryption). It also retains backward compatibility for standard 6-digit inputs during testing.

### Section 2.2.4 - Code Sending and User Context on OTP Screen
* **What we should add:** Along with returning the masked mobile number and/or masked email address, the initial login response now returns the authenticated user's name. This allows the frontend to display the user's name on the Two-Factor Verification screen so the user can confirm their identity before entering the code.

### Section 2.2.5 - Resending and "Back to Sign In" Navigation Handling
* **Why this is needed:** On the Two-Factor Verification screen, users have both a "Resend OTP" button and a "Back to Sign In" button. If the 60-second cooldown and the 3-resend limit were only tracked per screen transaction, a user could click "Back to Sign In" and log in again repeatedly to generate unlimited codes. Conversely, if a user accidentally clicked "Back to Sign In" within 60 seconds while holding a valid code, blocking them on the login page would prevent them from entering the code they already received.
* **What we should add to the document:**
  1. The 60-second cooldown and the maximum limit of 3 resends within a 10-minute window are enforced at the **User ID level** rather than just the screen session level.
  2. **Smart Session Resumption:** If a user clicks "Back to Sign In" and re-authenticates while their previously generated code is still valid (less than 5 minutes old and fewer than 5 failed attempts), and they are still within the 60-second cooldown (or have used all 3 resends), the application will take them to the OTP screen and resume their existing valid code and remaining countdown timers without dispatching a duplicate SMS or email.
  3. **Quota Enforcement on Re-Login:** If a user goes back to the Sign-In page and logs in again after the 60-second cooldown within the 10-minute window, the system generates a fresh code and counts it as one of their 3 permitted resends within that 10-minute window.
  4. **Bypass Prevention:** If a user exhausts all 5 attempts on a code (marking it expired) and immediately goes back to the Sign-In screen to try to bypass the 60-second timer or the 3-resend limit, the system will block the request and display the standard wait or limit message.

### Section 2.2.8 - Verification and Double-Click Protection
* **What we should add:** The verification step includes an atomic request lock so that if a user accidentally double-clicks the Verify button, the second simultaneous request is safely ignored and does not cause a false session error or token mismatch. All OTP-related counters and locks for that user are immediately cleared from Redis upon a successful login.

### Section 2.2.9 - OTP Entry Screen Idle Time-Out
* **What we should add:** The 15-minute screen time-out operates as a sliding inactivity window. Every time the user actively interacts with the verification or resend APIs, the 15-minute session timer is refreshed in Redis. The backend also passes the configured timeout duration (`OTP_SCREEN_TIMEOUT_MIN`) in the login and resend responses so the frontend can automatically redirect the user back to the Sign-In screen if the screen remains untouched for 15 minutes.

---

## 2. New Sub-Sections to Add Under Section 2.2

### Section 2.2.10 - Pre-Login Concurrent Session Control
* **Description:** To prevent the same User ID from initiating parallel OTP sessions across multiple devices or browsers at the same time:
  1. If a second device attempts to log in with the same User ID while the 60-second dispatch cooldown is active, the simultaneous request is prevented from generating a second code.
  2. Only one active OTP transaction is allowed per User ID at any given time. Whenever a new OTP session is legitimately initiated or resumed on one device, any previously open OTP transaction for that User ID on another device or browser tab is automatically invalidated.

### Section 2.2.11 - Database Parameterization and Caching
* **Description:** All MFA time intervals and threshold limits are parameterized in the `LOGIN_PARAM` database table rather than hardcoded in the application:
  * Code Validity Period (Default: 5 minutes)
  * Maximum Wrong Attempts per Code (Default: 5 attempts)
  * Resend Cooldown Period (Default: 60 seconds)
  * Maximum Resends Allowed (Default: 3 resends)
  * Resend Tracking Window (Default: 10 minutes)
  * OTP Entry Screen Idle Timeout (Default: 15 minutes)
* These parameters are cached in memory using Caffeine with randomized expiration intervals (jitter) so the database is not queried on every login request, and any changes made in the database are automatically refreshed or can be applied immediately by restarting the service.

---

## 3. Wording Alignment Check for Section 2.4 (User Messages)

In Section 2.4 of the current Solution Document, a few messages use the word **"Code"**, while our current application responses use the word **"OTP"** in those specific places. Please confirm with the BA/SIT team whether we should update the Solution Document table to say **"OTP"** for the following rows, or if they prefer us to keep the word **"Code"** in the backend:

| Scenario | Current Solution Document Text | Current Application Text |
| :--- | :--- | :--- |
| Neither mobile nor email present | `Code cannot be sent: No valid mobile number or email address is mapped with User ID.` | `OTP cannot be sent: No valid mobile number or email address is mapped with User ID.` |
| Initial dispatch (Both / Mobile / Email) | `Code sent to ***1234 and a***o@sbi.co.in` | `OTP sent to ***1234 and a***o@sbi.co.in` |
| Resend dispatch (Both / Mobile / Email) | `New Code sent to ***1234 and a***o@sbi.co.in` | `New OTP sent to ***1234 and a***o@sbi.co.in` |

*(Note: All other error messages for invalid input, remaining attempts, expired code, cooldown wait time, max resend limit, provider error, and 15-minute session timeout match the Solution Document word-for-word).*

---

## 4. Pre-SIT Guidance and Checklist (Engineering Recommendations)

Before you promote this build to the SIT environment, here are the most important practical checks to complete so your testing cycle goes smoothly:

1. **Confirm the "Code" vs. "OTP" Wording Early:**
   SIT testing teams in banking environments often log defects if a message says `"OTP sent to..."` when the specification document says `"Code sent to..."`. Getting your manager to either update those rows in Section 2.4 of the document or confirming which word they want before SIT starts will save you unnecessary defect tickets.

2. **Execute the `LOGIN_PARAM` Database Script in SIT First:**
   Make sure the `ALTER TABLE LOGIN_PARAM` script adding the five new MFA columns (`OTP_MAX_ATTEMPTS`, `OTP_RESEND_COOLDOWN_SEC`, `OTP_MAX_RESENDS`, `OTP_RESEND_WINDOW_MIN`, `OTP_SCREEN_TIMEOUT_MIN`) is executed in the SIT database before deploying the new Login Service jar. Even though we added safe fallback defaults in Java, having the columns missing in the SIT table will cause Hibernate/JPA schema or query errors when `LoginParamRepository.getLoginParam()` runs.

3. **Coordinate with the Frontend Team on Smart Session Resumption:**
   Let the frontend developer know that when `/api/auth/login` returns `202 Accepted`, the UI should always initialize its countdown timer using the `resendCooldownSeconds` and `resendsRemaining` values returned in the JSON payload, rather than assuming the cooldown is always 60 seconds and resends remaining is always 3. That way, when a user clicks "Back to Sign In" and comes back 20 seconds later, the UI will accurately show `40 seconds` left on the Resend button and `2 resends` remaining.

4. **Check Clock Synchronization and Redis Database Index in SIT:**
   Verify that the Login Service and Common Utilities in the SIT environment point to the exact same Redis database index (`spring.data.redis.database`), and ensure the SIT servers are synced via NTP so JWT/JWE expiration timestamps (`exp`) do not drift between pods.

5. **Temporarily Enable Plain-Text OTP Logging Only in SIT (Or Use a Fixed SIT Test Mode):**
   If the bank's SMS/Email gateway in SIT has delays or is not connected to real tester phone numbers, SIT testers will struggle to test the happy path because the OTP is BCrypt-hashed in Redis and AES-encrypted in Kafka. Coordinate with the Notification Service team so SIT testers have a reliable way to view dispatched OTPs during testing, and ensure no plain-text OTPs are ever logged in the Login Service in UAT or Production.