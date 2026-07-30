---
title: "Time-Based Blind SQL Injection in REDACTED.com — A 20-Second Wait That Told Me Everything"
date: 2026-07-30 07:05:28 +0000
---

A write-up on finding and confirming a blind SQL injection vulnerability through simple response-time analysis.

| Target | Vulnerability | Severity | Status |
| :--- | :--- | :--- | :--- |
| `REDACTED.com` | SQL Injection (Time-Based Blind) | **Critical** | 🟢 Reported & Patched |

---

### 📌 The Setup

I was going through `REDACTED.com` the way I usually do — clicking around, watching how the app handled different parameters, not looking for anything specific yet. Most of the endpoints looked [...]

Then I noticed a product listing page that passed a `category_id` parameter directly in the URL:

```url
https://redacted.com/products?category_id=15
```

Nothing unusual about that on its own. Plenty of apps do this safely. But something about the way the page behaved when I changed the value — a slightly inconsistent error page when I sent a non[...]

### 🔍 First Signs

I tried the classic single-quote test:

```url
https://redacted.com/products?category_id=15'
```

The page didn't throw a visible SQL error, which normally means one of two things:
1. The input is being handled safely.
2. The errors are just suppressed and hidden from the response.

Given how the app reacted differently to malformed input versus normal input, I leaned toward the second option and decided to test for blind injection instead of giving up.

### 🧪 Confirming It

Since there was no visible feedback, I went with a time-based approach. If the parameter was actually vulnerable, I could get the database to confirm it indirectly — by making it pause before re[...]

I sent this payload:

```url
https://redacted.com/products?category_id=15+AND+SLEEP(20)--
```

The response took a little over 20 seconds to come back. 

```http
HTTP/1.1 200 OK
Content-Type: text/html; charset=UTF-8
X-Response-Time: 20.014s

[... Page Content ...]
```

I ran it again with a baseline request (`category_id=15` with no injection) right after, just to rule out server load or a fluke — that one came back instantly, like normal. Then I tried the pay[...]

That was enough to confirm it. The application was passing the `category_id` value straight into a SQL query without sanitizing or parameterizing it, and the database was executing whatever logic [...]

### ⚠️ Why This Matters

A time-based blind SQLi doesn't hand you data directly the way a UNION-based injection might, but it's just as dangerous — sometimes more, because it's quieter and easier to miss in logs. Once y[...]

* **Extract data character-by-character** using conditional time delays.
* **Enumerate database structure** (tables, columns, versions).
* **Escalate toward more damaging outcomes** depending on DB permissions (in some configurations).

The fact that the app suppressed error messages actually made this more interesting, not less — it just meant relying on time instead of error text to get confirmation.

### 📄 The Report

I documented the finding with:
* The vulnerable parameter and endpoint.
* The exact payload and observed response times across multiple requests.
* A short explanation of impact and potential blast radius.
* A suggested fix: parameterized queries / prepared statements instead of raw string concatenation, plus input validation as defense-in-depth.

The team acknowledged it quickly and patched it within their SLA. No data was extracted or accessed beyond confirming the vulnerability existed — the goal was always proof of concept, not exploi[...]

### 💡 Takeaway

This one's a good reminder that you don't always need a verbose error message to find SQLi. If a parameter feels off — even subtly — response timing is one of the most reliable signals you hav[...]

> **Note:** When testing time-based payloads on production applications, keep your sleep values reasonable (e.g., 5-10 seconds) to verify the vulnerability without locking up production database t[...]
