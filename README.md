.Net Core 8 & Blazor server

This is latest version : v3a

Blazor Cross-Site v3 (v3a) — English Guide

====================================================
1) Cross-Site Scripting (XSS)
====================================================
What it is
- XSS is when an attacker injects client-side code (usually JavaScript) that runs in a victim’s browser via inputs such as forms, comments, query strings, etc.
- Types: Stored (persisted in DB/content), Reflected (bounces back immediately), DOM-based (manipulates the DOM on the client).

Why it matters
- Can steal sessions/tokens and lead to account takeover.
- Can submit actions on behalf of the user, deface pages, or run phishing flows within a trusted origin.
- Damages trust and may break compliance requirements.

Key defenses (short list)
- Always encode output according to context (HTML/attribute/URL/JS).
- Avoid rendering raw HTML from user input (e.g., Html.Raw or MarkupString) without sanitizing first.
- If you must allow HTML, sanitize it with an allow‑list (this demo uses HtmlSanitizer).
- Consider adding Content-Security-Policy (CSP) and, for complex apps, Trusted Types to reduce XSS sinks.


====================================================
2) Cross-Site Request Forgery (CSRF)
====================================================
What it is
- CSRF tricks a user’s authenticated browser into sending a state‑changing request to your site (the browser includes cookies/credentials automatically).

Why it matters
- Critical actions may be performed without the user’s intent: password changes, fund transfers, deletes, approvals, etc.
- MFA doesn’t help if the session is already active.

Key defenses (short list)
- Use anti‑CSRF tokens (Synchronizer Token Pattern): generate a server‑side token bound to the user/session; require the client to send the token with each state‑changing request; validate server‑side.
- Double‑submit cookie, SameSite cookies, and Origin/Referer checks can help as secondary defenses.
- Prefer submitting tokens via custom headers in XHR/fetch requests (harder to forge cross‑site).


====================================================
Project Overview: Blazor Cross-Site v3 (v3a)
====================================================
Tech stack
- ASP.NET Core .NET 8, Blazor Server.
- HtmlSanitizer v9.0.886 (namespace Ganss.Xss) for XSS mitigation.
- Session‑backed CSRF token (stored server‑side in ISession). NOTE: A session cookie (name: .BlazorCrossSiteV3.Session) identifies the session; the CSRF token itself is not stored in the cookie.

Security design (what the demo demonstrates)
- XSS: Any user HTML is sanitized first (allow‑list in HtmlSanitizerService), then rendered as MarkupString.
- CSRF: Synchronizer Token Pattern using Session:
  - Client requests a token from the server (GET /api/session-csrf/token).
  - For POSTs, the client sends the token in header X‑CSRF‑TOKEN to /api/session-secure/echo.
  - Server validates header token against the token stored in Session.

Endpoints
- GET /api/session-csrf/token      -> creates/returns the session CSRF token.
- POST /api/session-secure/echo    -> validates X‑CSRF‑TOKEN; echoes input if valid.

Key files/folders
- /Pages/XssDemo.razor             -> XSS demo.
- /Pages/CsrfDemo.razor            -> CSRF demo (v3a: absolute URLs via NavigationManager + timeout + on‑page error display).
- /Services/HtmlSanitizerService.cs -> configures HtmlSanitizer allow‑list (e.g., h1, h2, class, data scheme).
- /Services/SessionCsrfService.cs   -> issues/validates CSRF token in Session.
- /Controllers/SessionCsrfController.cs  -> token endpoint.
- /Controllers/SessionSecureController.cs -> echo endpoint with validation.
- /Models/EchoRequest.cs            -> shared request record.

Program.cs (essentials)
- Adds DistributedMemoryCache + Session and UseSession().
- Registers HtmlSanitizerService and SessionCsrfService.
- Blazor Server + MVC controllers enabled.


====================================================
How to Run
====================================================
1) dotnet restore
2) dotnet run
Open https://localhost:<port>


====================================================
How to Test — XSS
====================================================
1) Navigate to /xss.
2) Enter safe HTML like: <h2>Hello</h2>  -> should render (allowed by allow‑list).
3) Enter dangerous HTML like: <script>alert('xss')</script> -> script should be removed; no alert.

Interpretation
- If disallowed tags/attributes are removed or encoded, XSS sanitization is working.
- Expand allow‑list only if necessary (the more you allow, the bigger the attack surface).


====================================================
How to Test — CSRF (v3a)
====================================================
In the page:
1) Go to /csrf.
2) Type a message and press “POST (with Session CSRF token)”.
   - The page calls GET /api/session-csrf/token (creates/returns the token in Session).
   - Then it sends POST /api/session-secure/echo with header X‑CSRF‑TOKEN: <token>.

Expected results
- 200 OK + body “Echo: <your message>”  -> valid token (request authenticated vs session).
- 401 Unauthorized + “Invalid CSRF token” -> missing/incorrect token, or session mismatch.

Curl example
# 1) Get token and store the session cookie
curl -k -c cookies.txt https://localhost:<port>/api/session-csrf/token
# 2) Use the same cookie + token to POST
curl -k -b cookies.txt -H "Content-Type: application/json" -H "X-CSRF-TOKEN: <token>" -d "{"text":"hello"}" https://localhost:<port>/api/session-secure/echo


====================================================
Troubleshooting
====================================================
- POST hangs or fails:
  - Check that GET /api/session-csrf/token returns 200; in v3a, CsrfDemo shows errors and uses absolute URLs + a timeout.
  - Make sure you are hitting the correct HTTPS port shown by your app.
- 401 Invalid CSRF token:
  - You might be using a token from a different session (e.g., fresh browser tab without requesting a new token).
  - Refresh /csrf to regenerate and retry.
- At scale:
  - Move Session to a distributed store (Redis, Azure Cache) or use sticky sessions so tokens persist across instances.


====================================================
Production Hardening (Checklist)
====================================================
- XSS:
  - Keep the HtmlSanitizer allow‑list as tight as possible.
  - Add a strict Content-Security-Policy (CSP) and consider nonces for scripts.
  - Avoid rendering unsanitized MarkupString from user inputs.
- CSRF:
  - Use Synchronizer Token Pattern (as in this demo); validate Origin/Referer for sensitive operations as an extra layer.
  - Rotate/expire tokens appropriately.
  - Ensure Session is durable across instances (distributed cache) if you scale out.
- Security headers:
  - HSTS, frame-ancestors (or X-Frame-Options), X-Content-Type-Options, Referrer-Policy.
- Logging/Monitoring:
  - Capture anomalies in request patterns and failed validations.


End of document.

