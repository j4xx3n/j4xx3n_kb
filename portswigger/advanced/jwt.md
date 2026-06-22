## [JWT authentication bypass via unverified signature](https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-unverified-signature)

 - Log in, capture the GET /my-account request in Burp
 - Send to Repeater, find the JWT in the Cookie header
 - In the JWT Editor tab (or Burp's JWT plugin), decode the payload
 - Change "sub": "wiener" → "sub": "administrator"
 - Do not re-sign — just save the modified token as-is
 - Send the request → confirm you access the admin panel
 - Navigate to /admin and delete the target user ✅


## [JWT authentication bypass via flawed signature verification](https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-flawed-signature-verification)

 - Log in, capture a request with your JWT
 - Decode the header, change "alg": "RS256" → "alg": "none"
 - Change payload: "sub": "wiener" → "sub": "administrator"
 - Remove the signature entirely (keep the trailing dot): header.payload.)
 - Send the modified request to /admin
 - Delete the target user ✅


## [JWT authentication bypass via weak signing key](https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-weak-signing-key)

 - Log in, copy your JWT from Burp
 - Run hashcat against the JWT to crack the secret:


hashcat -a 0 -m 16500 <YOUR-JWT> /usr/share/wordlists/rockyou.txt

 - Wait for hashcat to return the secret (e.g. secret1)
 - Encode creacked secret in base64 
 - In Burp JWT Editor, create new symmetric key, add the encoded secret as the k value
 - In the repester use the jwt web token extention to change sub to administrator and Re-sign the token using the cracked secret (HS256)
 - Send to /admin → delete the target user ✅

## [JWT authentication bypass via jwk header injection](https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-jwk-header-injection)

 - In Burp, go to JWT Editor Keys → New RSA Key → generate a key
 - Log in, send a request with your JWT to Repeater
 - In the JWT tab, modify payload: "sub" → "administrator"
 - Click Attack → Embedded JWK — Burp injects your public key into the jwk header and signs with your private key
 - Send the request to /admin
 - Delete the target user ✅


## [JWT authentication bypass via jku header injection](https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-jku-header-injection)

 In Burp, go to JWT Editor Keys → New RSA Key → generate and note the key ID
 Copy the public key in JWK Set format:

json{
  "keys": [ { ...your public JWK... } ]
}


- On the exploit server, paste the JWK Set as the body, set `Content-Type: application/json`, and **Store**
- In Repeater, edit your JWT header:
  - Add `"jku": "https://YOUR-EXPLOIT-SERVER/exploit"`
  - Ensure `"kid"` matches the key ID in your JWK Set
- Change payload: `"sub"` → `"administrator"`
- Sign the token with your RSA private key (HS256 → RS256)
- Send to `/admin` → delete the target user ✅



## [JWT authentication bypass via kid header path traversal](https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-kid-header-path-traversal)



- In Burp, go to **JWT Editor Keys** → **New Symmetric Key**
- Set the key value (`k`) to the Base64-encoded null byte:
```
AA==
```
- Log in, send a JWT request to Repeater
- Edit the JWT header: set `"kid"` to a path traversal pointing to `/dev/null`:
```
../../../../../../../dev/null

 - Change payload: "sub" → "administrator"
 - Sign with your symmetric key (the secret is effectively a null byte, matching /dev/null)
 - Send to /admin → delete the target user ✅