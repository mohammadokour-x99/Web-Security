
# Lab: JWT authentication bypass via unverified signature


![](images/Pasted%20image%2020261003042810.png)

# Accepting tokens with no signature

![](images/Pasted%20image%2020261003063434.png)

![](images/Pasted%20image%2020261003063509.png)
now past what u copied but delete signature part , don't delete the dot placed after payload part


# Brute-forcing secret keys using hashcat  For HS256 algorithm


- download this wordlist:
```
https://github.com/wallarm/jwt-secrets/blob/master/jwt.secrets.list
```

- `hashcat -a 0 -m 16500 <YOUR-JWT> /path/to/jwt.secrets.list`

- encode the secret using base64 encoding

- generate New Symmetric key , then place k value with base64 encoded secret
- sign using the new Symmetric key


# Lab: JWT authentication bypass via jwk header injection

in this lab they used we are handling with RS256 (public and private keys) , but u can do it for HS256 (symmetric key)

- generate  New RSA Key
-  change sub to administrator 
-  click on `Attack` choose Embedded JWK (adding JWK header with New RSA public key info  , signing JWT using New RSA private key )


# Lab: JWT authentication bypass via jku header injection

- Generate a New RSA Key
- copying the New RSA public key
![](images/Pasted%20image%2020261004094909.png)

- on exploit server body section :

Add this :
```
{

    "keys":[

  

    ]

}
```

past the copied RSA public key: 

![](images/Pasted%20image%2020261004095228.png)


- changing sub to administrator
-  replace  kid value with  the kid value of the  new RSA key
- add JKU header parameter with value : link to exploit server
- signing JWT with NEW RSA Private Key


# Lab: JWT authentication bypass via kid header path traversal

If the server stores its verification keys in a database, the `kid` header parameter is also a potential vector for SQL injection attacks.

This attack can only work with HS256. 

However, an asymmetric `kid` vulnerability **can still be exploitable** if you can make the server load a file containing a **public key that you control/know**, while you possess the corresponding private key.

- change kid to  `../../../../../dev/null` since null file returns empty string
- change sub to administrator
- attack -> sign with empty key



### Other interesting JWT header parameters

- cty
- x5c



# Lab: JWT authentication bypass via algorithm confusion

 RS256 -> HS265

1) obtain RSA Public Key   /jwks.json   , /.well-known/jwks.json
2) convert the public key format  to another format that match the format of the public key stored on server. here suppose the server has public key in PEM format and then encoded it using base64 encoding 
3) creating a new symmetric key using RSA Public Key as a secret key
4) modifying JWT payload and set `alg` header parameter to `HS256`
5) signing JWT using the created symmetric key
6) send the request to admin page and delete carlos user

# Lab: JWT authentication bypass via algorithm confusion with no exposed key
- Obtain two valid JWT
1)  copy the JWT session 
2) logout then Re-Login 
3) copy the JWT session
![](images/Pasted%20image%2020261005080502.png)

- `docker run --rm -it portswigger/sig2n <token1> <token2>`
![](images/Pasted%20image%2020261005083359.png)
![](images/Pasted%20image%2020261005084018.png)

- try the two Tampered JWT , and take the base64 encoded Key that corresponds the Tampered JWT  that doesn't make u logs out from the account
  in other words:
  take the  Tampered JWT and send a request using it. 
  after sending the request, if u are still logged in then this is the used public key. 

 - creating a new symmetric key using RSA Public Key as a secret key
 - modifying JWT payload and set `alg` header parameter to `HS256`
 - signing JWT using the created symmetric key
 - send the request to admin page and delete carlos user

