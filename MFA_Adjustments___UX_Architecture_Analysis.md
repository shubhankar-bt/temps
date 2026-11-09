# 🎯 FinCore MFA Phase 3 — Idle Timeout, User Details & "Back to Sign In" UX Architecture

This document addresses your three adjustment requirements in detail, aligning frontend behavior, backend state management, and the bank's Solution Document.

---

## 1. How the 15-Minute Idle Timeout (`OTP_SCREEN_TIMEOUT_MIN`) Works (Frontend + Backend)

### What the Solution Document Requires (Section 2.2.9 & Table 2.4)
* **Requirement (2.2.9):** *"In case the user remains idle on OTP entry screen for 15 minutes, the user will be prompted to enter user ID and password again."*
* **User Message (2.4):** `"Your session has timed out. Please log in again."`

### How Frontend and Backend Coordinate This
To deliver a true **idle timeout** (sliding window), both the frontend and backend work together using the `otpScreenTimeoutMin` (`OTP_SCREEN_TIMEOUT_MIN`) value sent in the `/api/auth/login` (`202 Accepted`) response:

1. **Frontend Behavior (Proactive Idle Timer on the OTP Screen):**
   * When the user lands on the Two-Factor Verification screen, the frontend reads `otpScreenTimeoutMin` (e.g., `15`) / `OTP_SCREEN_TIMEOUT_MIN` from the response.
   * The frontend starts a **15-minute inactivity timer** (`15 * 60 * 1000` ms).
   * **Resetting the Timer on Activity:** If the user interacts with the OTP screen (typing digits, clicking *Verify*, or clicking *Resend OTP*), the frontend resets its 15-minute idle timer back to `15:00`.
   * **When 15 Minutes of Inactivity Elapsed:** If the user does not touch the screen for 15 full minutes, the frontend timer hits `00:00`. The frontend **automatically clears the `transactionId` from memory, redirects the user back to the Sign-In screen**, and displays the toast/alert:
     `"Your session has timed out. Please log in again."`

2. **Backend Behavior (Sliding Window Security Enforcement in Redis):**
   * On `/login`, the backend creates `OTP_SESSION:{txId}` with a TTL of `OTP_SCREEN_TIMEOUT_MIN` (`15 minutes`).
   * Whenever the user performs an active API call on the OTP screen (`/verify-otp` wrong attempt or `/resend-otp`), the backend **refreshes the TTL of `OTP_SESSION:{txId}` back to 15 minutes** (sliding idle window).
   * If the user minimizes their browser or puts their laptop to sleep (which pauses JavaScript timers in Chrome/Edge) and clicks *Verify* or *Resend* after 16 minutes of inactivity, `OTP_SESSION:{txId}` will have vanished from Redis, and the backend immediately returns `401 Unauthorized` with `"Your session has timed out. Please log in again."`

---

## 2. Sending User Details in the `/login` (`202 Accepted`) Response

### The Requirement
The frontend needs the user's name and profile details in the `/api/auth/login` response when `202 Accepted` is returned so it can display the user's name on the Two-Factor Verification screen.

### AppSec-Safe Implementation (`OtpInitiatedResponseDTO`)
Because the user has only completed **Factor 1 (Password)** and has not yet verified their **Factor 2 (OTP)**, sending raw, unmasked `phoneNumber` or `email` inside the response would violate AppSec PII rules (since we are showing `maskedMobile` and `maskedEmail`).

Therefore, we have enriched `OtpInitiatedResponseDTO` with:
* `userId` (e.g., `"2488766"`)
* `userName` (Formatted full name, e.g., `"Amit Kumar Rao"`)
* `firstName`, `middleName`, `lastName`
* `userDetails` (A sanitized `UserTokenDataDto` object containing the user's name, role, and branch info, where `phoneNumber` and `email` are replaced with their masked values `***1234` and `a***o@sbi.co.in`).

---

## 3. Solving the "Back to Sign In" vs. "Resend OTP" UX Conflict

### The Problem You Identified
When we tied the 60-second cooldown and 3-resend quota to the `userId`, clicking **"Back to Sign In"** and clicking **"Sign In"** again on the login screen caused `/login` to throw the Resend error messages (`"Please wait {N} seconds before requesting another code."` or `"Maximum resend limit reached. Kindly Wait."`) right on the Sign-In page!

**Why this hurt User Experience:**
1. Suppose a user logs in at `10:00:00` and receives OTP `482910` on their phone.
2. At `10:00:10`, they accidentally click **"Back to Sign In"** (or hit browser Refresh).
3. At `10:00:15`, they re-enter their User ID and Password to get back to the OTP screen so they can type `482910`.
4. Blocking them on the Sign-In screen for 45 seconds prevented them from entering the valid OTP they already received!

### The Industry-Standard Solution: Smart Session Resumption + Quota Enforcement
Instead of blindly throwing a Resend error on the Sign-In screen when the user already has a valid OTP, the backend now distinguishes between **Resuming an Active OTP** vs. **Generating a New OTP**:

| User Action via "Back to Sign In" -> "Sign In" | Is Previous OTP Still Valid (< 5m & < 5 failures)? | How the Backend Behaves Now | UX Result on Frontend |
| :--- | :---: | :--- | :--- |
| **1. Re-logins within 60s Cooldown** (e.g., 20s after initial login) | **YES** (Valid OTP exists) | **Smart Session Resumption:** Backend invalidates the old `txId`, transfers the existing OTP hash to a new `txId`, **does NOT send a duplicate SMS/Email**, and returns `202 Accepted` with the **live remaining cooldown** (`resendCooldownSeconds: 40`) and `expiresInSeconds: 280`. | User smoothly enters the OTP screen, sees the Resend button locked for the remaining `40s`, and can immediately type the OTP they already received! |
| **2. Re-logins after 60s Cooldown** (within 10-min window, `resendsRemaining > 0`) | **YES or NO** | **Counts as 1 Resend:** Backend invalidates the old `txId`, increments the user's 10-min resend counter (`resendsRemaining` drops from `3` $\rightarrow$ `2`), generates a **new OTP**, starts a fresh `60s` cooldown, dispatches to Kafka, and returns `202 Accepted`. | User enters the OTP screen with a fresh OTP code and sees `resendsRemaining: 2` (preventing infinite OTP generation via "Back to Sign In"). |
| **3. Re-logins when `resendsRemaining == 0`** (All 3 resends used in 10m) | **YES** (4th OTP is still valid) | **Smart Session Resumption:** Backend transfers the valid 4th OTP to a new `txId`, sends **no new SMS/Email**, and returns `202 Accepted` with `resendsRemaining: 0` and `resendCooldownSeconds: 0`. | User is not locked out of the OTP screen! They can still enter their last valid 4th OTP code, while the Resend button stays disabled (`0 remaining`). |
| **4. Enters 5 Wrong Attempts in < 60s**, clicks "Back to Sign In" & "Sign In" | **NO** (Code burned by 5 wrong attempts) | **Blocks Cooldown Bypass (`429`):** Since they burned their code and `< 60s` has elapsed, backend blocks new OTP generation: `"Please wait {N} seconds before requesting another code."` | Prevents attackers from bypassing the 60-second cooldown after brute-forcing 5 wrong attempts. |
| **5. All 3 Resends Used AND 4th Code Expired/Burned**, clicks "Back to Sign In" & "Sign In" | **NO** (All 4 codes in 10m are dead) | **Blocks Quota Bypass (`429`):** Backend rejects with `"Maximum resend limit reached. Kindly Wait."` | User has completely exhausted 1 initial + 3 resends (20 wrong attempts or 4 expired codes) in 10 minutes and must wait for the 10m window to clear. |