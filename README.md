# OTP Bypass via Response Manipulation — Signup Flow

A writeup documenting an OTP (One-Time Password) verification bypass found during
authorized testing of a signup flow, where the server's response to an **incorrect**
OTP still contained fields that allowed the client to treat the account as verified.

> **Target in this writeup is fictionalized.** All domains, endpoints, tokens,
> emails, and phone numbers below are fabricated for illustration. No real company,
> session data, or user data is referenced.

---

## Lab / Target Context

- Target: a fictional fintech signup flow (`veripay-example.test`), used here purely
  to illustrate the vulnerability class and testing methodology.
- Tooling: Burp Suite Community Edition, intercepting proxy mode.
- Scope: tested only against an account created by the tester, in an authorized
  context (bug bounty program / explicit permission). **Never test a live
  application you do not have written authorization to assess.**

## Vulnerability Class

Many signup/verification flows send an OTP to an email and/or phone number, then
call a "verify OTP" endpoint. The vulnerability here is a class of **trust-boundary
failure**: the server's response to a *failed* OTP check still includes fields the
client-side code uses to decide whether verification succeeded — and those fields
don't match the actual outcome.

In this case, submitting a deliberately wrong OTP produced a response where:

- `"Success": true`
- `"Message": "Invalid OTP"`
- `"EmailVerified": false`
- `"MobileVerified": false`

The top-level `Success: true` combined with a `200 OK` status meant the client-side
JavaScript treated the call as having succeeded, while the "Invalid OTP" message and
`false` verification flags were only used for *display* — they weren't enforced
server-side on the next step. By intercepting and adjusting the response fields
(or replaying a crafted request), it was possible to make the client proceed past
OTP verification without ever submitting a correct code.

## Methodology (Step by Step)

1. **Start the signup flow** and reach the "Verify OTP" step, where OTPs are sent
   to both an email address and a phone number.
2. **Turn on Burp Suite's intercepting proxy** and submit a deliberately incorrect
   OTP for one or both fields.
3. **Inspect the raw response.** Example (fictional values):

   ```http
   POST /accounts/signup-emailotp.aspx/VerifyEmailOtp HTTP/2
   Host: veripay-example.test
   Content-Type: application/json; charset=utf-8

   { "otp": "0000" }
   ```

   ```json
   HTTP/2 200 OK
   Content-Type: application/json

   {
     "d": {
       "__type": "VERIPAY.DTO.VerifyOtpResponse",
       "Success": true,
       "Message": "Invalid OTP",
       "EmailVerified": false,
       "MobileVerified": false
     }
   }
   ```

4. **Identify the inconsistency**: `Success: true` at the top level, despite the
   OTP being wrong and both verification flags being `false`.
5. **Manipulate the response** in Burp (right-click → "Do intercept" → forward a
   modified response, or use Match & Replace) to flip `EmailVerified` and
   `MobileVerified` to `true` before it reaches the browser.
6. **Observe client behavior**: the frontend's "Verified ✅" indicator appeared for
   both fields despite no correct OTP ever being submitted, and the "Verify OTP"
   button proceeded to the next step of the signup flow.

## Root Cause

- The server computed the correct verification result (`false`/`false`) but
  **did not gate the client's forward progress on that result alone** — the
  top-level `Success` flag and the displayed state were not cross-checked
  server-side on the next request.
- Verification state was effectively **trusted from client-visible response
  data**, rather than being re-validated server-side when the signup flow
  advanced to the next step.
- No rate limiting was observed on OTP submission attempts, which would also
  allow brute-forcing of a 4-digit OTP independent of this specific bypass.

## Remediation

- Re-validate verification status **server-side** on every step that depends on
  it (e.g., account creation), never trusting a client-supplied or
  client-displayed "verified" state.
- Return a single, consistent, minimal response on failure (no mix of
  `Success: true` with a failure message) — ideally a non-200 status code for
  failed verification.
- Invalidate the OTP after each failed attempt beyond a small threshold, and
  apply rate limiting / exponential backoff per account and per IP.
- Expire OTPs quickly (e.g., 5–10 minutes) and invalidate immediately after
  first successful use.

## Disclaimer

This writeup describes a vulnerability class and testing methodology for
educational purposes. All identifying details (domain, tokens, user data) have
been fictionalized or redacted. Only test applications you have explicit,
written authorization to assess.

---

## Tags

`cybersecurity` `web-security` `otp-bypass` `burpsuite` `owasp` `api-security`
