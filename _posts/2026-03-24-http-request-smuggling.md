---
title: HTTP Request Smuggling
date: 2026-03-24
categories: [Web Security, HTTP]
tags: [request-smuggling, desync, burp, http2, web-security]
---

# HTTP Request Smuggling Notes

---

## Table of Contents
1. [What is HTTP Request Smuggling?](#1-what-is-http-request-smuggling)
2. [How Does It Work?](#2-how-does-it-work)
3. [How Vulnerabilities Arise](#3-how-vulnerabilities-arise)
4. [Attack Types](#4-attack-types)
5. [Detection Techniques](#5-detection-techniques)
6. [Exploitation Scenarios](#6-exploitation-scenarios)
7. [HTTP/2 and Advanced Variants](#7-http2-and-advanced-variants)
8. [Tools](#8-tools)
9. [Real Bug Reports](#9-real-bug-reports)
10. [Prevention](#10-prevention)
11. [Resources](#11-resources)

---

## 1. What is HTTP Request Smuggling?

HTTP request smuggling is a technique used to interfere with the way a web site processes sequences of HTTP requests. It exploits the **inconsistency in parsing non-RFC-compliant HTTP requests** between two HTTP devices — typically a front-end proxy and a back-end server.

It is critical in nature because it can:
- Bypass security controls (WAF, access controls)
- Gain unauthorized access to sensitive data
- Directly compromise other application users
- Enable cache poisoning, XSS, session hijacking, and credential theft

> Request smuggling is primarily associated with HTTP/1 requests, however HTTP/2 can also be vulnerable depending on the backend architecture — particularly when HTTP/2 downgrading to HTTP/1.1 is in play.

---

## 2. How Does It Work?

### The Infrastructure

Modern web apps typically sit behind at least one front-end server that forwards traffic to back-end servers:

```
User → [Front-end: Load Balancer / Reverse Proxy / CDN] → [Back-end Server]
```

Both front-end and back-end **share the same TCP/TLS connection** — this is the root of the problem. In HTTP/1.1, the TLS tunnel is reused for many connections (keep-alive), meaning multiple users' requests flow through the same channel. When the front-end and back-end disagree on where one request ends, the leftover bytes become the **start of the next user's request**.

```
                   ┌─────────────────────────┐
                   │    SHARED CONNECTION     │
User A ────────►   │  [Req A][Req B][Req C]  │  ◄──── Users share this channel
User B ────────►   │                         │
                   └────────────┬────────────┘
                                │
                         ┌──────▼──────┐
                         │  BACKEND    │
                         │  Server     │
                         └─────────────┘
```

The attacker causes part of their request to be interpreted by the back-end as the **start of the next request** — effectively prepending a malicious prefix to whatever the next innocent user sends.

![[assets/Attachments/Pasted image 20260308033130.png]]
![[assets/Attachments/Pasted image 20260308033240.png]]

---

## 3. How Vulnerabilities Arise

### HTTP/1 — Two Ways to Specify Request Length

In HTTP/1.1 there are two headers that define where a request body ends:

**`Content-Length`** — specifies the exact byte size of the body, sent as a single block:
```http
POST /search HTTP/1.1
Host: normal-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 11

q=smuggling
```

**`Transfer-Encoding: chunked`** — body is sent in chunks, each preceded by its size in hex, terminated by a `0` chunk:
```http
POST /search HTTP/1.1
Host: normal-website.com
Content-Type: application/x-www-form-urlencoded
Transfer-Encoding: chunked

b
q=smuggling
0

```

> **Note:** `Content-Length` does **not** count the headers — only the body bytes. Burp automatically adds/updates it when you edit a request body. Burp also auto-unpacks chunked encoding to make messages easier to read, which is why many testers miss it.

> **Note:** Every request must end with a newline (`\r\n`). PUT and POST requests must have `Content-Length` or `Transfer-Encoding`.

### The Conflict

When **both headers appear in the same request**, servers must decide which one to trust. The RFC says to prefer `Transfer-Encoding` and ignore `Content-Length` — but:
- Some servers don't support `Transfer-Encoding` in requests at all
- Some servers that do support it can be **tricked into ignoring it** via header obfuscation

If the front-end and back-end handle this conflict differently → **request smuggling vulnerability**.

### RFC Compliance and the "MAY/SHOULD" Problem

RFCs are the rules servers use to handle requests (RFC = Request for Comments). They define how a protocol should be implemented — but they're filled with `MAY` and `SHOULD` rather than `MUST`, leaving room for differing implementations. This is exactly where smuggling lives.

Example quirk: if there's a **space between the colon and the header name** (e.g., `Content-Length : 11`), some back-end servers return `400 Bad Request` and start a new request from the remaining bytes — while the front-end passes it through without complaint.

```
Without space: Content-Length: 11  ← front-end and back-end both accept it
With space:    Content-Length : 11 ← front-end accepts, back-end rejects → desync
```

![[assets/Attachments/Pasted image 20260308213939.png]]
![[assets/Attachments/Pasted image 20260308214006.png]]

### HTTP/2 — One Length Mechanism (Usually Safe)

In HTTP/2 there is only one built-in length mechanism — no `Content-Length` vs `Transfer-Encoding` ambiguity. This makes **pure HTTP/2 end-to-end connections immune to classic request smuggling**.

However, if the **front-end is HTTP/2 but the back-end is HTTP/1.1** (HTTP downgrading), a new attack surface opens — see [Section 7](#7-http2-and-advanced-variants).

---

## 4. Attack Types

> **Important for Burp:** Browsers and Burp use HTTP/2 by default if the server supports it. To test HTTP/1.1 smuggling, manually switch to HTTP/1.1 in Burp Repeater via **Request attributes** in the **Inspector** panel.

All attacks depend on how both the front-end and back-end interpret the ambiguous headers. There is no CL.CL variant because all servers support `Content-Length` — if both use it, there's no conflict.

---

### CL.TE

**Front-end trusts `Content-Length` → Back-end trusts `Transfer-Encoding`**

```http
POST / HTTP/1.1
Host: vulnerable-website.com
Content-Length: 13
Transfer-Encoding: chunked

0

SMUGGLED
```

**What happens:**
- Front-end reads `Content-Length: 13` → forwards the full 13 bytes (`0\r\n\r\nSMUGGLED`) to the back-end
- Back-end reads `Transfer-Encoding: chunked` → processes the `0` chunk as end-of-request → treats `SMUGGLED` as the start of the **next** request

![[assets/Attachments/Pasted image 20260308172941.png]]
![[assets/Attachments/Pasted image 20260308173019.png]]
![[assets/Attachments/Pasted image 20260308173036.png]]

> `Content-Length: 13` = the full body including `0\r\n\r\nSMUGGLED` — count carefully, the `0` chunk + blank line + `SMUGGLED` = 13 bytes.

---

### TE.CL

**Front-end trusts `Transfer-Encoding` → Back-end trusts `Content-Length`**

```http
POST / HTTP/1.1
Host: vulnerable-website.com
Content-Length: 3
Transfer-Encoding: chunked

8
SMUGGLED
0

```

**What happens:**
- Front-end reads `Transfer-Encoding: chunked` → sees the `8` chunk (`SMUGGLED`) and the terminating `0` → forwards the full chunked body to back-end
- Back-end reads `Content-Length: 3` → only reads `8\r\n` (3 bytes) → treats `SMUGGLED\r\n0\r\n\r\n` as the start of the **next** request

> Key difference from CL.TE:
> - In CL.TE: `Content-Length` = size of **entire body including the smuggled part** — the `0` chunk comes **before** the malicious request
> - In TE.CL: `Content-Length` = size of the **first line only** — the `0` chunk comes **after** the malicious request

![[assets/Attachments/Pasted image 20260310025117.png]]
![[assets/Attachments/Pasted image 20260310025242.png]]

---

### TE.TE

**Both servers trust `Transfer-Encoding` — but one can be tricked into ignoring it via obfuscation**

Since both servers normally use `Transfer-Encoding`, the attacker must **obfuscate the header** so that one server rejects it and falls back to `Content-Length`. This turns it into either a CL.TE or TE.CL attack depending on which server was fooled.

Common obfuscation techniques:
```http
Transfer-Encoding: xchunked
Transfer-Encoding: chunked
Transfer-Encoding: chunked
Transfer-Encoding: x
Transfer-Encoding:[tab]chunked
[space]Transfer-Encoding: chunked
X: X[\n]Transfer-Encoding: chunked
Transfer-Encoding
 : chunked
```

![[assets/Attachments/Pasted image 20260310030556.png]]

---

### CL.0

A newer variant where the **back-end ignores the request body entirely** (treats `Content-Length` as 0) for certain endpoints or when specific headers trigger an error response without consuming the socket bytes.

- Front-end uses `Content-Length` normally
- Back-end ignores the body and starts reading from it as the next request

Testing for CL.0: send a request with a partial second request in the body, then send a normal follow-up. If the follow-up returns a 404 (or a response matching the smuggled path), you have a CL.0.

> Refer to PortSwigger's dedicated CL.0 page: [https://portswigger.net/web-security/request-smuggling/browser/cl-0](https://portswigger.net/web-security/request-smuggling/browser/cl-0)

---

### Response Queue Poisoning

The most dangerous scenario. Instead of smuggling a **partial** request (a prefix), the attacker smuggles a **complete** second request. This forces the server to serve that second request's response to the **next innocent user** instead of the attacker.

With HTTP/1.1 persistent connections, multiple users share the same response queue. By poisoning it:
1. Attacker sends a request that smuggles a full second request
2. Server processes both and puts two responses in the queue
3. The attacker gets the first response (their own)
4. The **next legitimate user gets the attacker's second response** — which could be an admin panel, another user's session, or anything the attacker crafted

![[assets/Attachments/Pasted image 20260310030921.png]]
![[assets/Attachments/Pasted image 20260310030935.png]]
![[assets/Attachments/Pasted image 20260310030950.png]]

---

## 5. Detection Techniques

### Method 1 — Time Delay (Safest, non-destructive)

Send a request designed to hang the back-end if the smuggling type matches:

**Detecting CL.TE** (front-end trusts CL, back-end trusts TE):
```http
POST / HTTP/1.1
Host: vulnerable-website.com
Transfer-Encoding: chunked
Content-Length: 4

1
A
X
```
If CL.TE exists, the back-end waits for the rest of the `X` chunk that never arrives → **time delay**.

**Detecting TE.CL** (front-end trusts TE, back-end trusts CL):
```http
POST / HTTP/1.1
Host: vulnerable-website.com
Transfer-Encoding: chunked
Content-Length: 6

0

X
```
If TE.CL exists, the back-end reads 6 bytes as `Content-Length` but the chunked encoding terminated at `0` → **time delay**.

### Method 2 — Differential Responses

Send two requests:
1. A request that smuggles a prefix designed to poison the next request
2. A normal "innocent" follow-up request

If the follow-up returns an **unexpected response** (404, error, or a different page) → the smuggled prefix was prepended to it → **confirmed vulnerability**.

```
Request 1: smuggles GET /404page
Request 2: normal GET /
← If response 2 is 404, the vulnerability is confirmed
```

### Method 3 — Quickest Probe (from James Kettle's research)

Start clean:
1. Strip all unnecessary headers — get the simplest possible working request
2. Switch to POST
3. Set `Content-Length: 0` and add `Transfer-Encoding: chunked`

**If the front-end uses CL:** the request will have no body and the back-end won't respond to the chunked data — you'll get **no response / timeout** → confirms front-end trusts CL.

Then swap and test the back-end behavior with TE.

![[assets/Attachments/Pasted image 20260309212630.png]]
![[assets/Attachments/Pasted image 20260309212943.png]]

### Method 4 — Using Wireshark

Capture traffic and inspect the raw TCP stream. If the server splits your request into two parts (first part rejected, second part starts a new pipeline request), you're looking at a desync. The malicious prefix sits in the pipeline waiting for the next legitimate request to push it through.

![[assets/Attachments/Pasted image 20260308214349.png]]
![[assets/Attachments/Pasted image 20260308214615.png]]

> **False Positives:** Not every timing anomaly is a real vulnerability. Refer to: [HTTP Request Smuggling - False Positives](https://youtu.be/7wq2e2nxa38)

---

## 6. Exploitation Scenarios

### Bypass Front-End Security Controls

If the front-end enforces access controls (e.g., blocks access to `/admin`), smuggle a request directly to the back-end that bypasses those rules entirely.

### Reflected XSS → Stored XSS

If the application has a reflected XSS point, request smuggling can turn it into **stored XSS** that fires against the next victim user without any interaction needed.

![[assets/Attachments/Pasted image 20260308215815.png]]

> Normally reflected XSS requires the victim to click a malicious link. With request smuggling, the XSS payload is prepended to the next real user's request — **no link needed**.

### Capture Other Users' Requests

Smuggle a partial request that routes the next user's request to an endpoint that reflects it back (e.g., a search field or profile page). You receive the victim's full HTTP request — including their cookies, tokens, and credentials.

### Account Takeover via Cookie Hijacking

Demonstrated in [كيف قدرت أخترق اي حساب](https://youtu.be/v_CUm93Pcik):

![[assets/Attachments/Pasted image 20260308221918.png]]
![[assets/Attachments/Pasted image 20260308222645.png]]

### Cache Poisoning

Smuggle a request with a malicious response that gets cached and served to all future visitors of a given URL.

### WAF Bypass

The WAF (typically at the front-end) validates the full request as clean. The smuggled fragment is invisible to it — only the back-end sees and processes it.

### Header Trick — `X-Ignore-Me`

To control which part of the next request gets processed and which gets ignored, use:
```http
X-Ignore-Me: [injected content here]
```
This swallows part of the next victim's request into a header value, hiding it from the app while you control the rest.

---

## 7. HTTP/2 and Advanced Variants

### HTTP/2 — Why It's Different

HTTP/2 is a **binary protocol** — it doesn't use CRLF (`\r\n`) as delimiters. It has a built-in length mechanism per frame, making pure HTTP/2 immune to classic CL/TE conflicts.

However, many deployments use HTTP/2 only at the edge (front-end) and downgrade to HTTP/1.1 for the back-end. This creates a new attack surface.

### HTTP/2 Downgrade Smuggling (H2.CL / H2.TE)

When the front-end converts an HTTP/2 request to HTTP/1.1 for the back-end, it must reconstruct the HTTP/1.1 headers. If an attacker injects a `Content-Length` or `Transfer-Encoding` header via HTTP/2 pseudo-headers, the front-end may pass them through, causing desync at the back-end.

```
Attacker (HTTP/2) → Front-end → (converts to HTTP/1.1) → Back-end
                              ↑
                    Injected headers survive here
```

### CRLF Injection in HTTP/2 (H2.CRLF)

Since HTTP/2 treats `\r\n` as regular data (not as delimiters), an attacker can inject `\r\n` sequences into HTTP/2 header values. When the front-end converts to HTTP/1.1, it interprets these as **new header lines** — injecting headers that were never in the original request.

Example: inject `foo: bar\r\nTransfer-Encoding: chunked` into a header value → the back-end sees a legitimate `Transfer-Encoding: chunked` header that the front-end never validated.

This type of corrupted request is sometimes called **"Kettled"** (after James Kettle who researched it).

> For more: [Advanced request smuggling | Web Security Academy](https://portswigger.net/web-security/request-smuggling/advanced)

### HTTP Pipelining and Keep-Alive

You need to understand these two concepts to fully grasp why request smuggling is possible:

- **HTTP/1.0:** Each request opens and closes a new TCP/TLS connection — no shared state
- **HTTP/1.1 Keep-Alive:** A single TLS tunnel is reused for multiple requests from multiple users — this is what creates the shared queue that smuggling exploits

![[assets/Attachments/Pasted image 20260309212630.png]]

> No tool gives you a server-side view of the connection — to truly see the traffic as the server sees it, you need to be at the server level (Wireshark on the server, or similar).

---

## 8. Tools

| Tool | Use |
|---|---|
| **Burp Suite + HTTP Request Smuggler extension** | Primary testing tool — automated scanning, manual payload crafting |
| **smuggler** (Python CLI) | Automated CL.TE / TE.CL detection via timing |
| **Wireshark** | See the raw TCP stream and how the server splits the request |
| **Burp Repeater** | Manual payload testing — switch to HTTP/1.1 in Inspector → Request attributes |

**Burp tips:**
- Switch protocol: Inspector panel → **Request attributes** → HTTP/1.1
- To use chunked encoding: Burp can convert the request body for you ![[assets/Attachments/Pasted image 20260308213202.png]]
- Burp auto-calculates `Content-Length` when you edit the body ![[assets/Attachments/Pasted image 20260308213219.png]]
- Result of adding chunked encoding: ![[assets/Attachments/Pasted image 20260308213231.png]]

**DEF CON 27 — James Kettle "HTTP Desync Attacks: Smashing into the Cell Next Door":**

![[assets/Attachments/Pasted image 20260324111752.png]]
![[assets/Attachments/Pasted image 20260324111831.png]]

---

## 9. Real Bug Reports

- [HackerOne #726773](https://hackerone.com/reports/726773)
- [HackerOne #1063627](https://hackerone.com/reports/1063627)

---

## 10. Prevention

| Fix | Details |
|---|---|
| **Use HTTP/2 end-to-end** | Eliminates CL/TE ambiguity entirely — don't downgrade to HTTP/1.1 at the back-end |
| **Reject ambiguous requests** | If both `Content-Length` and `Transfer-Encoding` are present, return `400 Bad Request` |
| **Normalize at the front-end** | Strip or rewrite conflicting headers before forwarding |
| **Prefer `Transfer-Encoding`** | RFC says: if both present, ignore `Content-Length` — enforce this consistently on all servers |
| **WAF rules** | Block requests with redundant HTTP headers (e.g., Imperva's "Redundant HTTP Headers" rule) |
| **Disable keep-alive** | Last resort — removes the shared tunnel, but kills performance |
| **Validate CRLF in headers** | For HTTP/2 deployments, strip `\r\n` from all header values before downgrading |

---

## 11. Resources

### Videos (Arabic)
- [HTTP Request Smuggling Explained [Arabic]](https://youtu.be/s4_A1qP41Zc) ⭐ — starts from RFC basics, Wireshark demo, very detailed
- [ما هي ثغرة ال HTTP Request Smuggling](https://youtu.be/WFXEF2ovqVA)
- [كيف قدرت أخترق اي حساب من خلال أستغلال ثغرة Request Smuggling](https://youtu.be/v_CUm93Pcik) — account takeover demo
- [HTTP request smuggling part 1 شرح ثغره عربي](https://youtu.be/EuSLY58k4G4)

### Videos (English)
- [The Most Overlooked Bug in Web Apps: HTTP Request Smuggling (Deep Dive)](https://youtu.be/6Zck1649AP0)
- [HTTP Request Smuggling - False Positives](https://youtu.be/7wq2e2nxa38) — ⚠️ important: not all findings are real
- [HTTP Request Smuggling Explained (with James Kettle)](https://youtu.be/QjPFjd8GJWY) — understand HTTP pipelining and keep-alive first
- [HTTP Desync Attack Explained With Paper](https://youtu.be/dnyL7EKbRRk)
- [Request smuggling - do more than running tools!](https://youtu.be/qKsdu6WAoqo) — bug bounty case study
- [albinowax - HTTP Desync Attacks: Smashing into the Cell Next Door - DEF CON 27](https://youtu.be/w-eJM2Pc0KI) — the original James Kettle research talk
- [HTTP Request Smuggling | Applied Review #24](https://youtu.be/SE3yWRzRH_U?list=PLCAv0O3KglnWDcjzJWRlYuD5SkgDtUlYl)

### PortSwigger (Primary Source)
- [HTTP Request Smuggling | Web Security Academy](https://portswigger.net/web-security/request-smuggling) — the definitive learning resource, has labs
- [Advanced Request Smuggling | Web Security Academy](https://portswigger.net/web-security/request-smuggling/advanced)
- [CL.0 Request Smuggling | Web Security Academy](https://portswigger.net/web-security/request-smuggling/browser/cl-0)
- [HTTP Desync Attacks: Request Smuggling Reborn | PortSwigger Research](https://portswigger.net/research/http-desync-attacks-request-smuggling-reborn) — James Kettle's original research paper
- [HTTP/2: The Sequel is Always Worse | PortSwigger Research](https://portswigger.net/research/http2) — HTTP/2 smuggling research
- [Browser-Powered Desync Attacks | PortSwigger Research](https://portswigger.net/research/browser-powered-desync-attacks)

### Written Resources — Fundamentals
- [HTTP Request Smuggling Explained: Beginner's Guide | Laburity](https://laburity.com/http-request-smuggling-explained-a-beginners-guide-on-identification-and-mitigation/)
- [HTTP Request Smuggling: Attacks and Prevention | brightsec.com](https://brightsec.com/blog/http-request-smuggling-hrs/)
- [Transfer-Encoding header reference | MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Transfer-Encoding) — chunked encoding examples
- [The Ultimate Bug Bounty Guide to HTTP Request Smuggling | YesWeHack](https://www.yeswehack.com/learn-bug-bounty/http-request-smuggling-guide-vulnerabilities)
- [Original HTTP Request Smuggling Paper (2005) | cgisecurity.com](https://www.cgisecurity.com/lib/HTTP-Request-Smuggling.pdf)

### Written Resources — CL.TE Deep Dives
- [HTTP Request Smuggling — CL.TE Vulnerability | Scott Murray](https://sc.scomurr.com/http-request-smuggling-cl-te-vulnerability/)
- [CL.TE Request Smuggling: Deep Technical Analysis | revbrightintl](https://revbrightintl.blogspot.com/2025/12/clte-request-smuggling-deep-technical.html)
- [HTTP Request Smuggling — Basic CL.TE vulnerability | InfoSec Write-ups](https://infosecwriteups.com/http-request-smuggling-basic-cl-te-vulnerability-a2975c664c53)
- [Connection-Locked CL.TE HTTP De-Sync Attacks | Sharp Security](https://sharpsec.run/connection-locked-cl-te-http-de-sync-attacks/)
- [CL.TE Request Smuggling | toxsec.com](https://www.toxsec.com/p/http-request-smuggling-clte)
- [HTTP Request Smuggling: Part-1 (Concepts) | Medium](https://medium.com/nerd-for-tech/http-request-smuggling-part-1-concepts-b89bfe17b210)
- [HTTP Request Smuggling: Part-2 (Identify & Exploit) | Medium](https://medium.com/nerd-for-tech/http-request-smuggling-part-2-tl-ce-exploit-ec1171a88459)

### Written Resources — Advanced / HTTP/2
- [Request smuggling and HTTP/2 downgrading: exploit walkthrough | Outpost24](https://outpost24.com/blog/request-smuggling-http-2-downgrading)
- [Funky chunks: abusing ambiguous chunk line terminators | w4ke.info](https://w4ke.info/2025/06/18/funky-chunks.html) — new desync trick via chunk extensions
- [Smuggling Requests with Chunked Extensions: A New HTTP Desync Trick | Imperva](https://www.imperva.com/blog/smuggling-requests-with-chunked-extensions-a-new-http-desync-trick/)
- [HTTP Pipelining — A Security Risk Without Real Performance Benefits | F5](https://community.f5.com/kb/technicalarticles/http-pipelining-a-security-risk-without-real-performance-benefits/286621)

### Write-ups / Walkthroughs
- [TryHackMe HTTP Request Smuggling Write-Up](https://readmedium.com/tryhackme-http-request-smuggling-write-up-60685cef555f)
