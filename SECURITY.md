
# Copilot Security Policy & SAST Gate — Java Microservices

> **Purpose**
> These instructions are repository-wide guidance for GitHub Copilot (and reviewers) to ensure code changes follow secure-by-default practices and pass SAST gates for common vulnerability classes: SQLi, CSRF, XSS, insecure deserialization, broken access control, insecure data handling, and race conditions. [1](https://docs.github.com/en/copilot/how-tos/configure-custom-instructions/add-repository-instructions)[2](https://code.visualstudio.com/docs/copilot/customization/custom-instructions)

---

## 1) Non-negotiable Security Principles (Applies to every change)

1. **Server-side enforcement wins**: never rely on client/UI checks for authorization, validation, or security decisions. [3](https://owasp.org/Top10/2021/A01_2021-Broken_Access_Control/)[4](https://devguide.owasp.org/en/04-design/02-web-app-checklist/07-access-controls/)  
2. **Deny-by-default**: new endpoints/resources are *not accessible* unless explicitly authorized. [3](https://owasp.org/Top10/2021/A01_2021-Broken_Access_Control/)[4](https://devguide.owasp.org/en/04-design/02-web-app-checklist/07-access-controls/)  
3. **Least privilege** everywhere (service accounts, DB users, roles/scopes). [3](https://owasp.org/Top10/2021/A01_2021-Broken_Access_Control/)[4](https://devguide.owasp.org/en/04-design/02-web-app-checklist/07-access-controls/)  
4. **Validate input + encode output** with an allow-list mindset; never trust data from HTTP, headers, JWT claims, MQ events, files, or DB. [5](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)[6](https://cheatsheetseries.owasp.org/cheatsheets/Java_Security_Cheat_Sheet.html)  
5. **Security checks must be testable**: add unit/integration tests that prove authorization, validation, and concurrency protections. [4](https://devguide.owasp.org/en/04-design/02-web-app-checklist/07-access-controls/)[7](https://portswigger.net/web-security/race-conditions)  

---

## 2) Required SAST Coverage (What Copilot must avoid / implement)

When generating or modifying code, Copilot MUST proactively address **all** categories below and add defensive code + tests as needed.

### A) SQL Injection (SQLi) — MUST PREVENT
**Never** build SQL/HQL/JPQL queries by concatenating untrusted input.
- ✅ Use **prepared statements / parameterized queries** (JDBC `PreparedStatement`, JPA parameters, Spring Data binding).
- ✅ Allow-list only where parameters cannot be bound (e.g., dynamic column/table names).
- ❌ Avoid `Statement.executeQuery("..."+userInput)` and string-interpolated SQL. [8](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)[6](https://cheatsheetseries.owasp.org/cheatsheets/Java_Security_Cheat_Sheet.html)  

**Copilot SAST expectations**
- Ensure code patterns align with OWASP SQLi prevention (prepared statements, stored procedures, allow-lists; escaping is discouraged). [8](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)  

---

### B) Cross-Site Request Forgery (CSRF) — MUST PREVENT (when cookies/session auth is used)
If the service is accessed by browsers and uses **cookies** for auth/session:
- ✅ Prefer framework built-in CSRF protection (e.g., Spring Security CSRF).
- ✅ Add CSRF tokens for state-changing requests and validate server-side.
- ✅ Consider defense-in-depth such as Fetch Metadata headers and SameSite cookies (with care). [9](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)[10](https://owasp.org/www-community/attacks/csrf)  

**Important**
- XSS can bypass CSRF mitigations; therefore prevent XSS as a prerequisite. [9](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)[5](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)  

> If the service is purely stateless and uses Authorization headers (e.g., JWT bearer) and no cookies, CSRF risk is typically reduced; still ensure no cookie-based auth is introduced without CSRF defenses. [9](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)[10](https://owasp.org/www-community/attacks/csrf)  

---

### C) Cross-Site Scripting (XSS) — MUST PREVENT
- ✅ Apply context-aware output encoding/escaping in any server-rendered HTML or templating.
- ✅ Sanitize HTML only when truly required; avoid unsafe sinks.
- ❌ Avoid rendering untrusted input without encoding. [5](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)[11](https://owasp.deteact.com/cheat/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)  

**If any UI/frontend exists**
- Avoid dangerous DOM APIs/escape hatches and ensure proper encoding + sanitization where needed. [5](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)  

---

### D) Insecure Deserialization — MUST PREVENT
- ❌ Do not deserialize untrusted data using native Java serialization (`ObjectInputStream.readObject`) unless strongly hardened.
- ✅ Prefer safe data formats (JSON) and strict schemas.
- ✅ If deserialization is unavoidable: restrict allowable classes (allow-list), validate size/structure, and harden deserialization. [12](https://cheatsheetseries.owasp.org/cheatsheets/Deserialization_Cheat_Sheet.html)[13](https://owasp.org/www-community/vulnerabilities/Insecure_Deserialization)  

---

### E) Broken Access Control — MUST PREVENT (A01)
- ✅ Every endpoint must have explicit authorization checks (method-level and/or route-level).
- ✅ Enforce **record ownership** checks (prevent IDOR/BOLA).
- ✅ Ensure non-GET mutating endpoints (POST/PUT/PATCH/DELETE) are protected.
- ✅ Centralize authorization logic; log authorization failures. [3](https://owasp.org/Top10/2021/A01_2021-Broken_Access_Control/)[4](https://devguide.owasp.org/en/04-design/02-web-app-checklist/07-access-controls/)  

**Do not**
- ❌ Trust roles/scopes coming from client input.
- ❌ Expose admin-only endpoints by mistake (missing annotations/config). [3](https://owasp.org/Top10/2021/A01_2021-Broken_Access_Control/)  

---

### F) Insecure Data Handling — MUST PREVENT
- ✅ Minimize sensitive data storage (don’t store it unless necessary).
- ✅ Encrypt sensitive data at rest with vetted algorithms and proper key management practices.
- ✅ Never log secrets/PII; redact or hash where required; avoid stack traces in responses.
- ✅ Use dedicated secret/key management where possible; avoid hard-coded secrets. [14](https://cheatsheetseries.owasp.org/cheatsheets/Cryptographic_Storage_Cheat_Sheet.html)[15](https://docs.boostsecurity.io/rules/semgrep_java.html)  

---

### G) Race Conditions / Concurrency Bugs — MUST PREVENT
Race conditions happen when requests/threads operate concurrently on shared state without safeguards, producing unintended outcomes. [7](https://portswigger.net/web-security/race-conditions)[16](https://owasp.org/www-chapter-bangkok/slides/2024/2024-07-05_The-Race-is-On.pdf)  

**Required mitigations (as applicable)**
- ✅ Use **transactional** boundaries and appropriate isolation for critical state transitions.
- ✅ Use atomic DB operations, row-level locking, or optimistic locking/versioning.
- ✅ Implement idempotency keys for payment/order/transfer-like operations.
- ✅ Avoid “check-then-act” patterns without concurrency control. [7](https://portswigger.net/web-security/race-conditions)[16](https://owasp.org/www-chapter-bangkok/slides/2024/2024-07-05_The-Race-is-On.pdf)  

---

## 3) SAST Tooling & CI Gates (Minimum Standard)

### Primary SAST (Required)
1. **CodeQL** (Java/Kotlin) — run **default + security-extended** suites.
   - Ensure results include detection for classes like XSS, deserialization of user-controlled data, and disabled Spring CSRF protection, among others. [17](https://docs.github.com/en/enterprise-server@3.17/code-security/code-scanning/managing-your-code-scanning-configuration/java-kotlin-built-in-queries)  

2. **Semgrep** — run a security ruleset focused on injection, authz, crypto, and unsafe patterns.
   - Use custom rules where needed for project-specific sinks/sources.
   - Track and triage false positives with documented suppressions. [18](https://semgrep.dev/docs/learn/vulnerabilities/sql-injection)[15](https://docs.boostsecurity.io/rules/semgrep_java.html)  

### Optional (Strongly Recommended)
- SpotBugs + FindSecBugs (Java security bug patterns).
- Dependency scanning (SCA) and secret scanning (not SAST, but mandatory for release readiness).

### Gate Policy
- **Block merge** if a finding is:
  - Critical/High severity for any category listed in Section 2, OR
  - Medium severity with an exploitable path (security review required).
- Suppressions must include:
  - justification,
  - link to ticket,
  - and proof (tests or reasoning) that it’s not exploitable.

---

## 4) Secure Coding Defaults for Java Microservices (Spring-friendly)

### Authentication & Authorization
- Prefer centralized identity (OIDC/OAuth2) and enforce authorization per endpoint + per resource.
- Use method-level security for sensitive business operations and verify ownership in service layer. [3](https://owasp.org/Top10/2021/A01_2021-Broken_Access_Control/)[4](https://devguide.owasp.org/en/04-design/02-web-app-checklist/07-access-controls/)  

### Input Validation
- Validate request DTOs using allow-list constraints; reject unknown fields where feasible.
- Treat headers, query params, and JWT claims as untrusted. [6](https://cheatsheetseries.owasp.org/cheatsheets/Java_Security_Cheat_Sheet.html)[5](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)  

### Output Handling
- Never reflect untrusted input into HTML/JS contexts without encoding.
- Return generic error messages; avoid leaking internals. [5](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)[11](https://owasp.deteact.com/cheat/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)  

### Data Protection
- Encrypt sensitive data at rest; use proper key management; don’t store sensitive data unnecessarily. [14](https://cheatsheetseries.owasp.org/cheatsheets/Cryptographic_Storage_Cheat_Sheet.html)  

### Concurrency
- For balance/stock/booking/order flows: implement atomicity (tx + locks) and idempotency for retries. [7](https://portswigger.net/web-security/race-conditions)[16](https://owasp.org/www-chapter-bangkok/slides/2024/2024-07-05_The-Race-is-On.pdf)  

---

## 5) What Copilot MUST DO in Pull Requests

For every PR that changes APIs, persistence, or authorization-sensitive code, Copilot should:
1. Identify relevant risks among Section 2 categories.
2. Implement mitigations (not just comments).
3. Add tests (unit/integration) that demonstrate:
   - unauthorized access is rejected (403),
   - IDOR is prevented (ownership checks),
   - SQLi payloads don’t alter queries (parameter binding),
   - CSRF protections exist where cookies are used,
   - concurrent requests cannot double-spend / double-book.
4. Ensure SAST gates are clean or findings are documented and approved.

---

## 6) Reference Security Guidance (Canonical)
- OWASP Top 10 — Broken Access Control overview and prevention guidance. [3](https://owasp.org/Top10/2021/A01_2021-Broken_Access_Control/)  
- OWASP Cheat Sheet Series:
  - SQL Injection Prevention. [8](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)  
  - CSRF Prevention. [9](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)[10](https://owasp.org/www-community/attacks/csrf)  
  - XSS Prevention. [5](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)[11](https://owasp.deteact.com/cheat/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)  
  - Deserialization Security. [12](https://cheatsheetseries.owasp.org/cheatsheets/Deserialization_Cheat_Sheet.html)[13](https://owasp.org/www-community/vulnerabilities/Insecure_Deserialization)  
  - Cryptographic Storage / protecting data at rest. [14](https://cheatsheetseries.owasp.org/cheatsheets/Cryptographic_Storage_Cheat_Sheet.html)  
- Race conditions concept and why they occur in concurrent request handling. [7](https://portswigger.net/web-security/race-conditions)[16](https://owasp.org/www-chapter-bangkok/slides/2024/2024-07-05_The-Race-is-On.pdf)  
- GitHub CodeQL Java query coverage (examples include XSS, deserialization of user-controlled data, and disabled Spring CSRF protection). [17](https://docs.github.com/en/enterprise-server@3.17/code-security/code-scanning/managing-your-code-scanning-configuration/java-kotlin-built-in-queries)  

---
