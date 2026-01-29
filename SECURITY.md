# Security Policy



# Java Microservices Security Instructions (SAST)

When writing or reviewing Java/Spring Boot code, follow these security practices:

## 1. Input Validation & Sanitization
- Always sanitize input using libraries like OWASP ESAPI or StringEscapeUtils.
- Validate all user input against a strict whitelist (type, length, format).

## 2. Secure Coding Patterns
- **SQL Injection:** Use PreparedStatements/Parameterized Queries (Spring Data JPA, JdbcTemplate). Never concatenate queries.
- **Deserialization:** Avoid 'ObjectInputStream' with untrusted data. Use JSON serialization with type validation.
- **Logging:** Sanitize user input before logging to prevent log injection (CRLF). Use parameterized logging (SLF4J).
- **Secrets:** Do not hardcode credentials. Use environment variables or Secrets Management systems.

## 3. Vulnerable Components (OWASP A06)
- Suggest updating dependencies to the latest stable versions.
- Remind users to run 'mvn dependency-check:check' or similar scanners.

## 4. Error Handling
- Do not expose stack traces in API responses. Return generic error messages.

## 5. Security Headers
- EnsureSpring Security is configured to add headers like `X-Content-Type-Options`, `X-Frame-Options`, and `Content-Security-Policy`.

