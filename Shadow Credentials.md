# **Kerberos Authentication in Active Directory**

Active Directory uses the Kerberos protocol to securely authenticate users and services without transmitting passwords over the network. It relies on ticket-based authentication.

#### **Traditional Kerberos Flow (Symmetric Encryption)**

1. **AS-REQ**: Client sends a timestamp encrypted with a key derived from the user's password hash.
2. **AS-REP**: KDC validates the timestamp using its copy of the user's hash and returns:
    - TGT encrypted with the KDC's secret key (krbtgt)
    - Session key encrypted with the user's password hash
3. **TGS Exchange**: Client uses the TGT to request service tickets.


## **Problems with Traditional Kerberos

Traditional Kerberos authentication relies on **symmetric key cryptography** (where the client and server share a secret key derived from the user’s password)

Passwords have a number of vulnerabilities, including:

- the possibility of using dictionary passwords;
- the need to remember passwords;
- the risk of users accidentally disclosing passwords in publicly accessible files;
- in case of server compromise, an attacker can obtain users’ passwords and use them for replay attacks.

## **PKINIT**

PKINIT (Public Key Cryptography for Initial Authentication in Kerberos) is an extension to the Kerberos protocol that allows the use of public key (asymmetric) cryptography at the initial authentication stage (AS-REQ and AS-REP).

PKINIT introduces the **msDS-KeyCredentialLink** attribute in Windows Server 2016 to store public keys for authentication. This attribute is crucial for certificate-based authentication, and here are its key details:

1. The **msDS-KeyCredentialLink** is a multi-value attribute, meaning multiple public keys can be stored for a single account, often representing different devices linked to that account.

2. Each value in this attribute contains serialized objects called Key Credentials. These objects include:
	- Creation date
	- Distinguished name of the owner
	- A GUID representing a Device ID
	- The public key itself

3. During PKINIT authentication, the client’s public key is verified against the values stored in this attribute. If a match is found, the KDC proceeds with authentication.

4. _Therefore_, managing and modifying the **msDS-KeyCredentialLink** attribute is an action that requires specific permissions, typically held by accounts that are members of highly privileged groups, These groups include:

	**Key Admins:** Members of this group can perform administrative actions on key objects within the domain. The Key Admins group applies to the Windows Server operating system in Default Active Directory security groups.
	
	**Enterprise Key Admins:** Members of this group can perform administrative actions on key objects within the forest.
	
	**Domain Admins:** Members of this group have almost all the privileges within a domain, including the ability to modify attributes.

5. It is important to note that user objects can’t edit their own **msDS-KeyCredentialLink** attribute, while computer objects can.  On the other hand, computer objects can edit their own **msDS-KeyCredentialLink** attribute but can only add a KeyCredential if none already exists.

Here’s how it works:

1. **AS-REQ with PKINIT:** The client sends a request to the KDC, including a timestamp signed with the client’s private key and the corresponding public key (X.509 Certificate).
2. **Public Key Validation:** The KDC checks the client’s public key against the **msDS-KeyCredentialLink** attribute in Active Directory. Means, instead of directly using the certificate for authentication, the KDC is validating if any of the public keys in the **msDS-KeyCredentialLink** attribute of the user matches the one used in the AS-REQ. If the key is valid, the KDC verifies the signature of the timestamp using public key.
3. **AS-REP:** If validation is successful, the KDC issues a TGT to the client + a session key encrypted with the public key of the client.

## How Shadow Credentials Work

The Shadow Credentials attack takes advantage of improper permissions on the **msDS-KeyCredentialLink** attribute, allowing attackers to inject their own public key into the attribute of a target user or computer account. Once this is done, they can impersonate the target account using PKINIT.

**Step 1: Identify Target Permissions**

The attacker identifies an Active Directory object such as a user or computer account, where they have permissions to modify attributes. Permissions like **GenericWrite** or **GenericAll** are required to modify the **msDS-KeyCredentialLink** attribute.

**Step 2: Inject the Attacker’s Public Key**

Next, the attacker adds their own public key to the **msDS-KeyCredentialLink** attribute of the target account. This process essentially “registers” the attacker’s key as a valid authentication method for the target.

**Step 3: Generate a Certificate**

The attacker creates a certificate in PFX format using the private key associated with the injected public key. This certificate is now tied to the target account.

**Step 4: Authenticate as the Target Account**

With the generated certificate, the attacker authenticates to the domain using PKINIT. The KDC validates the attacker’s public key against the **msDS-KeyCredentialLink** attribute and issues a Ticket Granting Ticket (TGT) for the target account.

## Abuse

The attacker needs permissions like **GenericWrite** or **GenericAll**  over an user/computer account to modify the **msDS-KeyCredentialLink** attribute

1. User mike has GenericWrite access on alice and alice is a member of administrator group


![image info](./assets/Pasted%20image%2020250721180129.png)

2. We can add our public key on **msDS-KeyCredentialLink** attribute of alice user Object and use PKINIT to get a TGT for alice.

![image info](./assets/Pasted%20image%2020250721174027.png)

3. The TGT of alice is saved in alice.ccache file and we also got the NTLM hash of alice.
4. Using alice TGT to get shell on DC01.

![image info](./assets/Pasted%20image%2020250721180400.png)




