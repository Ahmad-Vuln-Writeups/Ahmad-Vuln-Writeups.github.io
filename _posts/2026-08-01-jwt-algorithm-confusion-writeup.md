---
title: "The JWT Implementation Was Solid — Until I Found the One Endpoint Still Trusting the Header on REDACTED.com"
date: 2026-08-01 11:30:00 +0300
categories: [Bug Bounty, Web Security]
tags: [jwt, algorithm-confusion, crypto, cryptography, writeup]
---

A write-up on an algorithm confusion vulnerability that only triggered on a legacy verification path most of the app had already moved past.

| Target | Vulnerability | Severity | Status |
| :--- | :--- | :--- | :--- |
| `REDACTED.com` | JWT Algorithm Confusion (RS256 → HS256) | **Critical** | 🟢 Reported & Patched |

---

### 📌 Starting From "This Looks Fine" Again

Same pattern as the admin panel piece — I want to be honest that most of what I tried here didn't work, because the main JWT handling on REDACTED.com was genuinely well implemented.

I decoded a token to see what I was working with:

```json
{
  "alg": "RS256",
  "typ": "JWT"
}
```

RS256 — asymmetric signing, public/private key pair. That's already a good sign, since the classic algorithm confusion attack (swapping `alg` to `HS256` and signing with the public key as if it were an HMAC secret) only works if the verification code doesn't explicitly check that the algorithm used matches what it expects. 

I tried it anyway, because it costs nothing to check:

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

...signed with the RSA public key treated as an HMAC secret. Sent it to the main API. Rejected — 401, invalid signature. The primary verification logic was explicitly pinning the expected algorithm rather than reading it from the token and trusting it, which is exactly the right way to do it. Good sign, dead end, moved on.

### 🔍 Widening the Search

Since the main API was solid, I went looking for anywhere else in the app that might be handling the same tokens independently — the assumption being that if there's more than one piece of code verifying JWTs, there's a decent chance they weren't all written or reviewed at the same time.

I found one: a password-reset confirmation flow that used a separate short-lived JWT, issued and verified through its own microservice rather than the main auth service. Made sense architecturally — reset tokens have different lifetime and claim requirements than session tokens, so a team splitting that logic out isn't unusual or sloppy on its face.

I pulled a reset token to look at it:

```json
{
  "alg": "RS256",
  "typ": "JWT"
}
```

Same algorithm as the main tokens. But same algorithm doesn't mean same verification code underneath, so I tested it the same way anyway.

### 🗺️ Testing the Reset Service

I took a valid reset token issued for my own test account, decoded it, and rebuilt it with the algorithm switched to `HS256` — signing it using the RSA public key (which, since it's a public key, was retrievable from the app's own `/.well-known/jwks.json` endpoint) as the HMAC secret.

Sent it to the password-reset confirmation endpoint. It was accepted.

### 📥 Why Only This One Endpoint

This is the part worth slowing down on, because "the same team wrote inconsistent code" isn't quite what happened here — it was closer to a config default nobody revisited. 

The main auth service used a JWT library configured with an explicit allow-list of accepted algorithms (`["RS256"]`), which is the correct, hardened way to do it — the verification function refuses to even consider a token claiming a different algorithm, regardless of what the token's header says.

The reset service, being a smaller, more recently spun-off microservice, used the same JWT library but with its default configuration — which, like a lot of JWT libraries out of the box, read the algorithm from the token header itself rather than requiring the caller to pin it explicitly. Nobody had gone back and locked it down the way the main service was locked down, most likely because it was treated as a smaller, lower-priority internal service when it was built.

So: same signing keys, same library, two different configs, one secure and one not.

### ⚠️ Turning It Into Impact

A forged reset-confirmation token is only interesting if it lets me do something real, so I checked what claims the reset endpoint actually trusted. The token carried a `sub` claim (the user ID the reset was for) and an action claim confirming reset intent. Since I controlled the token contents entirely once I could self-sign it, I could set `sub` to any user ID I wanted.

To confirm impact without actually taking over another account, I forged a token with my own `sub` value again, just using a completely fabricated signature path — proving I could produce an accepted token from scratch rather than replaying a legitimate one. 

That was enough to demonstrate that the same forged-token technique would work with any `sub` value, including someone else's, without needing to actually complete a takeover of an account that wasn't mine.

### 📋 The Report

I included:
* Both JWT verification paths, showing the config difference (explicit algorithm allow-list vs. library default)
* The forged token and the accepted response from the reset-confirmation endpoint
* The JWKS endpoint used as the public key source, since that's what made the forgery possible without needing to steal or guess anything private
* Clear notes that testing stopped at proving forgery was possible against my own account, and did not extend to targeting other users' `sub` values
* A recommended fix: explicitly pin the accepted algorithm in the reset service's JWT verification config, matching what the main auth service already did, and audit any other services sharing the signing keys for the same default-config issue

### 🛠️ The Fix

The team's response here was broader than just patching the one service — understandably, since "the same signing keys are used in multiple places with inconsistent verification configs" is the kind of finding that makes you want to check everything else touching those keys too. 

They pinned the algorithm explicitly across all services within a few days, and mentioned they were moving toward a shared internal verification library specifically so this kind of config drift couldn't happen between services again.

### 💡 Takeaway

Algorithm confusion isn't dead just because a target's main API handles it correctly — the same signing keys often get reused across multiple services, and each one is only as safe as its own config. On a mature target, the productive question usually isn't "is the main login flow vulnerable" (it probably isn't), it's "where else do these tokens get verified, and did that code get the same review."
