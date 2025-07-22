# Kerberos Authentication

Kerberos is a network authentication protocol that uses cryptographic tickets to verify user identities and grant access to services, It relies on a trusted third-party, the Key Distribution Center (KDC), to issue and manage these tickets, ensuring secure communication between clients and servers.
# How Kerberos Authentication Works

![image info](./assets/Pasted%20image%2020250722112011.png)

1. **AS REQ:** Client sends a AS REQ with a Timestamp Encrypted with key derived from client's password hash.
2. **AS REP:** KDC Decrypts the Timestamp to verify user identity, as KDC knows client's password, If timestamp gets decrypted KDC responds with a TGT encrypted with a key derived with krbtgt's account password hash + a session key encrypted with user's password hash.
3. **TGS REQ:** Now client can use this TGT + an Authenticator (Timestamp Encrypted with Session Key) to request a ST for a service.
4. **TGS REP:** Ticket Granting Server verifies the validity of TGT + checks the Authenticator, and it Responds with a Service Ticket + a new session key encrypted with old session key.
5. Client uses the Service Ticket to access the Service.

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

- padata: (Pre Auth Data) Current TImeStamp Encrypted with User's Password
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
            ciphertext: 7A3F9B44D21E... ()
            (Decrypted contents: session key, client principal, expiry, etc.)
    enc-part (Encrypted with user's key): aes256-cts-hmac-sha1-96
         ciphertext: 4E2F7C19B305...
        (Decrypted contents: session key, nonce, flags, etc.)
```

- ticket: TGT, encrypted with KDC's key
- enc-part: session key, encrypted with the user’s password, (we need this session key in TGS-REQ)

#### TGS REQ

```
Kerberos TGS-REQ
	pvno: 5
	msg-type: krb-tgs-req (12)
	padata: 1 item
	    PA-DATA
            padata-type: pA-TGS-REQ (1)
            padata-value: 6e82953d308290539a003920105
	req-body
	    Padding: 0
	    kdc-options: 40810010 (hex)
	    realm: DOMAIN.COM
	    sname:
            name-type: kRB5-NT-MS-PRINCIPAL (-128)
            sname-string:
               SNameString: DOMAIN.COM\web_srv
	    till: Jul 8, 2025 03:54:18.000000000 IST
	    nonce: 665478473
	    etype:
	        eTYPE-ARCFOUR-HMAC-MD5 (23) [RC4]
	        eTYPE-DES3-CBC-SHA1 (16)
	        eTYPE-DES-CBC-MD5 (3)
	        eTYPE-ARCFOUR-HMAC-MD5 (23) [RC4 - repeated]
```

- padata-value: TGT (encrypted with KDC secret key) + Authenticator ( timestamp and other data encrypted with session key)
- sname: service name
#### TGS REP

```
Kerberos TGS-REP
	pvno: 5
	msg-type: krb_tgs_rep 13
    crealm: DOMAIN.COM
	cname:
	    nametype: 1 (kRB5-NT-PRINCIPAL)
	    name: bob
	ticket
	    tkt-vno: 5
	    realm: DOMAIN.COM
	    sname:
	        nametype: -128 (kRB5-NT-MS-PRINCIPAL)
	        snamesting: DOMAIN.COM\web_svc
	    enc-part
	        etype: 23 (RC4 - ARCFOUR-HMAC-MD5)
	        kvno: 2
	        cipher: d5a12d0d96c40db1e1c923ac35a8bbb965ef275a8899a48fede4
	enc-type:
	    enctype: 23 (RC4 - ARCFOUR-HMAC-MD5)
	    cipher: a7e146fddf968e4b838ccae2687944c04fa3519d2fea8aeddff42

```

- cname: Client requesting the ticket.
- ticket: Service ticket for web_svc.
- ticket.enc-part: The ticket encrypted with the service's secret key (web_svc's password hash, only the service and KDC know this key).
- enc-type: here again session key, encrypted with the session key from TGT.

