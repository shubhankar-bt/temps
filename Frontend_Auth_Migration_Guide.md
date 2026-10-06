# 🚀 Frontend Authentication Migration Guide: JWE & Single-Screen Login

To comply with our latest Information Security (ISD) requirements, the backend authentication architecture has been upgraded. 

This guide outlines the two major changes required on the frontend: moving to a Single-Screen Login and handling highly secure JSON Web Encryption (JWE) tokens.

---

## Part 1: The New "Single-Screen" Login Flow
We have removed the `/check-user` endpoint to prevent User ID Enumeration attacks. The user must now enter both their User ID and Password on the **same screen**.

### Step 1: App Initialization
When the Fincore application loads, make a `GET` request to the initialization endpoint.
*   **Endpoint:** `GET /api/auth/init`
*   **Action:** Save the `publicKey` from the response in your application's state/memory. You will need this to encrypt the password.

### Step 2: The Login Request
When the user clicks "Login", encrypt the password using the RSA Public Key (just like before), and send both the User ID and Encrypted Password in a single request.
*   **Endpoint:** `POST /api/auth/login`

### Step 3: Handle the HTTP Status Codes
The backend now dictates the UI routing based on strict HTTP status codes:
*   🟢 **`200 OK`**: Login Successful. (Proceed to Part 2 below to read the token).
*   🟡 **`202 ACCEPTED`**: MFA is enabled. The response contains a `transactionId`. Route the user to the OTP Screen.
*   🟠 **`428 PRECONDITION REQUIRED`**: The password has expired. Route the user to the Update Password Screen.
*   🔴 **`401 UNAUTHORIZED`**: Incorrect credentials. Show the error message.
*   🔴 **`403 FORBIDDEN`**: Account is locked or disabled. Show the error message.

---

## Part 2: Reading the JWE Token (Breaking Change)
To protect sensitive PII (Email, Phone, Roles), the token is no longer a standard Base64 JWS. It is now a **5-part encrypted JWE (JSON Web Encryption)** token.

### The Impact
1.  **You cannot use `jwt-decode` or `atob()`** to read the token anymore. It will just return gibberish.
2.  You must install a robust cryptographic library. We strongly recommend **`jose`**.

```bash
npm install jose
```

### How to Decrypt and Read the User Data
Upon receiving a `200 OK` from `/login` or `/verify-otp`, the backend sends the encrypted token in the body, and the **decryption key in the HTTP Headers**.

Here is the exact JavaScript/TypeScript implementation to read the user's profile:

```javascript
import { jwtDecrypt } from 'jose';

async function handleLoginSuccess(httpResponse) {
    // 1. Extract the Encrypted Token and the AES Key
    const encryptedToken = httpResponse.data.data.accessToken; 
    const base64AesKey = httpResponse.headers.get('X-Payload-Key'); // Crucial!

    if (!base64AesKey) {
        console.error("Missing Decryption Key in Headers!");
        return;
    }

    // 2. Convert the Base64 AES Key into a Cryptographic Buffer
    const secretKey = Uint8Array.from(atob(base64AesKey), c => c.charCodeAt(0));

    try {
        // 3. Decrypt the token in the browser's memory
        const { payload } = await jwtDecrypt(encryptedToken, secretKey);
        
        // 4. Success! You can now read the User Profile for routing/UI rendering
        console.log("User ID:", payload.userId);
        console.log("Role:", payload.roleName);
        console.log("Branch:", payload.branchCode);
        
        // 5. Store the RAW, ENCRYPTED token (Do not store the decrypted payload)
        localStorage.setItem('auth_token', encryptedToken);
        
    } catch (error) {
        console.error("Failed to decrypt the JWE token!", error);
        // Handle forced logout or error state
    }
}
```

### Making Downstream API Calls
For all subsequent API calls (e.g., fetching a dashboard), you **do not** need to do any encryption. 

Simply attach the raw, encrypted JWE token to your authorization header just like you used to. The backend microservices will handle the decryption automatically.

```javascript
headers: {
    'Authorization': `Bearer ${localStorage.getItem('auth_token')}`,
    'Content-Type': 'application/json'
}
```