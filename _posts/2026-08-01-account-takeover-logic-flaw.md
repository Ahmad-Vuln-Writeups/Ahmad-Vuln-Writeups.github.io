---
title: "The Password Reset Flow Was Solid — The Account Takeover Came From Somewhere Else Entirely on REDACTED.com"
date: 2026-08-01 11:10:00 +0300
categories: [Bug Bounty, Web Security]
tags: [logic-flaw, access-control, account-takeover, oauth, writeup]
---

A write-up on how a properly built reset flow still enabled account takeover, once combined with an unrelated feature that trusted it a little too much.

| Target | Vulnerability | Severity | Status |
| :--- | :--- | :--- | :--- |
| `REDACTED.com` | Account Takeover (Logic Flaw) | **Critical** | 🟢 Reported & Patched |

---

### 📌 Starting From "This Looks Fine," Again

By now this is basically my default assumption on well-run targets, and REDACTED.com's password reset flow earned it. I ran through the usual list of things that go wrong with reset functionality, and all of them were handled properly:

* Reset tokens were long, random, and single-use — no sequential IDs, no guessable patterns
* Tokens expired quickly (15 minutes) and were invalidated the moment a new reset was requested
* The reset confirmation endpoint checked that the token matched the specific account it was issued for — no swapping the email field in the request body to redirect a valid token to a different account
* Rate limiting kicked in after a handful of reset requests from the same IP, so brute-forcing tokens wasn't realistic either

Every individual piece of the flow, tested in isolation, did what it was supposed to do. Normally that's where I'd stop and move on to a different feature.

### 🔍 Widening the Frame

Instead of continuing to poke at the reset flow itself, I started looking at what else in the app trusted "user just completed a password reset" as a signal — because reset flows often sit at a trust boundary that other features quietly lean on, assuming the hard part (proving account ownership) already happened.

I found one: REDACTED.com's account settings page let a logged-in user change their email address, and — reasonably enough — required re-entering their current password to do it, as a safeguard against a hijacked session being used to silently change the account's recovery email. Standard, sensible control.

But I noticed that immediately after completing a password reset, the app auto-logged the user into a fresh session — a normal, convenience-focused UX choice. So the question became: does the email-change feature's "re-enter your password" requirement account for the fact that the password was just reset, or does it just check "is there a valid session," full stop?

### 🗺️ Testing the Seam

I requested a password reset for my own test account, completed it normally, and landed in the auto-logged-in session the app gave me afterward. Then, instead of doing anything else, I immediately went to account settings and tried to change the email address.

It asked me to confirm my current password. I entered the new one I'd just set during the reset. It accepted it and changed the email — which, on its own, is completely correct behavior. That's exactly what should happen for the legitimate account owner.

The actual problem showed up when I looked at how that password field was being validated. Watching the request in the proxy, the email-change confirmation wasn't calling the same password-verification logic the login page used — it was calling a lighter internal check that verified the password matched the current session's password state, which is subtly different from verifying it against a re-entered credential the way a real re-authentication step should. 

Immediately after a reset, that current password state was, well, whatever I'd just set it to seconds earlier. Which meant the "confirm your password" step was really just confirming I had access to the session — not independently confirming I was the legitimate account owner.

### 📥 Why That Matters

That distinction matters because it changes what's actually needed for account takeover. If I could get a reset flow to complete and get me an authenticated session on someone else's account, I wouldn't need to guess or steal their real password at all to then hijack their email — I'd just need to walk straight into the email-change flow using the throwaway password I'd just set during the reset itself, since the confirmation step never checked anything beyond "does this match the current session."

That still requires actually completing a reset for someone else's account, which the well-built parts of the flow (token randomness, expiry, per-account binding) were correctly preventing. So this wasn't yet a full chain on its own — it was a second failure that would only matter if the first, well-defended layer ever had a crack in it.

### ⚠️ Where the Real Gap Was

Digging further, REDACTED.com supported login via a linked third-party OAuth provider in addition to email/password. And their reset flow, it turned out, allowed initiating a password reset for any account, including ones that had been created via OAuth and had never set a password at all — the reset email still got sent, and completing it set an initial password on an account that previously only had OAuth-based login.

That itself isn't unheard of, but combined with the email-change gap, it created a real path: an account created and always accessed via "Sign in with Google," whose owner had no reason to ever expect a "reset your password" email to be meaningful, since they'd never set a password to begin with. 

If someone triggered a reset for that account and — this is the part that would require e.g. tricking the victim into clicking a legitimate-looking reset link sent to an email they still checked, or a scenario where the victim's email itself was separately compromised — completed it, they'd land in an authenticated session and could immediately pivot into changing the account's email via the weak confirmation step, fully locking the original owner out without ever needing their OAuth credentials or their actual account password.

### 📋 Scoping the Report Responsibly

I want to be clear about where I stopped: I demonstrated this entire chain against test accounts I controlled end-to-end, including one I configured specifically as an OAuth-only account to mirror the scenario. 

I did not attempt any part of this against a real user account, and the "victim clicks a reset link" step is a meaningful precondition I flagged clearly as a real limitation of the chain's practical exploitability — not something I glossed over to inflate severity.

### 📋 The Report

I documented:
* The two individually-reasonable-looking behaviors that combined into a real chain: reset flow working on OAuth-only accounts, and the email-change confirmation step using session-state validation instead of independent re-authentication
* A full walkthrough using only my own test accounts, with screenshots and request logs at each step
* Clear labeling of the one external precondition required (victim interacting with a reset email) as a scoped limitation, not a hidden assumption
* A recommended fix: have the email-change confirmation step re-validate credentials independently rather than trusting current session state, and reconsider whether password reset should be offered at all for accounts with no password set in the first place

### 🛠️ The Fix

The team treated this as critical and moved fast — within 48 hours they'd patched the email-change confirmation to require a fresh, independent password check rather than a session-state comparison. 

About a week later they also shipped a change disabling password reset entirely for OAuth-only accounts, redirecting those attempts to a page explaining the account uses third-party login instead.

### 💡 Takeaway

The most dangerous account takeover chains I've found are rarely one big broken thing — they're two or three individually reasonable design decisions that nobody thought to test together. A hardened reset flow and a hardened email-change flow can each pass review on their own and still create a real path when you look at what one silently hands the other.
