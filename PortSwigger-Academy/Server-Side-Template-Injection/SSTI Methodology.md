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


```
SURFACE ATTACK :   Any user controllable input that get reflected on the web page.

SSTI Vulnerability  it's like SQL, XSS.
```

#### NOTE 
**if u can't achieve remote code execution , You can still leverage server-side template injection vulnerabilities for other high-severity exploits, such as file path traversal, to gain access to sensitive data.**


![](images/Pasted%20image%2020260929051438.png)



### Detecting SSTI

fuzzing this payload :
```
$
{{
<
%
[
%
'
"
}}
%
\
```


##### Plaintext context

`${7*7}`


##### Code context

user input being placed within a template expression.

```
greeting = getQueryParameter('greeting') engine.render("Hello {{greeting}}", data)
```

ensuring that input doesn't contain XSS vuln:
`data.username<tag>`

You might get:

- `Hello` → blank output
- encoded `<tag>`
- an error

breaking out of the statement:
`data.username}}<tag>`

```
Hello carlos <tag>
```



### Identifying Template Engine Type


##### Method 1: Trigger an Error

`<% foobar %>`   ,  an error:   `undefined local variable or method `foobar'
and the stack trace mentions: `erb.rb`

#####  Method 2: Mathematical Expressions

| Engine     | Example Syntax |
| ---------- | -------------- |
| Jinja2     | `{{7*7}}`      |
| Twig       | `{{7*7}}`      |
| ERB        | `<%= 7*7 %>`   |
| Freemarker | `${7*7}`       |
| Velocity   | `#set($x=7)`   |
brute forcing till one of these payloads get evaluated.

NOTE : One Payload Is Not Enough

in Twig `{{7*'7'}}`    -> 49
in Jinja2 `{{7*'7'}}`   -> 7777777


### EXPLOITATION STAGE

On Exploitation Stage the following Resources can be helpful :

```
https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Template%20Injection#freemarker
```

```
https://portswigger.net/research/server-side-template-injection
```

```
https://hacktricks.wiki/en/index.html
```


After identifying the template engine (ERB, Jinja2, Twig, Tornado, etc.) used :


1)  go to its documentation and look for:

- Security warnings
- Dangerous built-in functions
- File access features
- Command execution features
- Template inclusion/import features

2) Look for exploitation on google that others found.

3) searching for objects in template engine used  that can be helpful for information disclosure:
 - Explore the environment variables and try to discover all the objects to which you have access.
 - look for Developer-supplied objects since it may contain sensitive information or exploitable methods.

#### NOTE 
**for Burp Suite Professional users, the Intruder provides a built-in wordlist for brute-forcing variable names.**

## Create a custom attack

sometimes you will need to construct a custom exploit. For example, you might find that the template engine executes templates inside a sandbox, which can make exploitation difficult, or even impossible.

After identifying the attack surface, if there is no obvious way to exploit the vulnerability, the first step is to identify objects and methods to which you have access. 
then make a shortlist of objects that you want to investigate more thoroughly.

then you can discover combinations of objects and methods that you can chain together. Chaining together the right objects and methods sometimes allows you to gain access to dangerous functionality and sensitive data that initially appears out of reach.





