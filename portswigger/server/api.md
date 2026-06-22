## [Exploiting an API endpoint using documentation](https://portswigger.net/web-security/api-testing/lab-exploiting-api-endpoint-using-documentation)

 - Log in as wiener:peter and update your email address
 - In Proxy > HTTP history, find the PATCH /api/user/wiener request and send it to Repeater
 - In Repeater, remove /wiener from the path so it becomes /api/user — send and note the error
 - Remove /user so the path becomes /api — send and confirm API documentation is returned
 - Right-click the response and select Show response in browser, copy the URL and open it in Burp's browser
 - In the interactive docs, click the DELETE row, enter carlos, and click Send request to solve the lab

## [Exploiting server-side parameter pollution in a query string](https://portswigger.net/web-security/api-testing/server-side-parameter-pollution/lab-exploiting-server-side-parameter-pollution-in-query-string)

 - Trigger a password reset for the administrator user
 - In Proxy > HTTP history, find the POST /forgot-password request and send it to Repeater; also note the /static/js/forgotPassword.js file
 - In Repeater, change username to an invalid value like administratorx — confirm Invalid username error
 - Inject a second parameter using a URL-encoded &: username=administrator%26x=y — confirm Parameter is not supported error
 - Truncate the query string with a URL-encoded #: username=administrator%23 — confirm Field not specified error
 - Inject a field parameter and truncate after it: username=administrator%26field=x%23 — confirm Invalid field error
 - Send the request to Intruder, set payload position on the field value: username=administrator%26field=§x§%23
 - Use the built-in Server-side variable names payload list and run the attack — note username and email both return 200
 - Back in Repeater, set field=reset_token: username=administrator%26field=reset_token%23 — copy the reset token from the response
 - In the browser, navigate to /forgot-password?reset_token=<your-token> and set a new password
 - Log in as administrator, go to the Admin panel, and delete carlos to solve the lab

## [Finding and exploiting an unused API endpoint](https://portswigger.net/web-security/api-testing/lab-exploiting-unused-api-endpoint)

 - Browse to any product page — in Proxy > HTTP history note the GET /api/products/{id}/price request and send to Repeater
 - Change the method to OPTIONS and send — confirm the response lists GET and PATCH as allowed methods
 - Change the method to PATCH and send — confirm an Unauthorized error (authentication required)
 - Log in as wiener:peter, click the Lightweight "l33t" Leather Jacket, find its GET /api/products/1/price request in history and send to Repeater
 - Change the method to PATCH and send — note the error specifying Content-Type must be application/json
 - Add header Content-Type: application/json and set the body to {} — send and note the error requiring a price parameter
 - Set the body to {"price":0} and send — confirm success
 - Reload the jacket page in the browser — confirm price is now $0.00
 - Add the jacket to your basket and click Place order to solve the lab


## [Exploiting a mass assignment vulnerability](https://portswigger.net/web-security/api-testing/lab-exploiting-mass-assignment-vulnerability)

 - Log in as wiener:peter, add the Lightweight "l33t" Leather Jacket to your basket and attempt to place the order — confirm insufficient credit
 - In Proxy > HTTP history, find both the GET and POST requests to /api/checkout
 - Compare the two — note the GET response includes a chosen_discount parameter with a percentage field that is absent from the POST request body
 - Send the POST /api/checkout request to Repeater
 - Add chosen_discount to the JSON body and send — confirm no error:

json  {"chosen_discount":{"percentage":0},"chosen_products":[{"product_id":"1","quantity":1}]}

 - Change percentage to "x" and send — confirm a validation error proving the parameter is being processed
 - Change percentage to 100 and send — confirm the order goes through to solve the lab