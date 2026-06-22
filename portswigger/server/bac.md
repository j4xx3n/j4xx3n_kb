## [Unprotected admin functionality](https://portswigger.net/web-security/access-control/lab-unprotected-admin-functionality)

- Check robots.txt for admin panel

## [Unprotected admin functionality with unpredictable URL](https://portswigger.net/web-security/access-control/lab-unprotected-admin-functionality-with-unpredictable-url)

- Check the source code on the main page for a href to the admin panel.

`curl 'https://0a1e00820396f0d1819898ff002a00b1.web-security-academy.net' | grep '/admin'`


## [User role controlled by request parameter](https://portswigger.net/web-security/access-control/lab-user-role-controlled-by-request-parameter)

- There is a parameter in the cookie admin=false
- Change this in burp to admin=true
- Do this for view account -> admin panel -> delete carlos

## [User role can be modified in user profile](https://portswigger.net/web-security/access-control/lab-user-role-can-be-modified-in-user-profile)

- Change email and capture in burp
- Look for the roleid in the respose.
- Add roleid:2 to the JSON in the request.
- Access the admin panel and delete carlos

## [User ID controlled by request parameter](https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter)

- Log in and change id=wiener to id=carlos in the URL.


## [User ID controlled by request parameter, with unpredictable user IDs](https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter-with-unpredictable-user-ids)

- Go to Carlos's blog the userID will display in the URL
- Go to your profile and change the userID value in the url to carlos's


## [User ID controlled by request parameter with data leakage in redirect](https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter-with-data-leakage-in-redirect)

- Log in and change id=wiener to id=carlos in the URL.
- Capture request in burp.
- You can see carlos's account details and API key before you get redirected to the login page.


## [User ID controlled by request parameter with password disclosure](https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter-with-password-disclosure)

1. Log in using the supplied credentials and access the user account page.
2. Change the "id" parameter in the URL to `administrator`.
3. View the response in Burp and observe that it contains the administrator's password.
4. Log in to the administrator account and delete `carlos`.

## [Insecure direct object references](https://portswigger.net/web-security/access-control/lab-insecure-direct-object-references)

- Look for a view transcript option. 
- Change the file to 1.txt in the url to view other transcripts
- There should be some PII or creds in the 1.txt transcript

## [URL-based access control can be circumvented](https://portswigger.net/web-security/access-control/lab-url-based-access-control-can-be-circumvented)

- Add the following header to bypass access controle and access the admin panel:
```http
X-Original-URL: /admin
```

## [Method-based access control can be circumvented](https://portswigger.net/web-security/access-control/lab-method-based-access-control-can-be-circumvented)

- Change the request method to GET to access the admin panel

## [Multi-step process with no access control on one step](https://portswigger.net/web-security/access-control/lab-multi-step-process-with-no-access-control-on-one-step)

- Log in as admin and capture any request to change a user role (there should be an initial request and one to confirm the changes)
- Change the session cookie to wiener's cookie and attempt to change wiener's user type to administrator (the vulnerability lies in the second request that confirms the changes)

## [Referer-based access control](https://portswigger.net/web-security/access-control/lab-referer-based-access-control)

- Log in as admin and capture any request to change a user role (note the referrer header is pointing the the /admin panel)
- Change the session cookie to wiener's in the same request and attempt to change Wiener's user role to admin (this will not work without the referrer header)
