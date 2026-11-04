# 📘 FinCore Phase 3 MFA (OTP) — Complete Lifecycle & Postman Testing Guide

This guide explains the exact behavior of the OTP system from the moment a user enters valid credentials to the end of the OTP session, followed by a complete, step-by-step Postman test plan with exact request payloads and expected JSON responses.

---

## Part 1: Understanding the 4 Timers & Redis Keys

When a user enters a valid User ID and Password on `/api/auth/login` (and `IS_OTP_ENABLED = 'Y'`), the backend generates a unique `transactionId` (`txId`) and starts **4 independent clocks** in Redis.

Understanding how these 4 keys work together makes the entire system behavior crystal clear:

| Redis Key | Initial Value | TTL (Timer) | What Happens When It Expires / Deletes? |
| :--- | :--- | :--- | :--- |
| **`OTP_SESSION:{txId}`** | `"2488766"` (`userId`) | **15 Minutes** ($900\text{s}$) | **The Master Session Clock (Section 2.2.9).** As long as this key exists, the user is allowed to stay on the OTP screen and click "Resend". If the user is idle for 15 minutes, this key vanishes and the user is kicked back to the Login screen (`"Your session has timed out. Please log in again."`). |
| **`OTP_TX:{txId}`** | `{"userId":"2488766", "otpHash":"$2a$10$..."}` | **5 Minutes** ($300\text{s}$) | **The Active 6-Digit Code Clock (Section 2.2.7).** Holds the BCrypt hash of the current 6-digit code. It is destroyed if: **(a)** 5 minutes pass, **(b)** the user enters a wrong code 5 times, or **(c)** the user verifies successfully. When destroyed due to (a) or (b), `OTP_SESSION:{txId}` is **still alive**, so the user cannot log in with the old code, but *can* click "Resend". |
| **`OTP_FAILURES:{txId}`** | `"0"` | **5 Minutes** ($300\text{s}$) | **The Wrong Attempt Counter (Section 2.2.6).** Increments on every wrong OTP (`1` $\rightarrow$ `2` $\rightarrow$ `3` $\rightarrow$ `4` $\rightarrow$ `5`). On the 5th wrong attempt, the backend immediately deletes `OTP_TX:{txId}` so the code is permanently dead. When the user clicks "Resend", this counter resets to `"0"` for the new code. |
| **`OTP_COOLDOWN:{txId}`** | `"locked"` | **60 Seconds** ($60\text{s}$) | **The Resend Button Lock (Section 2.2.5).** Created the moment an OTP is sent (initial or resend). While this key exists in Redis, calling `/api/auth/resend-otp` is blocked and returns the exact remaining seconds (`TTL`) of this key: `"Please wait {N} seconds before requesting another code."` |
| **`OTP_RESENDS:{txId}`** | `"0"` (up to `"3"`) | **10 Minutes** ($600\text{s}$) | **The Resend Window Counter (Section 2.2.5).** Tracks how many times the user has clicked "Resend" within a 10-minute window. Maximum allowed is `3`. On the 4th attempt within 10 minutes, it blocks the request with `"Maximum resend limit reached. Kindly Wait."` |

---

## Part 2: Chronological Walkthrough (Start to End of Session)

Here is how the system behaves minute-by-minute during a user's journey:

1. **$T = 00\text{m }00\text{s}$ (Login Initiated):**
   * User submits valid `userId` and `password` to `POST /api/auth/login`.
   * Backend checks DB for mobile/email. Suppose both exist (`9876541234` and `amit.rao@sbi.co.in`).
   * Backend generates `Code #1` (e.g., `482910`), hashes it with BCrypt, and creates all 5 Redis keys above.
   * Backend returns `HTTP 202 Accepted` with `"Code sent to ***1234 and a***o@sbi.co.in"`.
2. **$T = 00\text{m }15\text{s}$ (Early Resend Attempt - Blocked):**
   * User clicks "Resend" after 15 seconds.
   * Backend checks `OTP_COOLDOWN:{txId}`. It still has $45\text{ seconds}$ left.
   * Backend rejects with `HTTP 429 Too Many Requests`: `"Please wait 45 seconds before requesting another code."`
3. **$T = 00\text{m }30\text{s}$ to $00\text{m }50\text{s}$ (5 Wrong Attempts - Code Burned):**
   * Attempt 1 (Wrong): `OTP_FAILURES` becomes `1`. Returns `400 Bad Request`: `"Invalid code. 4 attempts remaining."`
   * Attempt 2 (Wrong): `OTP_FAILURES` becomes `2`. Returns `400 Bad Request`: `"Invalid code. 3 attempts remaining."`
   * Attempt 3 (Wrong): `OTP_FAILURES` becomes `3`. Returns `400 Bad Request`: `"Invalid code. 2 attempts remaining."`
   * Attempt 4 (Wrong): `OTP_FAILURES` becomes `4`. Returns `400 Bad Request`: `"Invalid code. 1 attempts remaining."`
   * Attempt 5 (Wrong): `OTP_FAILURES` hits `5`. Backend **deletes `OTP_TX:{txId}`** and returns `400 Bad Request`: `"Too many incorrect attempts. The code has been expired. Please request a new code."`
   * Attempt 6 (Even if user enters the *correct* `Code #1` now!): Because `OTP_TX:{txId}` was deleted on Attempt 5, the code is dead. Backend returns `400 Bad Request`: `"The code is invalid or has expired. Please request a new code."`
4. **$T = 01\text{m }05\text{s}$ (Resend #1 - Fresh Code & Reset Counter):**
   * 60 seconds have passed, so `OTP_COOLDOWN:{txId}` has expired.
   * User clicks "Resend" (`POST /api/auth/resend-otp`).
   * `OTP_RESENDS:{txId}` increments to `1` ($3 - 1 = 2$ resends remaining).
   * Backend generates `Code #2`, overwrites `OTP_TX:{txId}` (new 5-min TTL), **resets `OTP_FAILURES:{txId}` to `0`** (5 fresh attempts!), sets a new 60s `OTP_COOLDOWN:{txId}`, and refreshes `OTP_SESSION:{txId}` to 15 mins.
   * Backend returns `HTTP 202 Accepted`: `"New Code sent to ***1234 and a***o@sbi.co.in"`.
5. **$T = 06\text{m }10\text{s}$ (Natural 5-Minute Code Expiry):**
   * Suppose the user did nothing for 5 minutes after Resend #1.
   * `OTP_TX:{txId}` naturally expires from Redis at $06\text{m }05\text{s}$, while `OTP_SESSION:{txId}` is still alive.
   * User tries to submit `Code #2` at $06\text{m }10\text{s}$.
   * Backend sees `OTP_SESSION` exists, but `OTP_TX` is gone. Returns `HTTP 400 Bad Request`: `"The code is invalid or has expired. Please request a new code."`
6. **$T = 21\text{m }10\text{s}$ (15-Minute Idle Screen Timeout):**
   * If the user leaves the OTP screen idle for 15 minutes, `OTP_SESSION:{txId}` expires from Redis.
   * If the user now clicks "Verify" or "Resend", the backend sees `OTP_SESSION:{txId}` is missing and immediately returns `HTTP 401 Unauthorized`: `"Your session has timed out. Please log in again."`
7. **Alternative Happy Path (Successful Verification):**
   * Whenever the user submits the valid 6-digit code while `OTP_TX:{txId}` is active, the backend immediately deletes **all** OTP Redis keys (`OTP_SESSION`, `OTP_TX`, `OTP_FAILURES`, `OTP_RESENDS`, `OTP_COOLDOWN`) so the code can never be replayed, generates the JWE Access Token + Refresh Cookie + `X-Payload-Key` header, and returns `HTTP 200 OK`.

---

## Part 3: Complete Postman Testing Playbook (All Scenarios)

**Base URL:** `http://localhost:8085` (or your local Login Service port)

> 💡 **Pro-Tip for Fast Testing in Local Dev:**
> Since OTPs are BCrypt-hashed in Redis and AES-encrypted in Kafka, you can either check your **Communication Service logs** for the plain-text OTP, or temporarily log `plainOtp` in `LoginServiceImpl` during local Postman testing.
> Also, instead of waiting a real 5 or 15 minutes on your stopwatch, I have included `redis-cli` commands below to fast-forward Redis TTLs in 1 second!

---

### 🧪 Scenario 1: Login When User Has NO Mobile and NO Email (Section 2.2.1)
**Pre-requisite:** In Oracle DB, set `PHONE_NUMBER = NULL` and `EMAIL = NULL` for test user `2488766`, and ensure `LOGIN_PARAM.IS_OTP_ENABLED = 'Y'`.

* **Method & URL:** `POST http://localhost:8085/api/auth/login`
* **Request Headers:** `Content-Type: application/json`
* **Request Payload:**
```json
{
  "userId": "2488766",
  "password": "<RSA_ENCRYPTED_PASSWORD>"
}
```
* **Expected Status:** `400 Bad Request`
* **Exact Expected Response Body:**
```json
{
  "success": false,
  "message": "Code cannot be sent: No valid mobile number or email address is mapped with User ID.",
  "errorCode": "BAD_REQUEST",
  "timestamp": "2026-11-04T15:38:00.123456"
}
```

---

### 🧪 Scenario 2: Login Success -> OTP Sent (Both Mobile & Email Present) (Section 2.2.4)
**Pre-requisite:** User `2488766` has Mobile `9876541234` and Email `amitrao@sbi.co.in`.
*(Note: If only mobile is present, `message` will be `"Code sent to ***1234"`. If only email is present, `message` will be `"Code sent to a***o@sbi.co.in"`).*

* **Method & URL:** `POST http://localhost:8085/api/auth/login`
* **Request Payload:**
```json
{
  "userId": "2488766",
  "password": "<RSA_ENCRYPTED_PASSWORD>"
}
```
* **Expected Status:** `202 Accepted`
* **Exact Expected Response Body:**
```json
{
  "success": true,
  "message": "Code sent to ***1234 and a***o@sbi.co.in",
  "data": {
    "transactionId": "c9a646d3-9c61-4d8e-8a2b-123456789abc",
    "maskedMobile": "***1234",
    "maskedEmail": "a***o@sbi.co.in",
    "expiresInSeconds": 300,
    "resendCooldownSeconds": 60,
    "resendsRemaining": 3,
    "message": "Code sent to ***1234 and a***o@sbi.co.in"
  },
  "timestamp": "2026-11-04T15:40:00.123456"
}
```
📌 **Action:** Copy the `transactionId` from this response for the next tests!

---

### 🧪 Scenario 3: Invalid OTP Format (Non-Numeric or Not 6 Digits) (Section 2.2.3)
Test with letters (`"12ab56"`), fewer than 6 digits (`"123"`), or empty (`""`).

* **Method & URL:** `POST http://localhost:8085/api/auth/verify-otp`
* **Request Payload:**
```json
{
  "transactionId": "c9a646d3-9c61-4d8e-8a2b-123456789abc",
  "otp": "12ab5"
}
```
* **Expected Status:** `400 Bad Request`
* **Exact Expected Response Body:**
```json
{
  "success": false,
  "message": "Please enter a valid 6-digit numeric code.",
  "errorCode": "VALIDATION_ERROR",
  "timestamp": "2026-11-04T15:40:10.123456"
}
```

---

### 🧪 Scenario 4: Resend Clicked Before 60 Seconds Cooldown (Section 2.2.5)
Immediately after Scenario 2 (within 60 seconds), call the Resend API.

* **Method & URL:** `POST http://localhost:8085/api/auth/resend-otp`
* **Request Payload:**
```json
{
  "transactionId": "c9a646d3-9c61-4d8e-8a2b-123456789abc"
}
```
* **Expected Status:** `429 Too Many Requests`
* **Exact Expected Response Body:**
```json
{
  "success": false,
  "message": "Please wait 48 seconds before requesting another code.",
  "errorCode": "TOO_MANY_REQUESTS",
  "timestamp": "2026-11-04T15:40:12.123456"
}
```
*(Note: `48` will dynamically match whatever seconds are left on `OTP_COOLDOWN:{txId}` in Redis).*

---

### 🧪 Scenario 5: Wrong OTP Attempts 1 to 4 (Section 2.2.6)
Send a wrong 6-digit numeric OTP (e.g., `"000000"`) 4 times in a row.

* **Method & URL:** `POST http://localhost:8085/api/auth/verify-otp`
* **Request Payload:**
```json
{
  "transactionId": "c9a646d3-9c61-4d8e-8a2b-123456789abc",
  "otp": "000000"
}
```
* **Expected Status:** `400 Bad Request`
* **Exact Expected Response Body (1st Wrong Attempt):**
```json
{
  "success": false,
  "message": "Invalid code. 4 attempts remaining.",
  "errorCode": "BAD_REQUEST",
  "timestamp": "2026-11-04T15:40:20.123456"
}
```
* **2nd Wrong Attempt `message`:** `"Invalid code. 3 attempts remaining."`
* **3rd Wrong Attempt `message`:** `"Invalid code. 2 attempts remaining."`
* **4th Wrong Attempt `message`:** `"Invalid code. 1 attempts remaining."`

---

### 🧪 Scenario 6: 5th Wrong OTP Attempt -> Code Marked Expired (Section 2.2.6 & 2.2.7)
Send the wrong OTP a **5th time**.

* **Method & URL:** `POST http://localhost:8085/api/auth/verify-otp`
* **Request Payload:**
```json
{
  "transactionId": "c9a646d3-9c61-4d8e-8a2b-123456789abc",
  "otp": "000000"
}
```
* **Expected Status:** `400 Bad Request`
* **Exact Expected Response Body:**
```json
{
  "success": false,
  "message": "Too many incorrect attempts. The code has been expired. Please request a new code.",
  "errorCode": "BAD_REQUEST",
  "timestamp": "2026-11-04T15:40:35.123456"
}
```

---

### 🧪 Scenario 7: Trying to Verify After 5th Failed Attempt OR After 5-Minute Expiry (Section 2.2.7)
Now that the code was expired by the 5th wrong attempt (or if you waited 5 minutes / ran `DEL OTP_TX:<txId>` in `redis-cli`), try sending any 6-digit code (even the original correct code!).

* **Method & URL:** `POST http://localhost:8085/api/auth/verify-otp`
* **Request Payload:**
```json
{
  "transactionId": "c9a646d3-9c61-4d8e-8a2b-123456789abc",
  "otp": "482910"
}
```
* **Expected Status:** `400 Bad Request`
* **Exact Expected Response Body:**
```json
{
  "success": false,
  "message": "The code is invalid or has expired. Please request a new code.",
  "errorCode": "BAD_REQUEST",
  "timestamp": "2026-11-04T15:40:45.123456"
}
```

---

### 🧪 Scenario 8: Valid Resend After 60 Seconds (Resets Attempt Counter!) (Section 2.2.5)
Wait until 60 seconds have passed since Login (or delete the cooldown key in `redis-cli`: `DEL OTP_COOLDOWN:c9a646d3-9c61-4d8e-8a2b-123456789abc`), then call Resend.

* **Method & URL:** `POST http://localhost:8085/api/auth/resend-otp`
* **Request Payload:**
```json
{
  "transactionId": "c9a646d3-9c61-4d8e-8a2b-123456789abc"
}
```
* **Expected Status:** `202 Accepted`
* **Exact Expected Response Body (Resend #1):**
```json
{
  "success": true,
  "message": "New Code sent to ***1234 and a***o@sbi.co.in",
  "data": {
    "resendCooldownSeconds": 60,
    "resendsRemaining": 2,
    "message": "New Code sent to ***1234 and a***o@sbi.co.in"
  },
  "timestamp": "2026-11-04T15:41:15.123456"
}
```
*(Verify Attempt Reset: If you enter a wrong code now, it will say `"Invalid code. 4 attempts remaining."`—proving the failure counter was reset to 0!).*

---

### 🧪 Scenario 9: Exceeding Maximum 3 Resends in 10 Minutes (Section 2.2.5)
Perform Resend #2 (`resendsRemaining: 1`) and Resend #3 (`resendsRemaining: 0`) after each 60-second cooldown.
Then, after another 60 seconds, click Resend a **4th time**.

* **Method & URL:** `POST http://localhost:8085/api/auth/resend-otp`
* **Request Payload:**
```json
{
  "transactionId": "c9a646d3-9c61-4d8e-8a2b-123456789abc"
}
```
* **Expected Status:** `429 Too Many Requests`
* **Exact Expected Response Body:**
```json
{
  "success": false,
  "message": "Maximum resend limit reached. Kindly Wait.",
  "errorCode": "TOO_MANY_REQUESTS",
  "timestamp": "2026-11-04T15:44:30.123456"
}
```

---

### 🧪 Scenario 10: 15-Minute OTP Entry Screen Idle Timeout (Section 2.2.9)
To test this without waiting 15 minutes on your clock, run this in your terminal:
`redis-cli DEL OTP_SESSION:c9a646d3-9c61-4d8e-8a2b-123456789abc`
Now call either `/api/auth/verify-otp` OR `/api/auth/resend-otp`.

* **Method & URL:** `POST http://localhost:8085/api/auth/verify-otp` (or `/api/auth/resend-otp`)
* **Request Payload:**
```json
{
  "transactionId": "c9a646d3-9c61-4d8e-8a2b-123456789abc",
  "otp": "123456"
}
```
* **Expected Status:** `401 Unauthorized`
* **Exact Expected Response Body:**
```json
{
  "success": false,
  "message": "Your session has timed out. Please log in again.",
  "errorCode": "UNAUTHORIZED",
  "timestamp": "2026-11-04T15:55:00.123456"
}
```

---

### 🧪 Scenario 11: Successful OTP Verification & One-Time Use Check (Section 2.2.8)
1. Initiate a fresh login (`POST /api/auth/login`) to get a new `transactionId` and 6-digit OTP (e.g., `"839201"`).
2. Call `POST /api/auth/verify-otp` with the correct OTP.

* **Method & URL:** `POST http://localhost:8085/api/auth/verify-otp`
* **Request Payload:**
```json
{
  "transactionId": "new-tx-uuid-here",
  "otp": "839201"
}
```
* **Expected Status:** `200 OK`
* **Expected Response Headers:**
  * `X-Payload-Key: <base64-aes-key>`
  * `Set-Cookie: refresh_token=<5-part-jwe>; Path=/; Max-Age=43200; HttpOnly; Secure; SameSite=Strict`
* **Exact Expected Response Body:**
```json
{
  "success": true,
  "message": "Login successful",
  "data": {
    "userId": "2488766",
    "accessToken": "eyJraWQiOiIyNDg4NzY2IiwiY3R5IjoiSldUIiwiZW5jIjoiQTI1NkdDTSIsImFsZyI6ImRpciJ9...",
    "tokenType": "Bearer",
    "expiresIn": 900,
    "sub": "2488766",
    "jti": "ade5c06b-7bcc-4cd1-844a-16c7873dc11a",
    "roleData": {
      "menus": [...]
    },
    "validCredentials": true,
    "userStatus": "ACTIVE",
    "passwordLoginStatus": "ACTIVE"
  },
  "timestamp": "2026-11-04T15:56:00.123456"
}
```
3. **One-Time Use Replay Test:** Immediately click **Send** again on the exact same `/api/auth/verify-otp` request!
   * Because `cleanupAllOtpKeys(txId)` deleted `OTP_SESSION` and `OTP_TX` upon success, the second request is immediately rejected with `401 Unauthorized` (`"Your session has timed out. Please log in again."`).

---

### 🧪 Scenario 12: Kafka / SMS-Email Provider Down (Section 2.4 & 6)
Stop your local Kafka container/service and call `POST /api/auth/login` (or `POST /api/auth/resend-otp`).

* **Expected Status:** `503 Service Unavailable`
* **Exact Expected Response Body:**
```json
{
  "success": false,
  "message": "Unable to reach the SMS/email provider. Please try again after some time.",
  "errorCode": "SERVICE_UNAVAILABLE",
  "timestamp": "2026-11-04T15:58:00.123456"
}
```
*(Behind the scenes, `rollbackOtpState(txId)` automatically deletes all Redis keys created for that `txId` so no orphaned keys remain).*