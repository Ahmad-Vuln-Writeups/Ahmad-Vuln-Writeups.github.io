---
title: "Time-Based Blind SQL Injection in REDACTED.com — A 20-Second Wait That Told Me Everything"
date: 2026-07-30
---

A write-up on finding and confirming a blind SQL injection vulnerability through simple response-time analysis.

### 📌 The Setup
I was going through REDACTED.com the way I usually do — clicking around, watching how the app handled different parameters, not looking for anything specific yet. 

Then I noticed a product listing page that passed a `category_id` parameter directly in the URL:
```url
https://redacted.com
```

### 🧪 Confirming It
Since there was no visible feedback, I went with a time-based approach. I sent this payload:
```url
https://redacted.com AND SLEEP(20)--
```

The response took a little over 20 seconds to come back. That was enough to confirm it.
