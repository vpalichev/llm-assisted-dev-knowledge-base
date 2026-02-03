## The Pattern

HTTP status codes follow a simple structure. The first digit tells you the category:

|Range|Category|Meaning|
|---|---|---|
|**1xx**|Informational|"Hold on, still processing..."|
|**2xx**|Success|"All good!"|
|**3xx**|Redirection|"Go look over there"|
|**4xx**|Client Error|"You made a mistake"|
|**5xx**|Server Error|"We made a mistake"|

---

## 2xx — Success

|Code|Name|When You'll See It|
|---|---|---|
|**200**|OK|The standard "everything worked" response|
|**201**|Created|A POST request successfully created a new resource|
|**202**|Accepted|Request received, but processing isn't complete yet (async operations)|
|**204**|No Content|Success, but nothing to return (common for DELETE)|

---

## 3xx — Redirection

|Code|Name|When You'll See It|
|---|---|---|
|**301**|Moved Permanently|Resource has a new permanent URL; update your bookmarks|
|**302**|Found|Temporary redirect; keep using the original URL|
|**304**|Not Modified|Caching response — "use what you already have"|
|**307**|Temporary Redirect|Like 302, but guarantees the HTTP method won't change|
|**308**|Permanent Redirect|Like 301, but guarantees the HTTP method won't change|

---

## 4xx — Client Errors

|Code|Name|When You'll See It|
|---|---|---|
|**400**|Bad Request|Malformed syntax, invalid request|
|**401**|Unauthorized|Authentication required (who are you?)|
|**403**|Forbidden|Authenticated but not permitted (you can't do that)|
|**404**|Not Found|The classic — resource doesn't exist|
|**405**|Method Not Allowed|Wrong HTTP verb (e.g., POST to a GET-only endpoint)|
|**409**|Conflict|Request conflicts with current state (duplicate, edit collision)|
|**410**|Gone|Resource existed once but has been permanently removed|
|**422**|Unprocessable Entity|Valid syntax but semantically wrong (validation errors)|
|**429**|Too Many Requests|Rate limited — slow down!|

---

## 5xx — Server Errors

|Code|Name|When You'll See It|
|---|---|---|
|**500**|Internal Server Error|Generic "something broke on our end"|
|**501**|Not Implemented|Server doesn't support the requested functionality|
|**502**|Bad Gateway|Proxy/gateway got an invalid response from upstream|
|**503**|Service Unavailable|Server is overloaded or under maintenance|
|**504**|Gateway Timeout|Proxy/gateway didn't get a timely upstream response|

---

## Quick Reference Card

```
2xx  ✓  Success         "Here's your stuff"
3xx  →  Redirect        "It's over there now"
4xx  ✗  Client Error    "You asked wrong"
5xx  ⚠  Server Error    "We broke something"

Most common in practice:
  200  OK
  201  Created
  301  Moved Permanently
  400  Bad Request
  401  Unauthorized (need to log in)
  403  Forbidden (logged in, but no access)
  404  Not Found
  500  Internal Server Error
```

---

# More explanations:

# Understanding HTTP Status Codes

When your browser or application makes a request to a server, the server always responds with a three-digit number that tells you what happened. Think of it like a doctor giving you a diagnosis — the number is a standardized way of communicating the outcome so that any client, anywhere in the world, knows exactly what occurred.

The genius of HTTP status codes lies in their structure. You don't need to memorize all of them because the first digit tells you the category. If the code starts with a 2, something good happened. If it starts with a 4, you (the client) did something wrong. If it starts with a 5, the server made a mistake. This means that even if you encounter a code you've never seen before, you can immediately understand the general situation.

---

## The Success Family (2xx)

The **200 OK** response is the workhorse of the web. Every time you successfully load a webpage, every time an API call returns data, every time things just _work_, you're seeing a 200 behind the scenes. It simply means "your request was received, understood, and fulfilled."

**201 Created** is what you get when your request didn't just succeed — it also brought something new into existence. When you sign up for a website and your account gets created, or when you POST a new record to a database, the server responds with 201 to say "not only did that work, but I made a new thing for you." It's a subtle but meaningful distinction from 200.

**204 No Content** is an interesting one. It means "success, and I have nothing to say." This is common for DELETE operations — you ask the server to remove something, it does, and there's simply nothing to return. The empty response _is_ the confirmation.

---

## The Redirection Family (3xx)

Redirects tell your client "the thing you want isn't here, but I know where it is."

**301 Moved Permanently** is a statement of fact about the internet's geography. It says "this resource has moved to a new address, and it's never coming back. Update your bookmarks, update your links, tell search engines — the new location is the real location now." Browsers and search engines take this seriously and will remember the new URL.

**302 Found** (often called a temporary redirect) is more casual. It says "go over there for now, but keep my original address handy because things might change." The resource is temporarily available elsewhere, but the original URL is still considered the canonical one.

**304 Not Modified** is the server's way of saving bandwidth. When your browser already has a cached copy of something, it can ask the server "has this changed since I last grabbed it?" If the answer is no, the server sends back a 304 instead of re-sending the entire resource. It's a polite "you already have the latest version."

---

## The Client Error Family (4xx)

These codes all mean one thing: the problem is on your end.

**400 Bad Request** is the generic "I can't understand what you're asking for." Maybe the JSON was malformed, maybe a required parameter was missing, maybe the syntax was just wrong. The server tried to parse your request and failed.

**401 Unauthorized** and **403 Forbidden** are often confused, but they represent different situations. Think of 401 as a bouncer asking "who are you?" — you haven't proven your identity yet, or your credentials are invalid. The solution is to authenticate (log in, provide an API key, etc.). A 403, on the other hand, is the bouncer saying "I know exactly who you are, and you're not allowed in." You're authenticated, but you lack permission for this particular resource. No amount of re-authenticating will help; you simply don't have access.

**404 Not Found** is perhaps the most famous status code, familiar even to people who know nothing about HTTP. The resource you requested doesn't exist at this URL. Maybe it never did, maybe it was deleted, maybe you mistyped the address. The server looked and found nothing.

**422 Unprocessable Entity** is subtly different from 400. With a 400, the server couldn't even parse your request — the syntax was broken. With a 422, the syntax was fine, but the _content_ doesn't make sense. Imagine submitting a form where the "email" field contains "hello world" instead of an actual email address. The server understood the request perfectly; it just can't accept data that violates its rules.

**429 Too Many Requests** is the server protecting itself. You've hit a rate limit — sent too many requests in too short a time — and the server is asking you to slow down. Good APIs will often include headers telling you how long to wait before trying again.

---

## The Server Error Family (5xx)

These codes are the server's way of saying "this is our fault, not yours."

**500 Internal Server Error** is the catch-all for "something went wrong and we don't have a more specific code for it." An unhandled exception, a bug in the code, an unexpected condition — anything that causes the server to fail in an unplanned way typically results in a 500. As a client, there's usually nothing you can do except try again later and hope the server's operators fix the problem.

**502 Bad Gateway** and **504 Gateway Timeout** both involve servers talking to other servers. Modern web applications often have multiple layers: a load balancer talks to an application server, which talks to a database, which might talk to a cache. When one of these intermediate servers (acting as a "gateway" or "proxy") can't get a valid response from the next server in the chain, you get a 502. If it's specifically because the upstream server took too long to respond, you get a 504.

**503 Service Unavailable** means the server is temporarily unable to handle requests, usually because it's overloaded or undergoing maintenance. Unlike a 500 (which suggests something is broken), a 503 implies the situation is expected to be temporary. The server knows it can't help right now and is being honest about it.

---

## The Mental Model

The most useful thing to internalize is the pattern: **2 means yes**, **3 means go elsewhere**, **4 means you erred**, **5 means we erred**. With that framework, you can reason about any status code you encounter, even unfamiliar ones. A 418? Must be some kind of client error. A 507? Server-side problem. The first digit is your compass.