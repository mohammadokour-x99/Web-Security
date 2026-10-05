# JSON WEB TOKENS ATTACKS (JWT)

## What are JWTs?
JSON web tokens (JWTs) are a standardized format for sending cryptographically signed JSON data between systems.
used to send information ("claims") about users as part of authentication, session handling, and access control mechanisms.

### JWT format
A JWT consists of 3 parts: a header, a payload, and a signature.These are each separated by a dot.

![](images/Pasted%20image%2020261003032938.png)

The header and payload parts of a JWT are just base64url-encoded JSON objects. The header contains metadata about the token itself, while the payload contains the actual "claims" about the user.

after bas64 url decoding  the payload :
![](images/Pasted%20image%2020261003033054.png)

### JWT signature
The result of hashing the header and payload/encrypt the resulting hash. this process involves a secret signing key.

- As the signature is directly derived from the rest of the token, changing a single byte of the header or payload results in a mismatched signature.

- Without knowing the server's secret signing key, it shouldn't be possible to generate the correct signature for a given header or payload.

`https://www.jwt.io/`

### JWT vs JWS vs JWE

JWT defines a format for representing information ("claims") as a JSON object that can be transferred between two parties.

The JWT spec is extended by both the JSON Web Signature (JWS) and JSON Web Encryption (JWE) specifications.
![](images/Pasted%20image%2020261003033725.png)

a JWT is usually either a JWS or JWE token, but the most commonly used is JWS

JWEs are very similar, except that the actual contents of the token are encrypted rather than just encoded.


## What are JWT attacks?
a user sending modified JWTs to the server in order to achieve a malicious goal. Typically, this goal is to bypass authentication and access controls by impersonating another user who has already been authenticated.

## IMPACT
attacker able to escalate their own privileges or impersonate other users.


## How do vulnerabilities to JWT attacks arise?
flawed JWT handling within the application itself.
These implementation flaws usually mean that the signature of the JWT is not verified properly.
enabling an attacker to tamper with the values passed to the application via the token's payload. Even if the signature is robustly verified.


## How to work with JWTs in Burp Suite using JWT Extension

`https://portswigger.net/burp/documentation/desktop/testing-workflow/vulnerabilities/session-management/jwts`


## Exploiting flawed JWT signature verification

By design, servers don't usually store any information about the JWTs that they issue. Instead, each token is an entirely self-contained entity.

the server doesn't actually know anything about the original contents of the token, or even what the original signature was. Therefore, if the server doesn't verify the signature properly, there's nothing to stop an attacker from making arbitrary changes to the rest of the token.



# Lab: JWT authentication bypass via unverified signature

login using wiener:peter account

*NOTICE that we are using JWT editor extension so go and install it.*
go and discover the website.
then go back to burp proxy's history 

![](images/Pasted%20image%2020261003040851.png)
it showing that there is a JWT token.
now send it to burp repeater.

![](images/Pasted%20image%2020261003041207.png)

highlighting the payload part of JWT , in Inspector u'll see that payload got decoded.

```
{"iss":"portswigger","exp":1790992912,"sub":"wiener"}
```

now let's see if this site not validates the signature.
go to JSON Web Token Tab, then change sub with carlos user then click copy 

![](images/Pasted%20image%2020261003042356.png)


now go to my-account page and intercept the request, then replace session with the copied JWT and change id to carlos

![](images/Pasted%20image%2020261003042255.png)


![](images/Pasted%20image%2020261003042319.png)



now to solve the lab we need to access admin panel and delete carlos user

![](images/Pasted%20image%2020261003042810.png)



![](images/Pasted%20image%2020261003042620.png)


![](images/Pasted%20image%2020261003042650.png)


![](images/Pasted%20image%2020261003042726.png)



### Accepting tokens with no signature

 the JWT header contains an `alg` parameter. This tells the server which algorithm was used to sign the token and, therefore, which algorithm it needs to use when verifying the signature.

`{ "alg": "HS256", "typ": "JWT" }`


![](images/Pasted%20image%2020261003064502.png)



# Lab: JWT authentication bypass via flawed signature verification

login using wiener:peter account.
go and discover the website.
then go back to burp proxy's history 


![](images/Pasted%20image%2020261003062801.png)

![](images/Pasted%20image%2020261003062821.png)
highlighting the payload part of JWT , in Inspector u'll see that payload got decoded.


![](images/Pasted%20image%2020261003063434.png)
now turn intercept on and visit /admin  page , then send the request to Burp repeater.
on Json Web Token Tab , change alg : "non" , sub: "administrator" , then click copy 



![](images/Pasted%20image%2020261003063509.png)
now past what u copied but delete signature part , don't delete the dot placed after payload part


![](images/Pasted%20image%2020261003063525.png)


![](images/Pasted%20image%2020261003063612.png)




## Brute-forcing secret keys using hashcat

When implementing JWT applications, developers sometimes make mistakes like forgetting to change default or placeholder secrets. They may even copy and paste code snippets they find online, then forget to change a hardcoded secret that's provided as an example. In this case, it can be trivial for an attacker to brute-force a server's secret using a [wordlist of well-known secrets](https://github.com/wallarm/jwt-secrets/blob/master/jwt.secrets.list).

Hashcat will:

1. Take the JWT header and payload.
2. Take the first word from the wordlist.
3. Recalculate the JWT signature using that word as the secret.
4. Compare the result to the signature in the token.
5. If they match, the word is the secret key.

```
<jwt>:secret123
```





# Lab: JWT authentication bypass via weak signing key

login using wiener:peter account 
discover the website , then go back burp proxy's history.



![](images/Pasted%20image%2020261003062801.png)

visit and intercept /admin  page . then send it to burp repeater


![](images/Pasted%20image%2020261003070528.png)

notice that the algorithm used is `HS256`


we are gonna use hash cat for brute forcing the secret key.
download this wordlist:
```
https://github.com/wallarm/jwt-secrets/blob/master/jwt.secrets.list
```

use the following command :
`hashcat -a 0 -m 16500 <YOUR-JWT> /path/to/jwt.secrets.list`

![](images/Pasted%20image%2020261003071341.png)

![](images/Pasted%20image%2020261003071228.png)

now we know that the secret key used was `secret1`.


![](images/Pasted%20image%2020261003071554.png)
go to burp decoder and encode the key using base64. and copy it

![](images/Pasted%20image%2020261003071815.png)
go to JWT Editor tab then click on New `Symmetric Key` button. then click on generate.
change k with the base64 encoded key.


![](images/Pasted%20image%2020261003072000.png)
go to admin request then JSON Token Tab and change sub to administrator.
click sign then select the signing key that we've created. select `Don't modify header`

now copy the new JWT and send and intercept a request to admin panel with the new JWT  

![](images/Pasted%20image%2020261003072153.png)


![](images/Pasted%20image%2020261003072107.png)






### JWT Signing and Verification (RS256)

#### Signing

1. The company creates a private key and a public key.
2. When a user logs in, the server creates a JWT containing information about that user.
3. The server uses the **private key** to create a digital signature for the JWT.
4. The JWT is sent to the user.

#### Verification

1. The user sends the JWT back to the application.
2. The server receives the JWT and needs to check whether it was really created by the company and not modified.
3. The server obtains the corresponding **public key**.
4. The server uses the public key to verify the signature.
5. If the signature is valid, the JWT is trusted.
6. If the signature is invalid, the JWT is rejected.

As the names suggest, the private key must be kept secret, but the public key is often shared so that anybody can verify the signature of tokens issued by the server.



## JWT header parameter injections



- `jwk` (JSON Web Key) - provides the key/public key that server uses it to verify the JWT Signature.

- `jku` (JSON Web Key Set URL) - URL containing a set of public keys (JWKS)

- `kid` (Key ID) - Provides an ID that servers can use to identify the correct key in cases where there are multiple keys to choose from. Depending on the format of the key, this may have a matching `kid` parameter.


These user-controllable parameters each tell the recipient server which key to use when verifying the signature.


Examples :

```
{
  "alg": "RS256",
  "jku": "https://company.com/jwks.json",
  "kid": "key1"
}
```

```
{
  "keys": [
    {
      "kid": "key1",
      "kty": "RSA",
      "n": "...",
      "e": "AQAB"
   }
  ]
}
```

The server finds the key whose `kid` is `key1` and verifies the signature.



```
{
  "alg": "RS256",
  "typ": "JWT",
  "jwk": {
    "kty": "RSA",
    "kid": "key1",
    "n": "sXchh1N7j6z0H5iJjzL5Q8uY6m3dY6nYk9q7vR9...",
    "e": "AQAB"
  }
}
```

The server will  use that embedded public key to verify the signature.


so here  an attacker can create a  new RSA key (private and public keys) then use it to sign the modified JWT and embedding the public key into JWK header parameter.

JWK injection can happen if the server trust jwk header and not validates whether  it comes from a trusted source. 

u can do JWK injection attack either by using JWT Editor extension or add JWK header manually but you may also need to update the JWT's `kid` header parameter to match the `kid` of the embedded key. The extension's built-in attack takes care of this step for you.

JWT Editor JWK injection steps :
![](images/Pasted%20image%2020261004085143.png)



# Lab: JWT authentication bypass via jwk header injection

login using wiener:peter account 
discover the website , then go back burp proxy's history.


![](images/Pasted%20image%2020261004075945.png)


![](images/Pasted%20image%2020261004080018.png)
send /admin request to burp repeater, and study the decoded JWT payload part.

![](images/Pasted%20image%2020261004080246.png)

Notice that the algorithm used was RS256.


![](images/Pasted%20image%2020261004080821.png)
change sub to `administrator` 

now we wanna create a new RSA Key and use it to sign the modified JWT with its private key and inject its public key into JWK header parameter.

![](images/Pasted%20image%2020261004080422.png)
go to JWT Editor, click on New RSA Key -> click Generate then click ok to save it.  




![](images/Pasted%20image%2020261004080609.png)

![](images/Pasted%20image%2020261004080639.png)

Go back to /admin request on burp repeater then click Attack -> choose Embedded JWK.
Embedded JWK Attack window will pop up -> select our generated key and choose the algorithm used which is RS256.


![](images/Pasted%20image%2020261004080751.png)![](images/Pasted%20image%2020261004080803.png)

notice that the modified  JWT signed with generated key's private key , a JWK header parameter injected having generated key's public key.


![](images/Pasted%20image%2020261004080955.png)
copy the new JWT , send a request to /admin ,intercept it and past the copied JWT 

![](images/Pasted%20image%2020261004081340.png)

![](images/Pasted%20image%2020261004081403.png)





# Lab: JWT authentication bypass via jku header injection

![](images/Pasted%20image%2020261005091727.png)

![](images/Pasted%20image%2020261005091742.png)

login using wiener:peter account.

![](images/Pasted%20image%2020261004093948.png)
discover the website.


![](images/Pasted%20image%2020261004094040.png)
send /admin request to burp repeater.
study the decoded JWT payload part.


![](images/Pasted%20image%2020261004094052.png)
go to JSON Web Token Tab. notice that the algorithm used was RS256



![](images/Pasted%20image%2020261004094524.png)
change sub to administrator



now we wanna do JKU header parameter injection attack :

1) creating new RSA Key (private and public keys)
2)  creating a json file on exploit server having the JWK Set of public keys 
3)  changing kid value with the generated RSA key , adding jku header into JWT 
4)  signing the JWT using private key
5) copying the new modified JWT and use it to authentication bypass.




![](images/Pasted%20image%2020261004094242.png)
copy kid value , we need it later !!



![](images/Pasted%20image%2020261004094909.png)


go to exploit server and replace the content of the body section with  the Empty JWK Set.  

```
{

    "keys":[

  

    ]

}
```

past the copied RSA public key: 

![](images/Pasted%20image%2020261004095228.png)

store the code on exploit server.


![](images/Pasted%20image%2020261004095350.png)

- replace  kid value with  the kid value if the  new RSA key . because we want server to choose our RSA Public key
- add jku header parameter and set the value :  The url of exploit server /exploit
- click sign then select the signing key, ensure that alg used RS256 and Header options sat  `Don't modify header`

copy the modified JWT.


![](images/Pasted%20image%2020261004095431.png)
send a request to /admin and replace session content with the  copied JWT. 

![](images/Pasted%20image%2020261004095454.png)


![](images/Pasted%20image%2020261004095532.png)




### Injecting self-signed JWTs via the kid parameter

Sometimes developers use the `kid` parameter to point to a particular entry in a database, or even the name of a file.

For example, they might point to the file path for the key used for verification.

So if an attacker can do a file path traversal, he can point to an arbitrary file on the server's filesystem and make the server use its contents as the verification key.

This is especially dangerous with **symmetric algorithms like HS256**, because the same secret is used for both signing and verification.

For example, an attacker can point the `kid` parameter to `/dev/null`. Since `/dev/null` is an empty file, the server reads an empty string and uses it as the verification secret.

The attacker can then sign their own JWT using an empty string as the secret. Since the server also uses the empty string to verify the token, the signature is valid and the server may accept the attacker's JWT.

With asymmetric algorithms like **RS256**, this specific `/dev/null` technique does not work because the server expects a valid public key, not an empty string.

However, an asymmetric `kid` vulnerability **can still be exploitable** if you can make the server load a file containing a **public key that you control/know**, while you possess the corresponding private key.

If the server stores its verification keys in a database, the `kid` header parameter is also a potential vector for SQL injection attacks.

# Lab: JWT authentication bypass via kid header path traversal


login using wiener:peter account.

![](images/Pasted%20image%2020261004110825.png)
discover the website


![](images/Pasted%20image%2020261004110848.png)
send /admin request to repeater then study the decoded JWT payload part



![](images/Pasted%20image%2020261004110935.png)
in JSON Web Token Tab, notice that algorithm used was HS256 (Symmetric Encryption- single key)
![](images/Pasted%20image%2020261004110908.png)
change sub to administrator

in this lab : 
```
this website uses kid parameter to select the file path of the key used for verification.
so what we are going to do is changing the file path to /dev/null file which returns null / "" empty string using file path traversal.

so  key = ""

then we will sign the modified JWT with a empty key.
```


![](images/Pasted%20image%2020261004111711.png)


![](images/Pasted%20image%2020261004111411.png)
- changing kid value to /dev/null using file path traversal 
- change sub to administrator

![](images/Pasted%20image%2020261004111441.png)
signing our modified JWT with empty key.


![](images/Pasted%20image%2020261004111539.png)
- copy JWT then 
- send a request to /admin but  replace the session content with the copied JWT
![](images/Pasted%20image%2020261004111556.png)

![](images/Pasted%20image%2020261004111625.png)


another solution : 
![](images/Pasted%20image%2020261004112135.png)







### Other interesting JWT header parameters

The following header parameters may also be interesting for attackers:

- `cty` (Content Type) - Sometimes used to declare a media type for the content in the JWT payload. This is usually omitted from the header, but the underlying parsing library may support it anyway. If you have found a way to bypass signature verification, you can try injecting a `cty` header to change the content type to `text/xml` or `application/x-java-serialized-object`, which can potentially enable new vectors for [XXE](https://portswigger.net/web-security/xxe) and [deserialization](https://portswigger.net/web-security/deserialization) attacks.
    
- `x5c` (X.509 Certificate Chain) - Sometimes used to pass the X.509 public key certificate or certificate chain of the key used to digitally sign the JWT. This header parameter can be used to inject self-signed certificates, similar to the [`jwk` header injection](https://portswigger.net/web-security/jwt#injecting-self-signed-jwts-via-the-jwk-parameter) attacks discussed above. Due to the complexity of the X.509 format and its extensions, parsing these certificates can also introduce vulnerabilities. Details of these attacks are beyond the scope of these materials, but for more details, check out [CVE-2017-2800](https://talosintelligence.com/vulnerability_reports/TALOS-2017-0293) and [CVE-2018-2633](https://mbechler.github.io/2018/01/20/Java-CVE-2018-2633).



  

## JWT Algorithm Confusion (Key Confusion) Attack

### Definition

Algorithm confusion attacks (also called key confusion attacks) occur when a server verifies a JWT using a different algorithm than the developers intended.

This can allow an attacker to forge valid JWTs with arbitrary claims without knowing the server's private signing key.

---

## Why does this happen?

Many JWT libraries provide a generic verification function such as:

```
function verify(token, secretOrPublicKey) {
    algorithm = token.getAlgHeader();

    if (algorithm == "RS256") {
        // Use provided key as RSA public key
    }
    else if (algorithm == "HS256") {
        // Use provided key as HMAC secret
    }
}
```

The library determines how to verify the signature based on the `alg` header inside the JWT.

A developer may assume the application only uses RS256 and write code like:

```
publicKey = <server_public_key>;

token = request.getCookie("session");

verify(token, publicKey);
```

The problem is that the verification method trusts the `alg` value supplied by the JWT.

If an attacker changes:

```
{
  "alg": "RS256"
}
```

to:

```
{
  "alg": "HS256"
}
```

the library will treat the provided public key as an HMAC secret instead of an RSA public key.

As a result:

- The attacker signs the JWT using HS256 and the server's public key.
    
- The server verifies the JWT using HS256 and the same public key.
    
- The signature is accepted.
    

---

## Why does the attack work?

The application passes:

```
verify(token, publicKey);
```

The attacker controls:

```
{
  "alg": "HS256"
}
```

The library interprets:

```
publicKey -> HMAC secret
```

instead of:

```
publicKey -> RSA public key
```

Since both attacker and server use the same public key as the HS256 secret, the signature becomes valid.

---

## Important Note

The public key used by the attacker must be identical to the server's copy.

This includes:

- Same format (PEM, X.509, etc.)
    
- Same encoding
    
- Same spaces
    
- Same newlines
    
- Same non-printing characters
    

Even a small formatting difference changes the HMAC input and results in a different signature.

---

# Attack Process

## Step 1 - Obtain the server's public key

Public keys are often exposed through:

```
/jwks.json
```

or

```
/.well-known/jwks.json
```

Example JWK Set:

```
{
  "keys": [
    {
      "kty": "RSA",
      "e": "AQAB",
      "kid": "example",
      "n": "..."
    }
  ]
}
```

Where:

```
n = RSA modulus
e = RSA public exponent
```

Together they define the RSA public key.

If the key is not publicly exposed, it may sometimes be derived from existing JWTs.

---

## Step 2 - Convert the public key to the correct format

The public key exposed in a JWK Set may not match the format used internally by the server.

For the attack to work:

```
Attacker's key bytes == Server's key bytes
```

The key often needs to be converted into PEM/X.509 format.

Burp's JWT Editor extension can be used to:

1. Import the JWK.
    
2. Convert it to PEM format.
    
3. Base64-encode the PEM.
    
4. Create a symmetric key using the encoded PEM value.
    

The goal is to make the HS256 secret identical to the server's public key.

---

## Step 3 - Modify the JWT

Modify any claims you want.

Example:

```
{
  "sub": "administrator"
}
```

Change the JWT header to:

```
{
  "alg": "HS256"
}
```

instead of:

```
{
  "alg": "RS256"
}
```

---

## Step 4 - Sign the JWT

Sign the modified token using:

```
Algorithm: HS256
Secret: Server Public Key
```

Instead of:

```
RS256(private_key)
```

the attacker creates:

```
HS256(public_key)
```

---

## Summary

```
1. Obtain server public key
          |
          v
2. Convert key to same format used by server
          |
          v
3. Change alg from RS256 to HS256
          |
          v
4. Modify JWT payload
          |
          v
5. Sign JWT using HS256 with the public key as the secret
          |
          v
6. Server treats public key as HMAC secret
          |
          v
7. Signature is valid
          |
          v
8. Forged JWT accepted
```

#### Root Cause

The server trusts the attacker's `alg` header instead of enforcing the expected algorithm.

# Lab: JWT authentication bypass via algorithm confusion


login using wiener:peter account.

![](images/Pasted%20image%2020261004141404.png)
discover the webiste.

![](images/Pasted%20image%2020261004141428.png)
send /admin request to burp repeater. 
study the decoded JWT payload part.


![](images/Pasted%20image%2020261004141514.png)
go to JSON Web Token Tab.
algorithm used was RS256 (private and public keys).


now in this lab we're going to make algorithm confusion attack:

1) obtain RSA Public Key   /jwks.json   , /.well-known/jwks.json
2) convert the public key format  to another format that match the format of the public key stored on server. here suppose the server has public key in PEM format and then encoded it using base64 encoding 
3) creating a new symmetric key using RSA Public Key as a secret key
4) modifying JWT payload and set `alg` header parameter to `HS256`
5) signing JWT using the created symmetric key
6) send the request to admin page and delete carlos user



![](images/Pasted%20image%2020261004141630.png)

![](images/Pasted%20image%2020261004141745.png)
the server will use this public key because kid's public key matches kid's JWT header parameter.


![](images/Pasted%20image%2020261004142109.png)

add the public key. 



![](images/Pasted%20image%2020261004142141.png)
convert it to PEM format


![](images/Pasted%20image%2020261004142303.png)
encode Public Key PEM format as base64. copy the output


![](images/Pasted%20image%2020261004142440.png)
creating a new symmetric key:
click on `New symmetric key` button -> `generate` .
replace  `k` value with the output of encoding Public Key PEM format as base64.


![](images/Pasted%20image%2020261004142606.png)
change `alg` header parameter to HS256
change sub to administrator


![](images/Pasted%20image%2020261004142742.png)

click on sign  button then select the signing key that we created.
copy the Serialized JWT 



![](images/Pasted%20image%2020261004142847.png)

send a request to /admin page and intercept it.
replace session content with JWT 

![](images/Pasted%20image%2020261004142812.png)

![](images/Pasted%20image%2020261004142932.png)


# jwt_forgery.py / sig2n

`jwt_forgery.py` and PortSwigger's `sig2n` are tools used when testing **JWT algorithm confusion attacks with RSA (RS256)** and the server's public key is not directly available.

The main idea is that if we have **two valid JWTs signed with the same RSA private key**, the tools can use their signatures to calculate possible values of the RSA modulus **n**.

### What the tool does

1. Take two valid RS256 JWTs as input.
    
2. Analyze their signatures and use RSA mathematics to calculate one or more possible values of **n**.
    
3. Use each possible `n` to construct a candidate RSA **public key**.
    
4. For each candidate public key, the tool generates a **test JWT**.
    
    These are not normally signed using the RSA private key. They are specially constructed using information from the existing JWTs and RSA mathematics.
    
5. The tool outputs the candidate keys and their corresponding test JWTs.
    

### Finding the correct public key

Send each generated test JWT to the target application using Burp Repeater.

```
Candidate key 1 → Test JWT 1 → Rejected
Candidate key 2 → Test JWT 2 → Rejected
Candidate key 3 → Test JWT 3 → Accepted
```

The candidate whose test JWT is accepted corresponds to the **correct RSA public key** used by the server.

### After finding the public key

The recovered public key can then be used to perform an **algorithm confusion attack**:

```
RS256
  ↓
Change algorithm to HS256
  ↓
Use the RSA public key as the HMAC secret
  ↓
Sign a modified JWT
  ↓
Send it to the server
```

If the server incorrectly allows the algorithm to be changed from RS256 to HS256 and uses the same key material for verification, the forged JWT may be accepted.

### sig2n command

```
docker run --rm -it portswigger/sig2n <token1> <token2>
```

`sig2n` essentially automates the process of:

```
2 valid JWTs
     ↓
RSA signature analysis
     ↓
Possible n values
     ↓
Candidate public keys
     ↓
Test JWTs
     ↓
Find the correct public key
```

**Important:** The tool does NOT recover the RSA private key. It is used to recover/identify the **public key** needed for the algorithm confusion attack.






# Lab: JWT authentication bypass via algorithm confusion with no exposed key

before read this lab's solution , read above about  "Deriving public keys from existing tokens"  and `jwt_forgery.py / sig2n`  tools and what they do. 

```
https://github.com/silentsignal/rsa_sign2n/
```

```
docker run --rm -it portswigger/sig2n <token1> <token2>
```

login using wienre:peter account


![](images/Pasted%20image%2020261005080046.png)
discover the website


![](images/Pasted%20image%2020261005080114.png)
send /admin request to repeater.
study the JWT payload part.


now we wanna do algorithm confusion attack.
let's do the same steps we did on previous lab. 
but here notice that we didn't find the public key on /jwks.json    /.well-known/jwks.json.

so let's use a tool called jwt_forgery.py  which derive the public key from two existing JWTs .




![](images/Pasted%20image%2020261005080502.png)

we want two JWTs:

1)  copy the JWT session 
2) logout then Re-Login 
3) copy the JWT session


![](images/Pasted%20image%2020261005080946.png)
ensure that `docker-cli` installed 

![](images/Pasted%20image%2020261005085957.png)

![](images/Pasted%20image%2020261005083359.png)
now run the following command :
`docker run --rm -it portswigger/sig2n <token1> <token2>`


![](images/Pasted%20image%2020261005084018.png)


![](images/Pasted%20image%2020261005083706.png)

take the first Tampered JWT and send a request using it. 
after sending the request, if u are still logged in then this is the used public key. 

do the same process for the second Tampered JWT.



trying the second JWT :

![](images/Pasted%20image%2020261005083818.png)

so here the first public key  was the right one.


![](images/Pasted%20image%2020261005084200.png)
create a new symmetric key using the RSA public key

![](images/Pasted%20image%2020261005085827.png)

set alg header parameter to HS256
change sub to administrator.

now click on `sign` button then select the signing key then click ok


![](images/Pasted%20image%2020261005084315.png)
copy the JWT then send a request to /admin using it

![](images/Pasted%20image%2020261005084349.png)




![](images/Pasted%20image%2020261005090942.png)
