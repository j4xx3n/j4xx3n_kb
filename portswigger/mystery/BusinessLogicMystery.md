# PortSwigger Business Logic Flaws — Lab Solver Checklist

## Step 1 — Identify the Lab Type

Use the clues below to figure out which vulnerability you're dealing with before diving in.

| Clue                                                                   | Likely Lab Type                                                                  |
| ---------------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| Cart has a `price` parameter in the POST request                       | Excessive Trust in Client-Side Controls                                          |
| Cart has a `quantity` parameter you can manipulate                     | High-Level Logic Vulnerability                                                   |
| Admin panel restricted to a specific email domain                      | Inconsistent Security Controls **or** Inconsistent Handling of Exceptional Input |
| Two coupon codes available at login/newsletter signup                  | Flawed Enforcement of Business Rules                                             |
| Cart total overflows into a negative number with huge quantities       | Low-Level Logic Flaw                                                             |
| Order confirmation endpoint (`/cart/order-confirmation`) is replayable | Insufficient Workflow Validation                                                 |
| A role-selector step exists after login                                | Authentication Bypass via Flawed State Machine                                   |
| A discount coupon + gift cards are available in the store              | Infinite Money Logic Flaw                                                        |
| `stay-logged-in` cookie + comment form with a `notification` cookie    | Authentication Bypass via Encryption Oracle                                      |

---

## Step 2 — Solve It

---

### Excessive Trust in Client-Side Controls

- [ ] Add the target item to your cart and intercept the `POST /cart` request in Burp
- [ ] Find the `price` parameter and change it to `1` (or `0`)
- [ ] Forward the request — the item is now in your cart at the modified price
- [ ] Place the order

---

### High-Level Logic Vulnerability

- [ ] Add any item to your cart and intercept the `POST /cart` request
- [ ] Change the `quantity` parameter to a **negative number**
- [ ] Your cart total will go negative — now add the target (expensive) item
- [ ] Adjust quantities so the final total is **between $0 and your store credit**
- [ ] Place the order

---

### Inconsistent Security Controls

- [ ] Try to access `/admin` — note it's restricted to `@dontwannacry.com` emails
- [ ] Register a new account with any email and log in
- [ ] Go to account settings and **change your email** to anything `@dontwannacry.com`
- [ ] Access `/admin` — you now have access
- [ ] Complete the lab objective (e.g. delete a user)

---

### Flawed Enforcement of Business Rules

- [ ] Log in and note the first coupon code
- [ ] Sign up for the newsletter to receive a second coupon code
- [ ] Add the target item to your cart
- [ ] Apply coupon 1 → then coupon 2 → then coupon 1 again (alternate them repeatedly)
- [ ] Keep alternating until the price drops below your store credit
- [ ] Place the order

---

### Low-Level Logic Flaw

- [ ] Add the target item to your cart and send the `POST /cart` request to **Intruder**
- [ ] Set the `quantity` parameter as the payload position
- [ ] Use a **Null payloads** list, set to run **indefinitely**
- [ ] Watch your cart total in the browser — **stop the attack when the total goes negative**
- [ ] Send additional `POST /cart` requests to nudge the total to between **$0 and $100**
- [ ] Place the order

---

### Inconsistent Handling of Exceptional Input

- [ ] Try `/admin` — note it's restricted to `@dontwannacry.com`
- [ ] Register with this specially crafted email (the server truncates at 255 chars, landing on `@dontwannacry.com`):
    
    ```
    kjmpzszavbhwndanxmmjwjmlsfbtrcwqqcbenfixjjbsastsrtzfyqoucwasgxdczcwmoerlzyhvnrklethupstpnmwcswugvattojgpanekqggkqtbtvuxsibzrhjoemizgulexausvohcqhnmbovzjaltyyynxpzhnxdyujrfouazkftjazxoqipelgpyjmpolbgstdfgtsfjbccnnejaubcfwhqapckenpodldspkgg@dontwannacry.com.web-security-academy.net
    ```
    
- [ ] Log in — your stored email will be truncated to end in `@dontwannacry.com`
- [ ] Access `/admin` and complete the lab objective

---

### Weak Isolation on Dual-Use Endpoint

- [ ] Log in and go to the change password function — intercept the request in Burp
- [ ] Notice there is **no current-password field** and a `username` parameter is present
- [ ] Send the request to **Repeater** and change `username` to `administrator`
- [ ] Set the new password to anything you choose and send the request
- [ ] Log in as `administrator` with your new password
- [ ] Complete the lab objective (e.g. delete a user)

---

### Insufficient Workflow Validation

- [ ] Buy a cheap item you can afford — note the confirmation request: `GET /cart/order-confirmation?order-confirmation=true`
- [ ] Send that confirmation request to **Repeater**
- [ ] Add the target (expensive) item to your cart
- [ ] **Replay the confirmation request** — the order goes through for free
- [ ] Lab solved ✅

---

### Authentication Bypass via Flawed State Machine

- [ ] Start logging in and watch all requests in Burp Proxy
- [ ] Identify the `GET /role-selector` request that fires after login
- [ ] Log out, then log in again — this time **drop the `/role-selector` request** in Proxy
- [ ] Browse to the site — you should have landed with a default high-privilege role
- [ ] Access `/admin` and complete the lab objective

---

### Infinite Money Logic Flaw

- [ ] Log in as `wiener:peter` and sign up for the newsletter → save the `SIGNUP30` code
- [ ] **Verify the flaw manually**: add a $10 gift card, apply `SIGNUP30` (pay $7), complete the order, redeem the card → net +$3
- [ ] Set up a **Burp Macro** (Settings → Sessions → Session handling rules → Add):
    - Scope: all URLs
    - Record these 5 requests in order:
        1. `POST /cart`
        2. `POST /cart/coupon`
        3. `POST /cart/checkout`
        4. `GET /cart/order-confirmation?order-confirmed=true`
        5. `POST /gift-card`
- [ ] On request 4, configure a custom parameter `gift-card` by highlighting the code in the response
- [ ] On request 5, set the `gift-card` param to derive from **prior response (response 4)**
- [ ] Test the macro — confirm `POST /gift-card` returns a `302`
- [ ] Send `GET /my-account` to **Intruder**: Null payloads, **412 iterations**, max **1 concurrent request**
- [ ] Run the attack and wait — you'll accumulate ~$1,236 in store credit
- [ ] Buy the leather jacket ✅

---

### Authentication Bypass via Encryption Oracle

- [ ] Log in as `wiener:peter` with **Stay logged in** — note the `stay-logged-in` cookie
- [ ] Post a comment with an **invalid email** (e.g. `foo`) — note the `notification` cookie in the response; it decrypts to `Invalid email address: foo`
- [ ] Send the comment `POST` (**encrypt tab**) and a `GET /post` (**decrypt tab**) to Repeater
- [ ] **Decode the admin cookie format**: put `stay-logged-in` into the decrypt tab → reveals `wiener:TIMESTAMP` — copy the timestamp
- [ ] **Encrypt the target**: in the encrypt tab, set `email` to `xxxxxxxxxadministrator:TIMESTAMP` (9 x's prefix so the 23-byte error prefix + 9 = 32 bytes = 2 full blocks to strip)
- [ ] Copy the new `notification` cookie value → send to **Decoder**
    - URL-decode → Base64-decode → switch to Hex view → **delete the first 32 bytes** → Base64-encode → URL-encode
- [ ] **Verify**: paste the result into the decrypt tab's `notification` cookie → confirm it decrypts to exactly `administrator:TIMESTAMP`
- [ ] In Proxy history, send `GET /` to Repeater — **delete the `session` cookie**, set `stay-logged-in` to your forged value
- [ ] Send — confirm you're logged in as `administrator`
- [ ] Browse to `/admin/delete?username=carlos` ✅