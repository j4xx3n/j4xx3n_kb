**Setup**

- [ ]  Log in with `wiener:peter`, capture a request containing your JWT in Burp
- [ ]  Send to Repeater, open the **JSON Web Tokens** tab to inspect the token

---

**Identify which vulnerability applies — work through these in order:**

**Is the signature verified at all?**

- [ ]  Change `"sub": "wiener"` → `"sub": "administrator"` with no other changes
- [ ]  Send to `/my-account` or `/admin` — if it works → **unverified signature**, go delete carlos and you're done ✅

**Is `alg: none` accepted?**

- [ ]  Change header `"alg"` to `"none"`, change sub to `administrator`, strip the signature (keep trailing dot: `header.payload.`)
- [ ]  Send to `/admin` — if it works → **flawed signature verification**, delete carlos ✅

**Is it a weak HS256 secret?**

- [ ]  Copy the raw JWT, run hashcat:

```
  hashcat -a 0 -m 16500 <your-jwt> /usr/share/wordlists/rockyou.txt
```

- [ ]  If a secret is found: base64-encode it, create a symmetric key in JWT Editor with that `k` value, change sub to `administrator`, re-sign, send to `/admin` → delete carlos ✅

**Does the header contain a `jwk` parameter?**

- [ ]  JWT Editor Keys → New RSA Key → generate
- [ ]  Change sub to `administrator`, click **Attack → Embedded JWK**
- [ ]  Send to `/admin` — if it works → **JWK header injection**, delete carlos ✅

**Does the header contain a `jku` parameter?**

- [ ]  JWT Editor Keys → New RSA Key → generate, note the key ID
- [ ]  Copy public key as JWK Set, paste onto exploit server with `Content-Type: application/json`, click Store
- [ ]  Edit JWT header: set `"jku"` to your exploit server URL, ensure `"kid"` matches
- [ ]  Change sub to `administrator`, sign with your RSA private key
- [ ]  Send to `/admin` → delete carlos ✅

**Is there a `kid` header you can path-traverse?**

- [ ]  JWT Editor Keys → New Symmetric Key, set `k` to `AA==` (base64 null byte)
- [ ]  Edit JWT header: set `"kid"` to `../../../../../../../dev/null`
- [ ]  Change sub to `administrator`, sign with your null-byte symmetric key
- [ ]  Send to `/admin` → delete carlos ✅

---

**If none of the above work**, look closely at the JWT header for any unusual parameters (`x5u`, `x5c`, custom `kid` values pointing to a URL) and treat them similarly to `jku` — they may be fetching a key from an attacker-controllable location.