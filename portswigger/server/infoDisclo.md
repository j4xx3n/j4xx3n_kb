
## [Information disclosure in error messages](https://portswigger.net/web-security/information-disclosure/exploiting/lab-infoleak-in-error-messages)

- Open the lab and navigate to any product page.
- In the URL, locate the `productId` parameter (e.g. `/product?productId=1`).
- Modify the `productId` value to a non-integer string such as `abc` or a special character.
- Submit the request and observe that the application throws a verbose error message.
- Read the error output — it will contain a full stack trace revealing the framework and version.
- Copy the framework version string (e.g. `Apache Struts 2.3.31`).
- Click **Submit solution** in the lab banner and paste the version number to complete the lab.
## [Information disclosure on debug page](https://portswigger.net/web-security/information-disclosure/exploiting/lab-infoleak-on-debug-page)

- Open the lab and view the HTML source of the home page (`Ctrl+U` or right-click > View Source).
- Search the source for a comment containing a link to a debug page — look for `cgi-bin/phpinfo.php` or similar.
- Navigate directly to that URL (e.g. `/cgi-bin/phpinfo.php`).
- On the phpinfo page, use `Ctrl+F` to search for `SECRET_KEY` or `secret`.
- Copy the value of the `SECRET_KEY` environment variable.
- Submit the key via the **Submit solution** button to complete the lab.

## [Source code disclosure via backup files](https://portswigger.net/web-security/information-disclosure/exploiting/lab-infoleak-via-backup-files)

- Open the lab and navigate to `/robots.txt` in the browser.
- Read the disallow entries — one will reveal a backup directory such as `/backup`.
- Browse to that backup directory (e.g. `/backup`).
- Identify and download any backup source file listed (e.g. `ProductTemplate.java.bak`).
- Open the downloaded file and search for `password` or database connection strings.
- Extract the hardcoded password value from the source code.
- Submit the password as the lab solution.

## [Authentication bypass via information disclosure](https://portswigger.net/web-security/information-disclosure/exploiting/lab-infoleak-authentication-bypass)

- Open the lab and intercept traffic using Burp Suite. Open the **Proxy > Intercept** tab.
- With intercept on, navigate to `/admin` in the browser.
- Observe a 401 response — the page is blocked to non-local requests.
- Send the `/admin` request to Burp Repeater.
- In Repeater, issue a `TRACE /admin` HTTP request to check which headers the server echoes back.
- In the TRACE response, locate a custom header that the server reflects — typically `X-Custom-IP-Authorization`.
- Add the header `X-Custom-IP-Authorization: 127.0.0.1` to a `GET /admin` request in Repeater.
- Confirm admin panel access is granted in the response.
- Locate the URL to delete a user in the admin panel response (e.g. `/admin/delete?username=carlos`).
- Send a GET request to that delete URL, still including the spoofed IP header.
- Confirm carlos has been deleted to complete the lab.

## [Information disclosure in version control history](https://portswigger.net/web-security/information-disclosure/exploiting/lab-infoleak-in-version-control-history)

- Open the lab and navigate to `/.git` in the browser — confirm the directory listing is accessible.
- On your local machine, run: `wget -r --no-parent https://<LAB-URL>/.git` to mirror the `.git` folder.
- Enter the downloaded directory and run `git log` to view the commit history.
- Identify a commit with a message referencing removal of a password or credentials.
- Run `git show <commit-hash>` to inspect the diff of that commit.
- Locate the admin password that was removed in the diff (the line prefixed with `-`).
- Return to the lab in the browser and log in as `administrator` with the recovered password.
- Navigate to the admin panel and delete the user carlos to complete the lab.