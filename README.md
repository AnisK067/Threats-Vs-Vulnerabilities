# Lab 4: Active Activity — Web Application Vulnerability Assessment

**Name:** Nur Anis Najwa Binti Kamarudin
**Class:** B05

---

## 1. Target Setup (Victim)

| Detail | Value |
|---|---|
| Application | OWASP Juice Shop |
| Docker Image | `bkimminich/juice-shop` |
| Hosting Method | Docker Desktop (local container) |
| Access URL | `http://localhost:3000` |
| Port Mapping | `3000:3000` |

**Setup commands used:**
```bash
docker pull bkimminich/juice-shop
docker run -d -p 3000:3000 --name juice-shop bkimminich/juice-shop
docker ps
```

Juice Shop is an intentionally vulnerable modern web application (Node.js/Express backend, Angular frontend) developed by OWASP for security training. It was chosen as the target because it provides a safe, legal, and reproducible environment with a built-in scoring system that validates successfully exploited vulnerabilities.

---

## 2. Attacker Setup (Own Hosts)

| Detail | Value |
|---|---|
| OS | Windows |
| Tooling | Google Chrome (DevTools: Console & Network tabs) |
| Access Method | Browser-based manual testing (no external attacker VM — attacker and victim container run on the same host machine) |

Testing was performed manually through the browser's built-in developer tools rather than dedicated pentest VMs, since Juice Shop's vulnerabilities are primarily client/application-layer and reachable through standard browser interaction, request inspection (Network tab), and JavaScript console execution (Console tab, `fetch()` + `localStorage` token manipulation).

---

## 3. Findings

### Finding 1 — Hidden Admin Interface Discovery
- **OWASP Category:** Security Misconfiguration
- **Challenge:** Score Board
- **Steps:**
  1. Navigated directly to `http://localhost:3000/#/score-board`
- **Evidence:** Application confirmed: *"You successfully solved a challenge: Score Board (Find the carefully hidden 'Score Board' page.)"*
- **Impact:** An internal administrative/debug page (listing every known vulnerability in the app) was reachable without authentication or any access control, simply by guessing/knowing the route. In a production system, an equivalent exposed debug/admin route could reveal sensitive internal information to any unauthenticated user.

### Finding 2 — Improper Error Handling
- **OWASP Category:** Improper Error Handling / Information Disclosure
- **Challenge:** Error Handling
- **Steps:**
  1. Triggered the login form in a way that caused an unhandled error
- **Evidence:** Raw, unhandled error text `[object Object]` was displayed directly on the login page instead of a graceful, user-friendly error message.
- **Impact:** Displaying raw internal error objects to end users can leak implementation details (stack traces, object structures, internal logic) that help an attacker fingerprint the application and plan further attacks.

### Finding 3 — SQL Injection (Authentication Bypass)
- **OWASP Category:** Injection → Broken Authentication
- **Challenge:** Login Admin
- **Steps:**
  1. On the login page, entered the following in the **Email** field:
     ```
     ' OR 1=1--
     ```
  2. Entered any arbitrary value (`x`) in the **Password** field
  3. Submitted the login form
- **Evidence:** Application confirmed: *"You successfully solved a challenge: Login Admin (Log in with the administrator's user account.)"* Logged in successfully as `admin@juice-sh.op` without knowing the real credentials, confirmed via the basket page header showing the admin account.
- **Root Cause:** The backend likely concatenates user input directly into a SQL query such as:
  ```sql
  SELECT * FROM users WHERE email = '<input>' AND password = '<input>'
  ```
  The payload `' OR 1=1--` closes the string early, injects an always-true condition (`OR 1=1`), and comments out the rest of the query (`--`), returning the first user record in the database (the admin account) regardless of password.
- **Impact:** Critical. Complete authentication bypass allows an attacker to log in as any user — including administrators — without valid credentials, leading to full account takeover.

### Finding 4 — DOM-Based Cross-Site Scripting (XSS)
- **OWASP Category:** Cross-Site Scripting (XSS)
- **Challenge:** DOM XSS
- **Steps:**
  1. Used the search bar at the top of the application
  2. Entered the payload:
     ```html
     <iframe src="javascript:alert(`xss`)">
     ```
  3. Submitted the search
- **Evidence:** Application confirmed: *"You successfully solved a challenge: DOM XSS (Perform a DOM XSS attack with `<iframe src="javascript:alert('xss')">`.)"* The payload was reflected directly into the page DOM (visible as an empty bordered box under the search results), confirming the unsanitized injection was rendered by the browser.
- **Root Cause:** The search feature takes user input and inserts it directly into the DOM without sanitizing or encoding HTML special characters, allowing injected HTML/JavaScript to be parsed and executed by the browser.
- **Impact:** An attacker could craft a malicious link containing a JavaScript payload (e.g., for session hijacking, credential phishing, or redirecting users to malicious sites) and trick victims into clicking it, executing arbitrary code in the victim's browser session.

### Finding 5 — Broken Access Control (IDOR)
- **OWASP Category:** Broken Access Control (Insecure Direct Object Reference)
- **Challenge:** View Basket
- **Steps:**
  1. Logged in as admin (via Finding 3) and confirmed admin's own basket (ID 1) via the UI
  2. Opened browser DevTools → Console
  3. Retrieved the current session's JWT token:
     ```js
     let token = localStorage.getItem('token');
     ```
  4. Sent a direct authenticated request to a different basket ID (belonging to another user) using the admin's own token:
     ```js
     fetch('http://localhost:3000/rest/basket/2', {
       headers: { 'Authorization': 'Bearer ' + token }
     })
     .then(r => r.json())
     .then(data => console.log(data));
     ```
- **Evidence:** Application confirmed: *"You successfully solved a challenge: View Basket (View another user's shopping basket.)"* The request returned `{status: 'success', data: {...}}` containing another user's basket contents, despite the authenticated session only legitimately owning basket ID 1.
- **Root Cause:** The `/rest/basket/:id` endpoint does not verify that the authenticated user actually owns the basket ID being requested — it only checks that *a* valid token was presented, not that the token's owner matches the resource.
- **Impact:** Any authenticated user (not just admin) can view — and potentially modify — any other user's shopping basket simply by changing a numeric ID in the request, exposing other customers' private order data.

---

## 4. Summary Table

| # | Finding | OWASP Category | Severity |
|---|---|---|---|
| 1 | Hidden Score Board | Security Misconfiguration | Low |
| 2 | Improper Error Handling | Improper Error Handling | Low |
| 3 | SQL Injection — Admin Login Bypass | Injection / Broken Authentication | **Critical** |
| 4 | DOM-Based XSS (Search) | Cross-Site Scripting | High |
| 5 | IDOR — View Another User's Basket | Broken Access Control | High |

---

## 5. Limitations

- Testing was conducted entirely against a **local Docker instance** of Juice Shop (not a live/public deployment), so findings reflect the application's known intentional vulnerabilities in a controlled lab environment only.
- Only a subset of Juice Shop's full challenge catalog was tested; this report does not represent a complete security audit of the application (e.g., Insecure Deserialization, XXE, Cryptographic Issues, and other categories were not explored in this assessment).
- No dedicated automated scanning tools (e.g., Burp Suite, sqlmap, nikto) were used in this assessment — all vulnerabilities were identified and exploited manually through browser DevTools (Network and Console tabs). Automated scans may surface additional findings not covered here.
- Testing was performed with attacker and victim on the same physical host (no network segmentation between "attacker" and "victim" machines), which does not fully replicate a realistic remote-attacker scenario.
- As this is an intentionally vulnerable training application, findings and payloads here are specific to Juice Shop's known design and should not be assumed to generalize directly to production systems without further context-specific testing.

---

## 6. Conclusion

This assessment confirmed five distinct vulnerabilities across the OWASP Top 10 categories — ranging from low-severity information disclosure issues (hidden pages, verbose error messages) to critical authentication bypass via SQL injection and high-severity client-side (XSS) and access control (IDOR) flaws. The exercise demonstrates how a combination of simple manual techniques (crafted input payloads, browser DevTools request manipulation) can uncover serious security weaknesses in a web application without needing specialized exploitation frameworks.
