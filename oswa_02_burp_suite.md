# OSWA — Burp Suite

> Burp Suite is an integrated platform for web application security testing. It acts as a man-in-the-middle proxy between the browser and the target server, capturing and allowing modification of every request.

---

## Why Burp Suite Matters

Burp Suite is the primary tool for web assessments — used from the earliest enumeration phase through active exploitation.

```
As a web assessor, Burp Suite is where you'll spend 90%+ of your time.
It captures, modifies, replays, fuzzes, and decodes — all in one place.
```

---

## Browser Setup Options

|Option|How|Best For|
|---|---|---|
|Built-in browser (Chromium)|Proxy tab → Open Browser|Direct integration, no config needed|
|Firefox external|about:preferences#advanced → Network Settings → Manual proxy|When you need a full browser|

```
Proxy settings for Firefox:
- Host: 127.0.0.1
- Port: 8080
- Enable: Use this proxy for FTP and HTTPS
```

> [!tip] **FoxyProxy** — Browser extension that lets you toggle between proxy settings quickly without going back into Firefox preferences every time.

---

## Core Concept: Proxy

Burp Suite sits between your browser and the target server — every request passes through it.

```
Browser  ──►  Burp (port 8080)  ──►  Target Server
                    │
                    ▼
             Intercept / Log / Modify
```

### Intercept: On vs Off

|State|Behaviour|
|---|---|
|ON|Burp pauses each request — you must click Forward to send it|
|OFF|Requests pass through automatically — still logged in HTTP History|

> [!tip] **HTTP History stores everything** — even with intercept off. You can always go back and find a request.

### Match and Replace

Found in Proxy Settings (cog icon). Automatically modifies requests/responses based on rules — useful for mimicking different browsers or devices.

---

## Scope

> Scope defines which target(s) are relevant to your engagement. Burp will only log requests from in-scope targets.

### Why It Matters

Without scope, HTTP History fills up with JavaScript files, CSS, third-party requests — noise that makes it hard to find relevant data.

### Setting Up Scope

```
1. Target → Scope Settings → Add
2. Enter prefix: http://target.com
3. Click OK → "Stop sending OOS items to history?" → Yes
```

### In-Scope vs Out-of-Scope

||In-Scope|Out-of-Scope (OOS)|
|---|---|---|
|Logged in HTTP History|✅ Yes|❌ No (after config)|
|Appears in Site Map|✅ Yes|❌ No|

### Site Map

`Target → Site Map` — full list of every resource requested during the session. Useful for reviewing endpoints discovered over multiple days of testing.

---

# Tool: Repeater

> Repeater captures a request and lets you modify and resend it as many times as you want, viewing the response each time.

---

## Basic Workflow

```
1. Intercept a request in Proxy tab
2. Right-click → Send to Repeater
3. Click the Repeater tab
4. Modify the request in the Request panel
5. Click Send
6. Review the response in the Response panel
```

---

## Key Features

|Feature|What It Does|
|---|---|
|Request panel|Edit any part of the captured request|
|Response panel|Shows the full server response (headers + body)|
|Multiple tabs|Keep 20-40 requests open at once — one per test|
|Inspector|Decodes encoded strings inline (highlight text to activate)|

### Inspector / Decode Shortcut

```
Shortcut: C + B + u  →  decode selected encoded text inline
```

Paste encoded URI into request, highlight it, and Inspector shows the decoded value on the right panel.

```
Encoded:  %2Fsearch.php%3Fquery%3Dsample+search+query
Decoded:  /search.php?query=sample search query
```

---

# Tool: Comparer

> Comparer highlights the differences between two requests or responses — useful when two pages look the same but have different response sizes.

---

## When To Use It

- Two login responses with different sizes → which one succeeded?
- Two pages that look identical → find the subtle backend difference
- Before/after a modification → see what changed

---

## Workflow

```
1. In HTTP History, right-click request → Send to Comparer (Response)
2. Do the same for a second request
3. Click Comparer tab
4. Select both items → click Words or Bytes
```

### Comparison Types

|Button|What It Does|
|---|---|
|Words|Highlights differences word by word|
|Bytes|Highlights differences byte by byte|

### Difference Colour Coding

|Colour|Meaning|
|---|---|
|Modified|Content that changed|
|Deleted|Content removed|
|Added|Content that was inserted|

---

# Tool: Intruder

> Intruder automates sending a wordlist of payloads to a target position in a request — used for brute forcing, fuzzing, and enumeration.

> [!tip] **Community Edition throttles Intruder** — requests are slowed down. Still functional, just slower. Professional Edition removes this limit.

---

## Basic Workflow

```
1. Intercept a request → right-click → Send to Intruder
2. Click Intruder tab → Positions
3. Click Clear § to remove all auto-detected positions
4. Highlight the target value → click Add §
5. Click Payloads tab → Load wordlist
6. Click Start Attack
7. Sort results by Length — different length = interesting response
```

---

## Attack Types

|Type|Description|Use Case|
|---|---|---|
|**Sniper**|One wordlist, one position|Brute force single parameter (e.g. password)|
|**Battering Ram**|One wordlist, multiple positions — same value in all|Test same wordlist across username AND password simultaneously|
|**Pitchfork**|Multiple wordlists, one per position — paired row by row|Known username list + known password list, tested in pairs|
|**Cluster Bomb**|Multiple wordlists, all combinations — iterates everything|Full brute force — every username with every password|

```
# Cluster Bomb generates exponential requests:
# 100 usernames × 100 passwords = 10,000 requests
```

---

## § Character Reference

```
# Original captured request:
log=admin&pwd=test&wp-submit=Log+In

# After adding § around pwd value:
log=admin&pwd=§test§&wp-submit=Log+In
#                ↑ payload inserted here ↑
```

---

## Reading Attack Results

```
Sort by Length column:
- Most results: same size → failed attempts
- Outlier (different size) → successful match → that's your payload
```

---

# Tool: Decoder

> Decoder converts encoded strings to plaintext (and back). Supports URL, HTML, Base64, ASCII hex, Hex, Octal, Binary, and Gzip.

---

## When To Use It

Any time you find encoded data in a web application — cookies, parameters, hidden fields, tokens.

---

## Workflow

```
1. Click Decoder tab
2. Paste encoded string into the input box
3. Click "Decode as..." → select encoding type (e.g. Base64)
4. Decoded output appears in the box below
```

---

## Encoding Types Reference

|Type|Looks Like|Example|
|---|---|---|
|Base64|Alphanumeric + = padding|`V2VfUmVhbGx5...`|
|URL|% + hex digits|`%2F`, `%3A`, `+` for space|
|HTML|& entities|`&amp;`, `&lt;`, `&#x27;`|
|ASCII Hex|Hex pairs|`48 65 6c 6c 6f`|

```bash
# Base64 example:
Input:   V2VfUmVhbGx5X0RvX0xvdmVfQnVycF9TdWl0ZQo=
Output:  We_Really_Do_Love_Burp_Suite
```

> [!tip] **Also use Inspector inside Repeater** for quick inline decoding without switching tabs. Decoder is better for dedicated encoding/decoding sessions.

---

## Professional Edition Features (Overview)

|Feature|What It Adds|
|---|---|
|**Burp Scanner**|Automated crawl + active vulnerability scanning|
|**ActiveScan++**|Extension that boosts scanner coverage|
|**Collaborator**|Out-of-band server that detects blind interactions (SSRF, XXE, etc.)|
|**Intruder (no throttle)**|Full speed brute forcing — no rate limiting|
|**CSRF PoC Generator**|Auto-generates proof-of-concept for CSRF on any form|

> These are not required for OSWA course material — Community Edition covers everything needed.

---

## Burp Suite Tool Summary

|Tool|Primary Use|Key Action|
|---|---|---|
|**Proxy**|Intercept + log all traffic|Forward / Drop / Modify requests|
|**Repeater**|Replay + modify single requests|Right-click → Send to Repeater|
|**Comparer**|Diff two responses|Right-click → Send to Comparer|
|**Intruder**|Automated payload fuzzing|Right-click → Send to Intruder|
|**Decoder**|Encode / decode strings|Paste → Decode as...|
|**Site Map**|Overview of all visited endpoints|Target → Site Map|
|**Scope**|Filter history to target only|Target → Scope Settings → Add|

---

## Key Takeaways

```
1.  Built-in browser is the easiest setup — direct integration, no config
2.  Set Scope early — keeps HTTP History clean and relevant
3.  Intercept OFF still logs everything — HTTP History is always there
4.  Repeater is your primary testing tool — live in it
5.  Different response size = interesting — always sort Intruder results by Length
6.  403 → a § on that path + Intruder to bypass
7.  Find encoded data? → Decoder first, then understand what it contains
8.  Comparer spots differences humans miss — use it on similar responses
9.  Community Edition throttles Intruder — still works, just slower
10. Site Map = your engagement memory — check it before each session
```