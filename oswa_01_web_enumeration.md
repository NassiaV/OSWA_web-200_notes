# OSWA — Web Application Enumeration

> Web application enumeration is the systematic process of gathering information about a target web application before exploitation.

---

## Why Enumeration Matters

Enumeration is the foundation of every successful web application assessment.

```
Good enumeration answers:
- What pages and directories exist?
- What technologies are being used?
- What user accounts exist?
- What sensitive information is exposed?
```

---

## Client-Side vs Server-Side Attacks

|Type|Target|Examples|Result|
|---|---|---|---|
|Client-side|Users/browsers|XSS, CSRF|Session hijacking|
|Server-side|App/server|SQLi, RCE|Data theft, code execution|

---

## Enumeration Methodology

```
Phase 1 — Passive Reconnaissance
└── robots.txt, sitemap.xml, source code
└── Technology fingerprinting

Phase 2 — Active Enumeration
└── Directory brute forcing
└── Web crawling
└── Username enumeration

Phase 3 — Information Analysis
└── Connect findings together
└── Document everything

Phase 4 — Attack Preparation
└── Custom wordlists
└── Username variations
```

> [!tip] **Connect the dots** About page reveals names → names become usernames → usernames enable credential attacks. Nothing found during enumeration is irrelevant.

---

## HTTP Response Codes

|Code|Meaning|Action|
|---|---|---|
|200|OK — exists|Investigate|
|301/302|Redirect|Follow it|
|403|Forbidden — EXISTS|Note it|
|401|Unauthorised|Try credentials|
|404|Not Found|Move on|
|405|Method Not Allowed|Try different method|
|500|Server Error|May indicate vuln|

> [!tip] **403 means it EXISTS** — just blocked. Always note and investigate bypass. **405 means the endpoint exists** — try POST instead of GET.

> [!warning] **308 redirect means trailing slash** If ffuf finds `/auth` with status 308, the real path is `/auth/`. But POST requests to `/auth/` may still return 405 — the actual form endpoint is often deeper, e.g. `/auth/login`. Always check the form's `action` attribute to find the real POST target.

---

## Wordlists

```bash
# View available wordlists
ls -alh /usr/share/wordlists

# Key wordlists
/usr/share/wordlists/dirb/common.txt              # Quick scan
/usr/share/wordlists/dirb/big.txt                 # Deeper scan
/usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt  # Thorough
/usr/share/wordlists/rockyou.txt                  # Passwords

# Uncompress rockyou
sudo gunzip /usr/share/wordlists/rockyou.txt.gz

# A wordlist is literally just this: 
cat /usr/share/wordlists/dirb/common.txt | head -5 
.bash_history 
.bash_logout 
.bash_profile 
.bashrc 
.cvs 

# And rockyou.txt is just: 
head -5 /usr/share/wordlists/rockyou.txt 
123456 
12345 
123456789 
password 
iloveyou
```

---

# Tool: echo

> echo outputs text to the terminal or pipes it to other tools.

```bash
# Pipe URL to tool
echo "http://192.168.1.10" | hakrawler -u

# Create file
echo "admin" > users.txt # Creates users.txt with "admin" inside. If users.txt already existed — IT WIPES IT and starts fresh
echo "root" >> users.txt # Adds "root" to the END of users.txt. Existing content is preserved

echo "admin" > users.txt # users.txt contains: admin 
echo "root" >> users.txt # users.txt contains: admin + root 
echo "user" >> users.txt # users.txt contains: admin + root + user 
# But if you accidentally use > instead of >>: 
echo "admin" > users.txt # users.txt contains: admin 
echo "root" > users.txt # users.txt NOW ONLY contains: root # admin is GONE!

# Heredoc — best method for multiple entries instead of the above examples
cat > users.txt << 'EOF'
foo
admin
root
john.smith
j.smith
EOF
```

### Creating Wordlist Files

```bash
# Method 1 — Heredoc (recommended)
cat > users.txt << 'EOF'
foo
tomjones
tom.jones
t.jones
EOF

# Method 2 — Append with echo
echo "foo" > users.txt
echo "admin" >> users.txt

# Verify
cat users.txt
wc -l users.txt
cat -A users.txt   # check for hidden chars — lines end with $

# Fix Windows line endings if needed
sed -i 's/\r//' users.txt
```

---

# Tool: curl

> curl transfers data to or from a server — essential for manual testing and fingerprinting.

```bash
# Basic page fetch
curl http://192.168.1.10

# Headers only — fingerprint technology
curl -I http://192.168.1.10
# Server: Apache/2.4.49       → server version
# X-Powered-By: PHP/7.4.3    → PHP version
# X-Powered-By: ASP.NET      → .NET application

# Verbose output
curl -v http://192.168.1.10

# Always check these early
curl -s http://192.168.1.10/robots.txt
curl -s http://192.168.1.10/sitemap.xml

# Strip HTML — read as plain text
curl -s http://192.168.1.10/about/ | sed 's/<[^>]*>//g' | grep -v "^$"

# POST request
curl -X POST http://192.168.1.10/login \
  -d "username=admin&password=test" \
  -H "Content-Type: application/x-www-form-urlencoded"

# Discover the real POST endpoint from a login page's form
curl http://192.168.1.10/auth/ | grep -i "form\|input\|action"
# Look for: <form action="/auth/login" method="POST">
# This tells you the exact URL to POST to and the field names

# Get response size (baseline for ffuf)
curl -s -X POST http://192.168.1.10/login \
  -d "username=foo&password=bar" | wc -c

# Test HTTP methods
for method in GET POST PUT DELETE OPTIONS; do
  curl -s -X $method http://192.168.1.10/login \
    -w "$method: %{http_code}\n" -o /dev/null
done
```

---

# Tool: dirb

> dirb is a content scanner that brute forces directories using a wordlist.

```bash
# Basic scan (uses default wordlist)
dirb http://192.168.1.10

# Custom wordlist
dirb http://192.168.1.10 /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt

# Scan for extensions
dirb http://192.168.1.10 -X .php
dirb http://192.168.1.10 -X .php,.html,.txt,.bak

# HTTPS
dirb https://192.168.1.10

# Save output
dirb http://192.168.1.10 -o results.txt

# Add delay (avoid flooding/lockout)
dirb http://192.168.1.10 -z 100

# Silent — results only
dirb http://192.168.1.10 -S
```

### Flags Reference

|Flag|What it does|
|---|---|
|-X [ext]|Append extension to each word|
|-x [file]|Extensions from file|
|-o [file]|Save output|
|-N [code]|Ignore HTTP code|
|-z [ms]|Delay between requests|
|-r|Non-recursive|
|-S|Silent mode|

### Technology → Extension Mapping

|Technology|Detection|Extensions|
|---|---|---|
|PHP|X-Powered-By: PHP|.php,.php3,.phtml|
|ASP.NET|X-Powered-By: ASP.NET|.asp,.aspx,.ashx|
|Java|Server/error pages|.jsp,.jspx,.do|
|General|Always include|.txt,.html,.bak,.zip,.conf,.log|

```bash
# Fingerprint first
curl -I http://192.168.1.10 | grep -i "x-powered-by\|server"
# Then scan with matching extensions
dirb http://192.168.1.10 -X .php,.txt,.bak
```

---

# Tool: gobuster

> gobuster is a faster directory brute forcer written in Go.

```bash
# Directory scan
gobuster dir -u http://192.168.1.10 \
  -w /usr/share/wordlists/dirb/common.txt \
  -t 50 -q

# With extensions
gobuster dir -u http://192.168.1.10 \
  -w /usr/share/wordlists/dirb/common.txt \
  -x php,html,txt,bak,zip -o results.txt

# DNS subdomain enumeration
gobuster dns -d target.com \
  -w /usr/share/wordlists/dirb/common.txt -t 50

# Virtual host discovery
gobuster vhost -u http://192.168.1.10 \
  -w /usr/share/wordlists/dirb/common.txt --append-domain

# HTTPS (ignore SSL errors)
gobuster dir -u https://192.168.1.10 \
  -w /usr/share/wordlists/dirb/common.txt -k
```

### dirb vs gobuster

|Feature|dirb|gobuster|
|---|---|---|
|Speed|Slower|Faster (Go)|
|Default wordlist|Yes|Must specify|
|DNS enumeration|No|Yes|
|VHost discovery|No|Yes|

---

# Core Concept: Fuzzing

> Fuzzing sends large volumes of unexpected or semi-random data to a target application to find behaviour the developers did not intend.

---

## Why Fuzzing Matters

Most vulnerabilities exist at the boundary between expected and unexpected input.

```
Normal:  username=john    → "Invalid credentials"
Fuzzed:  username=john'   → "SQL syntax error" → SQLi found!
Fuzzed:  username=admin   → Redirects to dashboard → valid user!
```

---

## Types of Fuzzing

### 1. Directory and File Fuzzing

```bash
ffuf -u http://target.com/FUZZ \
  -w /usr/share/wordlists/dirb/common.txt -fc 404
# FUZZ = placeholder replaced with each wordlist entry
# -w = wordlist to use
# -fc 404 = filter out 404 responses (not found)  show only real findings
```

### 2. Username Enumeration Fuzzing

```bash
ffuf -w users.txt \
  -u http://target.com/login \
  -X POST \
  -d 'username=FUZZ&password=wrongpassword' \
  -H 'Content-Type: application/x-www-form-urlencoded'
# -X POST = send as POST request (login forms use POST)
# -d = Post body - FUZZ replaces the username value
# -H = Content-Type header required for form submissions
# no filter here - we want to see ALL responses first
# Then compare sizes against control element (foo)
# Different size from control = valid username
```

### 3. Password Fuzzing

```bash
ffuf -w /usr/share/wordlists/rockyou.txt \
  -u http://target.com/login \
  -X POST \
  -d 'username=admin&password=FUZZ' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -fs [failed_size]
# Username is now fixed - we confirmed it in step 2
# FUZZ now replaces the password value
# -fs [failed_size] = filter out failed login responses
# Get [failed_size] first with:
# curl -s -X POST http://target.com/login -d "username=admin&password=wrong" | wc -c
# Only responses with DIFFERENT size shown = successful login = password found 
```

### 4. Hidden Parameter Fuzzing

```bash
ffuf -u "http://target.com/profile?FUZZ=1" \
  -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt \
  -fs [default_size]
# FUZZ replaces the parameter NAME not the value 
# Tests thousands of parameter names against the endpoint
# Get [default_size] first with:
# curl -s "http://target.com/profile?test=1" | wc -c
# Different response = parameter exists and changes application behaviour
# Example findings: ?debug=1, ?admin=1, ?id=1
```

---

## The Control Element

> A control element is a known invalid value used as baseline — all responses are compared against it.

```bash
cat > users.txt << 'EOF'
foo          # CONTROL — known invalid
tomjones
tom.jones
t.jones
EOF

# After running ffuf:
# foo       [Size: 2093]  ← baseline
# tom.jones [Size: 2089]  ← DIFFERENT → valid username!
# tomjones  [Size: 2093]  ← matches baseline → invalid
```

---

## Fuzzing Risks

|Risk|Cause|Mitigation|
|---|---|---|
|Denial of Service|Too many requests|Limit threads with -t, add delay -p|
|Account Lockout|Failed logins|Use -t 1, check policy first|
|IDS Detection|Unusual patterns|Slow down with -p delay|

```bash
# Safe fuzzing
ffuf -u http://target.com/FUZZ -w wordlist.txt -t 10 -p 0.1
```

---

# Tool: ffuf

> ffuf (Fuzz Faster U Fool) is a fast, flexible web fuzzer for directories, parameters, and login brute forcing.

---

## Basic Syntax

```bash
ffuf -u [URL with FUZZ] -w [wordlist] [options]
```

---

## Common Uses

```bash
# Directory discovery
ffuf -u http://192.168.1.10/FUZZ \
  -w /usr/share/wordlists/dirb/common.txt -fc 404

# Username enumeration
ffuf -w users.txt \
  -u http://192.168.1.10/auth/login \
  -X POST \
  -d 'username=FUZZ&password=bar' \
  -H 'Content-Type: application/x-www-form-urlencoded'

# Password brute force
ffuf -w /usr/share/wordlists/rockyou.txt \
  -u http://192.168.1.10/auth/login \
  -X POST \
  -d 'username=r.jones&password=FUZZ' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -fs [failed_size]

# Subdomain discovery
ffuf -u http://FUZZ.target.com \
  -H "Host: FUZZ.target.com" \
  -w subdomains.txt -fs [default_size]

# With cookie
ffuf -u http://192.168.1.10/FUZZ \
  -w wordlist.txt \
  -H "Cookie: session=abc123" -fc 404

# Through Burp proxy
ffuf -u http://192.168.1.10/FUZZ \
  -w wordlist.txt -x http://127.0.0.1:8080
```

---

## Filtering Reference

|Filter|Flag|Example|
|---|---|---|
|Filter status code|-fc|-fc 404,403|
|Match status code|-mc|-mc 200,301|
|Filter response size|-fs|-fs 2093|
|Match response size|-ms|-ms 500|
|Filter word count|-fw|-fw 678|
|Filter line count|-fl|-fl 66|
|Filter by regex|-fr|-fr "Not Found"|
|Match by regex|-mr|-mr "Welcome"|

### -fc vs -fs — Which to use?

|Situation|Use|Why|
|---|---|---|
|Directory discovery|`-fc 404`|Status codes are reliable for directories|
|Login brute force|`-fs [size]`|All failed logins return 200 — status code is useless|

```bash
# WRONG for login brute force — filters nothing useful
ffuf ... -fc 200   # all failed logins ARE 200, so this removes everything

# RIGHT for login brute force — get baseline size first
curl -s -X POST http://target.com/auth/login \
  -d "username=foo&password=wrong" \
  -H "Content-Type: application/x-www-form-urlencoded" | wc -c
# → 2089
ffuf ... -fs 2089  # hides all 2089-byte failures, shows only different sizes
```

> [!tip] **If ffuf returns 0 results with -fs** — you have the filter backwards. The baseline you measured IS the failed login size. If everything gets filtered, it means all responses match — which means the password wasn't found in that wordlist, not that the filter is wrong.

---

## Reading ffuf Output

```
foo       [Status: 200, Size: 2093, Words: 678, Lines: 66]  ← baseline
r.jones   [Status: 200, Size: 2089, Words: 676, Lines: 66]  ← DIFFERENT!
tomjones  [Status: 200, Size: 2093, Words: 678, Lines: 66]  ← matches baseline
```

Even a 4-byte size difference indicates different application behaviour — r.jones is a valid username!

---

# Core Concept: Information Disclosure

> Information disclosure occurs when a web application unintentionally reveals sensitive information through error messages, headers, or behaviour.

---

## What Applications Leak

### Error Messages

```
"No matching user found"     → username doesn't exist → enumeration possible
"Incorrect password"         → username IS valid
"ORA-01756"                  → Oracle database
"SQL syntax error"           → MySQL database
"Fatal error: Uncaught"      → PHP application
```

### HTTP Headers

```bash
curl -I http://target.com
# Server: Apache/2.4.49       → exact version
# X-Powered-By: PHP/7.4.3    → PHP version
# Set-Cookie: PHPSESSID=...  → PHP application
# Set-Cookie: JSESSIONID=... → Java application
```

### robots.txt

```bash
curl -s http://target.com/robots.txt
# Admins list sensitive paths to hide from search engines
# This tells attackers exactly what to investigate!
# May contain: /admin, /backup, /config, even flags!
```

> [!warning] **robots.txt can contain decoys** Not everything in robots.txt is real or useful. Paths listed may return 404, and values that look like flags may be fake. Always verify — a path that returns 404 means it doesn't exist, regardless of what robots.txt says.

### Page Content

```
About pages      → staff names → username candidates
Contact pages    → email formats → username format clues
HTML comments    → developer notes, credentials, API keys
Error pages      → technology stack details
```

---

## Username Enumeration — Step by Step

```
Step 1 — Find login page
       ↓
Step 2 — Test with known invalid username (foo/bar)
         Note error message: "No matching user found"
       ↓
Step 3 — Find username candidates
         About pages, team pages, blog authors
       ↓
Step 4 — Build username variations
         "John Smith" → john.smith, j.smith, johnsmith, john_smith
       ↓
Step 5 — Add control element (foo) to wordlist
       ↓
Step 6 — Run ffuf
         Different response size = valid username confirmed
```

### Real Example from OSWA Sandbox

```bash
# About page revealed: Roxanne Jones and Ernesto Freeman

# Build username wordlist
cat > users.txt << 'EOF'
foo
roxannejones
roxanne.jones
r.jones
r_jones
roxanne_jones
ernestofreeman
ernesto.freeman
e.freeman
e_freeman
ernesto_freeman
EOF

# Run enumeration
ffuf -w users.txt \
  -u http://192.168.64.108/auth/login \
  -X POST \
  -d 'username=FUZZ&password=bar' \
  -H 'Content-Type: application/x-www-form-urlencoded'

# Results:
# foo       [Size: 2093]  ← baseline
# r.jones   [Size: 2089]  ← VALID USERNAME!
# e.freeman [Size: 2089]  ← VALID USERNAME!
```

---

# Tool: hakrawler

> hakrawler is a web crawler that discovers URLs by following links — complementing brute forcing tools.

---

## Installation

```bash
# Method 1 — Go install
go install github.com/hakluke/hakrawler@latest
echo 'export PATH=$PATH:~/go/bin' >> ~/.bashrc
source ~/.bashrc

# Method 2 — Build from source
git clone https://github.com/hakluke/hakrawler
cd hakrawler
go build .
sudo mv hakrawler /usr/local/bin/

# Verify
hakrawler -h
```

---

## Basic Syntax

```bash
echo "[URL]" | hakrawler [options]
```

---

## Common Uses

```bash
# Basic crawl
echo "http://192.168.1.10" | hakrawler

# Unique URLs only
echo "http://192.168.1.10" | hakrawler -u

# Set crawl depth
echo "http://192.168.1.10" | hakrawler -depth 3

# Include subdomains
echo "http://192.168.1.10" | hakrawler -subs

# Set threads
echo "http://192.168.1.10" | hakrawler -t 8

# Set timeout
echo "http://192.168.1.10" | hakrawler -timeout 10
```

---

## hakrawler vs dirb

|Feature|dirb|hakrawler|
|---|---|---|
|Method|Brute forces wordlist|Follows existing links|
|Finds|Hidden unlinked paths|Linked content not in wordlists|
|Requires|Wordlist|Just the URL|
|Misses|Linked content|Hidden unlinked paths|

> [!tip] **Always use both** dirb finds hidden unlinked paths. hakrawler finds linked content not in wordlists. Together they give complete coverage — as demonstrated in the OSWA sandbox where dirb found 6 pages but hakrawler found 15 including /manual/about!

---

# Tool: CeWL

> CeWL (Custom Word List generator) crawls a URL and creates a wordlist from words found on the pages.

---

## Basic Syntax

```bash
cewl [options] [URL]
```

---

## Common Uses

```bash
# Basic wordlist generation
cewl http://192.168.1.10 --write output.txt

# With minimum length and lowercase
cewl --write output.txt --lowercase -m 5 http://192.168.1.10/manual

# Check results
head -10 output.txt    # first 10 words
tail output.txt        # last 10 words by DEFAULT
wc -l output.txt       # total count
cat -n output.txt | head -20   # with line numbers

# Use for password brute force
ffuf -w output.txt \
  -u http://192.168.1.10/auth/login \
  -X POST \
  -d 'username=r.jones&password=FUZZ' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -fs [failed_size]
```

> [!warning] **CeWL accepts only ONE URL at a time** — passing multiple URLs causes a "Missing URL argument" error. To crawl multiple pages, run CeWL separately on each and combine the results:

```bash
# WRONG — causes error
cewl http://target.com http://target.com/about -w output.txt

# RIGHT — run separately and merge
cewl http://target.com -d 2 -m 3 -w w1.txt
cewl http://target.com/about -d 2 -m 3 -w w2.txt
cewl http://target.com/blog -d 2 -m 3 -w w3.txt
cat w1.txt w2.txt w3.txt | sort -u > wordlist_full.txt
wc -l wordlist_full.txt
```

---

## CeWL Flags Reference

|Flag|What it does|
|---|---|
|--write [file]|Save output to file|
|--lowercase|Convert to lowercase|
|-m [n]|Minimum word length|
|-d [n]|Crawl depth|
|--with-numbers|Include words with numbers|
|--email|Extract email addresses|

---

## Other Custom Wordlist Techniques

```bash
# Binary wordlist from system (for command injection)
ls /usr/bin | grep -v "/" > binaries.txt
tail binaries.txt

# Combine wordlists
cat list1.txt list2.txt | sort -u > combined.txt

# Extract matching lines
grep "admin" /usr/share/wordlists/dirb/common.txt > admin_words.txt

# First N lines of rockyou
head -1000 /usr/share/wordlists/rockyou.txt > top1000.txt
```

---

# Types of Web Application Attacks

---

## Authentication Bypass

> Gaining new permissions — bypassing login, escalating privileges, or hijacking sessions.

```
Methods:
- SQL Injection          → bypass login without credentials
- Brute forcing          → guess credentials with wordlists
- Session hijacking      → steal session via XSS
- CORS abuse             → perform actions as another user
- Cookie manipulation    → modify session tokens to elevate privileges
```

---

## Data Exfiltration

> Accessing restricted or sensitive data beyond the current user's permissions.

```
Types of sensitive data:
- Payment information (credit cards, bank accounts)
- PII (names, addresses, national IDs)
- Authentication data (password hashes, API keys)
- Business data (proprietary information)

Real-world impact:
SolarWinds (2019) → SEC fraud charges + $26M class-action settlement
```

---

## Remote Code Execution (RCE)

> The most severe — allows executing arbitrary code on the target system.

```
RCE enables:
- Complete system control
- Data exfiltration
- Malware installation
- File modification/deletion
- Launching further attacks
```

### Shell Types

|Type|How it works|When to use|
|---|---|---|
|Bind shell|Target listens, attacker connects|Direct network access|
|Reverse shell|Target connects to attacker|NAT/firewalled targets|
|Web shell|Commands via browser|HTTP-only access|

---

# Tool: netcat

> netcat reads and writes data across network connections — used for connecting to bind shells and catching reverse shells.

---

## Basic Syntax

```bash
netcat [host] [port]      # Connect to host
netcat -vlp [port]        # Listen for connection
```

---

## Connecting to a Bind Shell

```bash
# Target has bind shell on port 9999
netcat 192.168.1.10 9999

# Verify access
whoami
hostname
```

---

## Listening for a Reverse Shell

```bash
# Step 1 — Start listener
netcat -vlp 9090
# -v = verbose
# -l = listen mode
# -p = specify port

# Step 2 — Wait for target to connect back
# listening on [any] 9090 ...
# connect to [attacker] from [target] 36140
# #   ← shell prompt

# Step 3 — Verify
whoami
hostname
```

---

## Bind vs Reverse Shell

|Feature|Bind Shell|Reverse Shell|
|---|---|---|
|Connection|Attacker → Target|Target → Attacker|
|Firewall|May block inbound|Usually allows outbound|
|Security|Anyone can connect|Only your listener|
|Preferred?|Less common|More common|

> [!tip] **Reverse shells are preferred** Most servers block inbound connections but allow outbound. A bind shell is also a security risk — anyone finding the port can connect.

---

## Web Shells

> Web shells provide command execution through a browser-accessible script.

```bash
# View available web shells in Kali
ls /usr/share/webshells/
# asp/     → Windows IIS (Classic ASP)
# aspx/    → Windows IIS (.NET)
# jsp/     → Java web servers
# php/     → PHP applications
# perl/    → Perl CGI

# Choose shell matching target technology
curl -I http://target.com | grep "x-powered-by\|server"
# PHP  → simple-backdoor.php
# ASP  → cmdasp.asp
# JSP  → cmdjsp.jsp
```

---

# Tool Reference Appendix

## Proxy Tools

|Tool|Type|Key Use|
|---|---|---|
|Burp Suite|Commercial/Free|Web app testing — intercept, modify, replay|
|ZAP|Free/Open Source|Automated and manual vulnerability scanning|
|Fiddler|Commercial/Free|HTTP traffic capture — no security tools|

## Content Discovery Tools

|Tool|Key Feature|
|---|---|
|DIRB|Simple wordlist-based scanner|
|DirBuster|Multi-threaded, GUI or headless|
|Gobuster|Fast Go-based — supports DNS and S3|
|hakrawler|Web crawler — follows links|
|ffuf|Flexible fuzzer — directory, parameter, login|
|Wfuzz|Web brute forcing — similar to ffuf|

## Vulnerability Scanners

|Tool|Type|
|---|---|
|Nikto|Free — web server vulnerability scanning|
|Nessus|Commercial — comprehensive scanning|
|Qualys|Commercial — cloud-based|
|OpenVAS|Free/Open Source|

## Specialty Tools

|Tool|Key Use|
|---|---|
|sqlmap|SQL injection discovery and exploitation|
|Metasploit|Exploitation framework|
|CeWL|Custom wordlist generation|
|netcat|Bind/reverse shells, port scanning|

---

# Complete Enumeration Workflow

```bash
# Phase 1 — Initial Reconnaissance
curl -s http://192.168.1.10/robots.txt
curl -s http://192.168.1.10/sitemap.xml
curl -I http://192.168.1.10 | grep -iE "server|x-powered-by|set-cookie"

# Phase 2 — Directory and File Discovery
dirb http://192.168.1.10
gobuster dir -u http://192.168.1.10 \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x php,html,txt,bak -t 50 -o results.txt

# Phase 3 — Web Crawling
echo "http://192.168.1.10" | hakrawler -u

# Phase 4 — Information Analysis
curl -s http://192.168.1.10/about/ | sed 's/<[^>]*>//g' | grep -v "^$"
# Note names, emails, technology details

# Phase 5 — Custom Wordlists
cewl --write custom_words.txt --lowercase -m 5 http://192.168.1.10

# Build username variations from discovered names
cat > users.txt << 'EOF'
foo
john.smith
j.smith
johnsmith
john_smith
EOF

# Phase 6 — Username Enumeration
curl -s -X POST http://192.168.1.10/auth/login \
  -d "username=foo&password=bar" | wc -c   # get baseline size

ffuf -w users.txt \
  -u http://192.168.1.10/auth/login \
  -X POST \
  -d 'username=FUZZ&password=bar' \
  -H 'Content-Type: application/x-www-form-urlencoded'

# Phase 7 — Password Attack
ffuf -w custom_words.txt \
  -u http://192.168.1.10/auth/login \
  -X POST \
  -d 'username=r.jones&password=FUZZ' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -fs [failed_size]
```

---

# Password Brute Force Workflow

> This workflow ties together CeWL wordlist generation, baseline measurement, and ffuf password brute forcing into a single repeatable process.

---

## Step 0 — Find the Real Login Endpoint

> Before anything else, confirm the exact URL the login form POSTs to. Do not assume `/login` or `/auth` — these often redirect or return 404 for POST.

```bash
# 1 — Check what the visible login page URL is in the browser
# e.g. http://192.168.1.10/auth/

# 2 — Fetch the page and inspect the form action
curl http://192.168.1.10/auth/ | grep -i "form\|input\|action"
# Look for: <form action="/auth/login" method="POST">
# This is your actual POST endpoint and the correct field names

# 3 — Verify it accepts POST (405 = wrong method, 404 = wrong path)
curl -s -o /dev/null -w "%{http_code}" \
  -X POST http://192.168.1.10/auth/login \
  -d "username=foo&password=bar" \
  -H "Content-Type: application/x-www-form-urlencoded"
# Should return 200, not 404 or 405
```

> [!warning] **308 redirect trap** If ffuf finds `/auth` with status 308, it redirects to `/auth/`. But `/auth/` may only accept GET — the POST endpoint is usually `/auth/login`. Always inspect the form action, not just the page URL.

---

## Step 1 — Generate Custom Wordlist with CeWL

```bash
# Crawl the target application and extract words
cewl --write custom_words.txt --lowercase -m 5 http://192.168.64.108/manual
# --write     = save to file
# --lowercase = normalise all words to lowercase
# -m 5        = minimum word length of 5 characters

# Verify the wordlist was created
wc -l custom_words.txt       # count words
head -10 custom_words.txt    # preview first 10
```

---

## Step 2 — Confirm Valid Username

```bash
# You should already have a valid username from enumeration
# If not — run username enumeration first (Phase 6 above)
# In this example: r.jones and e.freeman are valid
```

---

## Step 3 — Get the Failed Login Baseline Size

```bash
# Send a login with known wrong password
# Note the response size — this is your filter value
curl -s -X POST http://192.168.64.108/auth/login \
  -d "username=r.jones&password=wrongpassword" \
  -H "Content-Type: application/x-www-form-urlencoded" | wc -c

# Example output: 2093
# This means every failed login returns 2093 bytes
# A successful login will return a DIFFERENT size
```

---

## Step 4 — Brute Force Password with ffuf

```bash
# Replace [baseline_size] with the number from Step 3
ffuf -w custom_words.txt \
  -u http://192.168.64.108/auth/login \
  -X POST \
  -d 'username=r.jones&password=FUZZ' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -fs [baseline_size]

# -fs filters OUT responses matching baseline size
# Only responses with DIFFERENT size are shown
# Different size = successful login = password found!
```

---

## Step 5 — Confirm the Password

```bash
# Verify manually with curl
curl -s -X POST http://192.168.64.108/auth/login \
  -d "username=r.jones&password=[found_password]" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -v 2>&1 | grep -i "location\|welcome\|dashboard\|set-cookie"
# A redirect (302) or different content confirms successful login
```

---

## Full Workflow — One Shot

```bash
# 0 — Find the real POST endpoint
curl http://192.168.64.108/auth/ | grep -i "form\|action"
# → <form action="/auth/login" method="POST">

# 1 — Generate wordlist (crawl content-rich pages, not just root)
cewl http://192.168.64.108 -d 2 -m 3 -w w1.txt
cewl http://192.168.64.108/manual -d 2 -m 3 -w w2.txt
cat w1.txt w2.txt | sort -u > custom_words.txt

# 2 — Get baseline (failed login size)
BASELINE=$(curl -s -X POST http://192.168.64.108/auth/login \
  -d "username=r.jones&password=wrongpassword" \
  -H "Content-Type: application/x-www-form-urlencoded" | wc -c)
echo "Baseline size: $BASELINE"

# 3 — Brute force
ffuf -w custom_words.txt \
  -u http://192.168.64.108/auth/login \
  -X POST \
  -d 'username=r.jones&password=FUZZ' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -fs $BASELINE
```

> [!tip] **Why CeWL wordlists work well here** Generic wordlists like rockyou.txt contain millions of random passwords. CeWL generates words specific to the target application — if the password is related to the company's content, services, or terminology, CeWL will find it much faster than a generic wordlist.

> [!tip] **Try both valid usernames** If r.jones doesn't yield results, try e.freeman with the same wordlist — different users often have different passwords.

---

## Key Takeaways

```
1.  Enumerate before exploiting — always
2.  robots.txt reveals what administrators want hidden — but verify paths exist and beware fake flags
3.  dirb finds hidden paths, hakrawler finds linked content — use both
4.  Information disclosure on login pages enables username enumeration
5.  About pages reveal names → username candidates
6.  Control elements anchor your ffuf analysis
7.  Custom wordlists outperform generic ones for specific targets
8.  Response size differences reveal valid usernames and passwords
9.  Connect the dots — every finding feeds the next attack
10. Consider risks — DOS, lockouts, detection
11. 308 redirect means trailing slash — but POST endpoints are usually deeper (e.g. /auth/login)
12. Always check the form action attribute — the POST URL is often different from the page URL
13. -fc for directories, -fs for login brute force — status codes don't distinguish failed logins
14. CeWL only accepts one URL — crawl multiple pages separately and merge with cat | sort -u
``````