
## [Excessive trust in client-side controls](https://portswigger.net/web-security/logic-flaws/examples/lab-logic-flaws-excessive-trust-in-client-side-controls)

- Check if adding an item to the cart has a price parameter
- Change it to $0 and send the request to get a free item

## [High-level logic vulnerability](https://portswigger.net/web-security/logic-flaws/examples/lab-logic-flaws-high-level)

- Check if there is a quantity parameter in the POST /cart request. 
- Change the quantity to a negative number. This will make the price of the items in your cart go to negative as well.
- Add a new item to the cart. The total amount should go over $0 but should be under the store credit amount.
- Place the order to get a free item.

## [Inconsistent security controls](https://portswigger.net/web-security/logic-flaws/examples/lab-logic-flaws-inconsistent-security-controls)

- Try to access the /admin panel. You should get an error that only DOntWannaCry users all allowed to access the panel.
- Create an account and change the email address to one with @dontwannacry.com
- You should now have access to the admin panel


## [Flawed enforcement of business rules](https://portswigger.net/web-security/logic-flaws/examples/lab-logic-flaws-flawed-enforcement-of-business-rules)

- Log in to get the coupon code.
- Sign up for new letter to get a second coupon code.
- You can add the codes more than once if you alternate them.
- Reuse the two codes until the item is lower than your remaining store credit.


## [Low-level logic flaw](https://portswigger.net/web-security/logic-flaws/examples/lab-logic-flaws-low-level)

- Add an item to the cart and send the request to the intruder
- Add the payload to the quantity parameter and change the list to a NULL value list then set it to continue indefinably. 
- Go to your cart in the browser and refresh the page until your total amount is a negative number. Stop the intruder attack.
- Send a new request with the quantity of the items adding up to no more that $100 and no less than $0.
- Place the order.


## [Inconsistent handling of exceptional input](https://portswigger.net/web-security/logic-flaws/examples/lab-logic-flaws-inconsistent-handling-of-exceptional-input)

- Try to access the /admin endpoint. You will get blocked with a message that only DontWannaCry users are allowed.
- Try to add this email to see if the front end only checks the first 255 charectors in the email. 
kjmpzszavbhwndanxmmjwjmlsfbtrcwqqcbenfixjjbsastsrtzfyqoucwasgxdczcwmoerlzyhvnrklethupstpnmwcswugvattojgpanekqggkqtbtvuxsibzrhjoemizgulexausvohcqhnmbovzjaltyyynxpzhnxdyujrfouazkftjazxoqipelgpyjmpolbgstdfgtsfjbccnnejaubcfwhqapckenpodldspkgg@dontwannacry.com.web-security-academy.net


## [Weak isolation on dual-use endpoint](https://portswigger.net/web-security/logic-flaws/examples/lab-logic-flaws-weak-isolation-on-dual-use-endpoint)

- Capture change password request and send to the repeater
- Notice there is no requirement to enter the old password and a parameter to set the user who's password you are changing.
- Change the username to administrator to takeover the account.


## [Insufficient workflow validation](https://portswigger.net/web-security/logic-flaws/examples/lab-logic-flaws-insufficient-workflow-validation)

- Buy any item you can afford. Notice the following request is made to confirm the order:
`GET /cart/order-confirmation?order-confirmation=true`
- Send the confirmation request to the repeater
- Add the leather jacket to the cart
- Send the confirmation request again. This should give you a free item.


## [Authentication bypass via flawed state machine](https://portswigger.net/web-security/logic-flaws/examples/lab-logic-flaws-authentication-bypass-via-flawed-state-machine)

- Log into the site and study the request that are made.
- Notice there is a request to `GET /role-selector` . Drop this request in the login process to bypass the role identification.
- You should then have access to the admin panel



## [Infinite money logic flaw](https://portswigger.net/web-security/logic-flaws/examples/lab-logic-flaws-infinite-money)

- Log in as `wiener:peter`
- [ ]  Sign up for the newsletter → grab the `SIGNUP30` coupon code

**Understand the flaw manually first**

- [ ]  Add a $10 gift card to your cart
- [ ]  Apply `SIGNUP30` at checkout (30% off → you pay $7, get a $10 card)
- [ ]  Complete the order, copy the gift card code
- [ ]  Redeem it on `/my-account` → net gain of **$3 per cycle**

**Set up the Burp Macro (to automate the cycle)**

- [ ]  Go to **Settings → Sessions → Session handling rules → Add**
- [ ]  Under **Scope**, set URL scope to "Include all URLs"
- [ ]  Under **Rule actions**, add "Run a macro" → record these 5 requests in order:
    1. `POST /cart`
    2. `POST /cart/coupon`
    3. `POST /cart/checkout`
    4. `GET /cart/order-confirmation?order-confirmed=true`
    5. `POST /gift-card`

**Configure the macro to extract the gift card code dynamically**

- [ ]  Select the `GET /cart/order-confirmation` request → **Configure item** → **Add** custom parameter named `gift-card`, highlight the code in the response
- [ ]  Select `POST /gift-card` → **Configure item** → set `gift-card` param to be derived from **prior response (response 4)**
- [ ]  Click **Test macro** and confirm the `POST /gift-card` gets a `302` response with the correct code

**Run the attack with Burp Intruder**

- [ ]  Send `GET /my-account` to **Intruder** (Sniper attack)
- [ ]  Payload type: **Null payloads** → generate **412 payloads**
- [ ]  Resource pool: **max 1 concurrent request** (important — keeps the macro sequential)
- [ ]  Start the attack and let it run

**Finish**

- [ ]  Once done, you'll have enough credit (~$1236) to buy the leather jacket
- [ ]  Purchase the jacket to solve the lab ✅

The key insight: buying a $10 gift card for $7 (with the 30% coupon) and redeeming it nets $3 profit per loop — the site never invalidates or rate-limits the coupon reuse.


## [Authentication bypass via encryption oracle](https://portswigger.net/web-security/logic-flaws/examples/lab-logic-flaws-authentication-bypass-via-encryption-oracle)


**Phase 1 — Discover the oracle**

- [ ]  Log in as `wiener:peter` with **"Stay logged in"** checked — note the encrypted `stay-logged-in` cookie
- [ ]  Post a comment with an **invalid email address** (e.g. `foo`) — notice a `notification` cookie is set in the response, and the error message on the next page says `Invalid email address: foo` in plaintext
- [ ]  Deduce: the `notification` cookie is the **encryption oracle** — the `email` parameter encrypts arbitrary input, and the `notification` cookie decrypts arbitrary ciphertext
- [ ]  Send both the `POST /post/comment` request and the `GET /post?postId=x` request to Repeater. Label them **`encrypt`** and **`decrypt`**

**Phase 2 — Decode the stay-logged-in cookie**

- [ ]  In the `decrypt` tab, replace the `notification` cookie value with your `stay-logged-in` cookie value
- [ ]  Send it — the response reveals the decrypted format: `wiener:1598530205184` (username:timestamp)
- [ ]  **Copy the timestamp** — you'll need it

**Phase 3 — Encrypt `administrator:timestamp` (with padding)**

- [ ]  In the `encrypt` tab, set `email` to `xxxxxxxxxadministrator:YOUR-TIMESTAMP` (9 `x`s prefix)
    - Why 9? The prefix `Invalid email address:` is 23 bytes. Adding 9 chars makes it 32 bytes — a clean multiple of 16 (two full cipher blocks to strip)
- [ ]  Send it and copy the `notification` cookie from the response

**Phase 4 — Strip the prefix bytes in Decoder**

- [ ]  Send the new `notification` cookie value to **Burp Decoder**
- [ ]  **URL-decode**, then **Base64-decode** it — you now have raw bytes
- [ ]  Switch to the **Hex** view in Repeater's message editor and **delete the first 32 bytes** (which correspond to the `Invalid email address:` prefix + your 9 padding chars)
	- You can right click and use delete bytes
- [ ]  **Re-encode**: Base64-encode → URL-encode the result

**Phase 5 — Verify the stripped ciphertext**

- [ ]  Paste the re-encoded value into the `notification` cookie of the `decrypt` tab and send it
- [ ]  Confirm the decrypted output is exactly `administrator:YOUR-TIMESTAMP` with no prefix — if you get a block size error, your byte count is off; re-check the padding

**Phase 6 — Use the forged cookie to log in as admin**

- [ ]  From Proxy history, send the `GET /` request to Repeater
- [ ]  **Delete the `session` cookie entirely**
- [ ]  Set the `stay-logged-in` cookie to your crafted, stripped, re-encoded ciphertext
- [ ]  Send — confirm you are logged in as `administrator`
- [ ]  Browse to `/admin/delete?username=carlos` → lab solved ✅

**Key concept:** The server uses the same encryption key for both the `stay-logged-in` and `notification` cookies. The comment `email` parameter acts as an encryption oracle (encrypt anything you want) and the `notification` cookie acts as a decryption oracle (decrypt anything you want). By encrypting `administrator:timestamp` with a known-length prefix, then stripping exactly those prefix bytes from the ciphertext, you forge a valid admin `stay-logged-in` cookie without ever knowing the key.