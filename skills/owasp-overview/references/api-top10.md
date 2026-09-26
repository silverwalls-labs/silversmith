# OWASP API Security Top 10 — quick reference

**Edition: API Security Top 10 2023**
([api-security.owasp.org](https://api-security.owasp.org/)) — checked
2026-09-26. Category links go to the official 2023 category page.

Apply this to every API surface (REST, GraphQL, gRPC gateways,
service-to-service) *in addition to* the general Top 10
([top10.md](top10.md)) — the failure modes below are API-specific and
routinely missed by a web-only review.

| # | Category | Most common mistake | Fix pattern | Cheat sheet |
|---|---|---|---|---|
| API1 | [Broken Object Level Authorization](https://api-security.owasp.org/editions/2023/en/0xa1-broken-object-level-authorization/) (BOLA) | Endpoint takes an object ID and never checks the caller owns/may access that object | Per-request, per-object authorization check against the session's user, in every function that touches an ID | [Authorization](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html) |
| API2 | [Broken Authentication](https://api-security.owasp.org/editions/2023/en/0xa2-broken-authentication/) | Weak/missing token validation, unauthenticated "internal" endpoints, credentials in URLs | Centralized, standard authentication on every route; short-lived validated tokens; MFA on sensitive ops | [Authentication](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html) |
| API3 | [Broken Object Property Level Authorization](https://api-security.owasp.org/editions/2023/en/0xa3-broken-object-property-level-authorization/) | Binding client input straight onto models (mass assignment) or returning whole objects (excessive data exposure) | Explicit allowlists of readable and writable properties per endpoint and role — never generic bind/serialize | [Mass Assignment](https://cheatsheetseries.owasp.org/cheatsheets/Mass_Assignment_Cheat_Sheet.html) |
| API4 | [Unrestricted Resource Consumption](https://api-security.owasp.org/editions/2023/en/0xa4-unrestricted-resource-consumption/) | No limits on request rate, payload size, pagination, or paid third-party calls | Rate limits, quotas, max payload/page sizes, timeouts, spend alerts — enforced server-side per client | [Denial of Service](https://cheatsheetseries.owasp.org/cheatsheets/Denial_of_Service_Cheat_Sheet.html) |
| API5 | [Broken Function Level Authorization](https://api-security.owasp.org/editions/2023/en/0xa5-broken-function-level-authorization/) | Admin endpoints "protected" only by obscurity or client-side role checks | Deny-by-default role checks at the routing/controller layer for every function, admin flows segregated | [Authorization](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html) |
| API6 | [Unrestricted Access to Sensitive Business Flows](https://api-security.owasp.org/editions/2023/en/0xa6-unrestricted-access-to-sensitive-business-flows/) | A legitimate flow (checkout, signup, posting) exposed with no protection against harmful automation | Identify business-critical flows and add anti-automation: device/human signals, rate limits, sequencing checks | (no dedicated sheet — see the [category page](https://api-security.owasp.org/editions/2023/en/0xa6-unrestricted-access-to-sensitive-business-flows/)) |
| API7 | [Server Side Request Forgery](https://api-security.owasp.org/editions/2023/en/0xa7-server-side-request-forgery/) | Fetching a user-supplied URI (webhooks, imports, previews) without validation | Allowlist protocols/domains/ports, block internal ranges, never follow redirects blindly | [SSRF Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html) |
| API8 | [Security Misconfiguration](https://api-security.owasp.org/editions/2023/en/0xa8-security-misconfiguration/) | Permissive CORS, verbose errors, missing security headers, unnecessary HTTP methods enabled | Hardened, repeatable API gateway/server configuration; only needed methods and headers exposed | [REST Security](https://cheatsheetseries.owasp.org/cheatsheets/REST_Security_Cheat_Sheet.html) |
| API9 | [Improper Inventory Management](https://api-security.owasp.org/editions/2023/en/0xa9-improper-inventory-management/) | Old versions (`/v1/`), debug and shadow endpoints still reachable because nobody tracks them | Maintained inventory of hosts, versions, and endpoints; retire or gate everything not current | (no dedicated sheet — see the [category page](https://api-security.owasp.org/editions/2023/en/0xa9-improper-inventory-management/)) |
| API10 | [Unsafe Consumption of APIs](https://api-security.owasp.org/editions/2023/en/0xaa-unsafe-consumption-of-apis/) | Trusting third-party API responses more than user input — no validation, generous parsing, followed redirects | Treat upstream API data as untrusted input: validate, sanitize, timeout, and limit what you accept | [Web Service Security](https://cheatsheetseries.owasp.org/cheatsheets/Web_Service_Security_Cheat_Sheet.html) |

For GraphQL-specific concerns (introspection, query depth/complexity,
batching abuse), also apply the
[GraphQL Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/GraphQL_Cheat_Sheet.html).
