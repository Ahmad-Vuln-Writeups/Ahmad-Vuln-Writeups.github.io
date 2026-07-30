---
title: "How a Broken Search Bar Led to a Full Database Dump on REDACTED.com"
date: 2026-07-30 13:00:00 +0300
categories: [Bug Bounty, Web Security]
tags: [sqli, union-injection, writeup]
---

A write-up on finding a UNION-based SQL injection through a feature nobody thinks to test twice.

Target: REDACTED.com
Vulnerability: SQL Injection (UNION-Based)
Severity: Critical
Status: Reported and patched

Where It Started

I'll be honest, this one wasn't some deep, calculated hunt. I was testing the search bar on REDACTED.com because search bars are, in my experience, criminally under-tested by dev teams. Everyone locks down the login form and the payment flow, and then the search box quietly passes raw user input into a query somewhere in the backend.

The search endpoint looked like this:

```url
https://redacted.com/search?q=laptop
```

Standard stuff. I typed a normal query first just to see the expected response shape — number of results, how items were displayed, that kind of thing. Then I broke it.

Breaking It

I threw in a single quote:

```url
https://redacted.com/search?q=laptop'
```

And got a 500 error with a stack trace in the response. Not just a generic "something went wrong" page — an actual database error, mentioning MySQL syntax, right there in the HTML.

That's basically a neon sign. Verbose SQL errors are a gift, because they usually tell you the exact syntax the backend is choking on, which massively speeds up figuring out how to build a working payload.

Mapping the Query

Before jumping to UNION, I needed to figure out how many columns the original query was selecting, since UNION-based injection requires matching the column count exactly. I used the classic ORDER BY trick, increasing the number until the page broke:

```sql
q=laptop'+ORDER+BY+1--
q=laptop'+ORDER+BY+2--
q=laptop'+ORDER+BY+3--
...
q=laptop'+ORDER+BY+7--
```

Six worked fine, seven threw an error. So the query was selecting six columns.

Getting Data Back

With the column count known, I tried a basic UNION SELECT to see which columns actually reflected values back onto the page:

```sql
q=laptop'+UNION+SELECT+1,2,3,4,5,6--
```

The response rendered the numbers 2 and 4 in place of where the product name and description normally showed up — meaning those two columns were being echoed directly onto the page. That's exactly what you want, because it means I could swap those positions for real data instead of placeholder integers.

From there it was just a matter of asking the database questions:

```sql
q=laptop'+UNION+SELECT+1,table_name,3,4,5,6+FROM+information_schema.tables--
```

That returned a list of table names in the database — including one that stood out immediately: users.

I stopped there. I confirmed I could enumerate table and column names, which was more than enough to prove impact. I did not query the users table itself or pull any actual user data — that's a line I don't cross in testing, since it turns "proof of concept" into "unauthorized data access."

Why This One Was Bad

This wasn't a quiet, theoretical bug. If someone with less restraint found this before I did, they'd have had a clear path to:

* Dumping the entire users table — emails, hashed passwords, possibly more
* Enumerating every other table in the schema
* Depending on DB permissions, potentially reading files off the server via LOAD_FILE()

Combine a verbose error message with an unsanitized parameter, and you've basically handed an attacker a map and the keys.

The Fix

In my report I flagged two separate issues, because they compound each other:

The root cause — raw string concatenation in the search query instead of parameterized queries
The verbose error output — stack traces and DB errors should never be shown to end users, full stop

The team fixed the query first (fast turnaround, which I appreciated) and followed up a week later with a patch that suppressed detailed error messages app-wide, replacing them with generic error pages.

Takeaway

Search bars, filters, sort parameters — anything that touches a database query but doesn't feel "sensitive" like a login form is exactly where I'd point a new bug hunter to start looking. Nobody thinks twice about testing them, which is precisely why they're worth testing first.
