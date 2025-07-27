# Active Directory Certificate Service

Microsoft defines Active Directory Certificate Services (AD CS) is a server role that allows you to build a public key infrastructure (PKI) and provide public key cryptography, digital certificates, and digital signature capabilities for your organization.

AD CS was introduced in **Windows 2000** and can be deployed in two primary configurations:

1. **Standalone CA** – Operates independently of Active Directory (AD), typically used in offline root CA scenarios.
2. **Enterprise CA** – Integrates with AD, enabling automated certificate enrollment and management for domain-joined systems.

## Digital certificates
Digital certificates  in **Active Directory Certificate Services (AD CS)** follow the **X.509 standard**, which defines the structure and fields of a certificate. These certificates are **digitally signed documents** used for encryption, Digital signatures (e.g., code signing, S/MIME), authentication (e.g., smart card logon, client/server auth).

### **Key Fields in an X.509 Certificate**

| **Field**                          | **Description**                                                                                                 |
| ---------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| **Subject**                        | The entity (user, device, or service) that owns the certificate. Example: `CN=User1, OU=IT, DC=contoso, DC=com` |
| **Public Key**                     | The public key bound to the subject (private key is stored securely by the owner).                              |
| **NotBefore / NotAfter**           | Validity period (certificates expire after `NotAfter`).                                                         |
| **Serial Number**                  | Unique identifier assigned by the CA (used in revocation checks).                                               |
| **Issuer**                         | The CA that issued the certificate (e.g., `CN=Contoso-CA, DC=contoso, DC=com`).                                 |
| **Subject Alternative Name (SAN)** | Alternate identities (e.g., DNS names, email, UPN).                                                             |
| **Basic Constraints**              | Determines if the certificate is a **CA** (can issue certs) or an **end-entity** (cannot issue certs).          |
| **Extended Key Usage (EKU)**       | Defines the certificate’s purpose via **OIDs** (e.g., Client Auth, Code Signing).                               |
| **Signature Algorithm**            | Algorithm used to sign the certificate (e.g., SHA256-RSA, ECDSA).                                               |
| **Signature**                      | The CA’s digital signature (proves authenticity).                                                               |
|                                    |                                       |

These are some EKUs:



| **EKU (OID)**                | **Friendly Name**                     | **Purpose**                                                                      |
| ---------------------------- | ------------------------------------- | -------------------------------------------------------------------------------- |
| **`1.3.6.1.4.1.311.20.2.1`** | `Certificate Request Agent`           | Allows signing CSRs for **Enrollment Agents** (most common).                     |
| **`1.3.6.1.4.1.311.21.19`**  | `Certificate Request Agent (KRA)`     | Used for **Key Recovery Agent** certificates (decrypting archived private keys). |
| **`1.3.6.1.4.1.311.21.20`**  | `Certificate Request Agent (Offline)` | For **offline request agents** (rare).                                           |
| **`1.3.6.1.5.5.7.3.2`**      | `Client Authentication`               | Sometimes required for **smartcard logon** CSRs.                                 |
| **`1.3.6.1.5.5.7.3.4`**      | `Secure Email`                        | Required for **S/MIME signing** requests.                                        |
| `1.3.6.1.4.1.311.20.2.2`     | `Smart Card Logon`                    | Required for Smart Card Logons                                                   |

## Certificate Template
A **certificate template** is a preconfigured blueprint in AD CS that defines the rules and settings for certificates issued by an **Enterprise CA**. It controls:

- **Certificate Properties**: Validity period, key length, encryption algorithms.
- **Extended Key Usages (EKUs)**: Purpose of the cert (e.g., Client Authentication, Code Signing).
- **Permissions**: Who can request/enroll (e.g., users, devices, security groups).


## Certificate Enrollment
To obtain a certificate from AD CS, clients go through a process called enrollment. 
a user can enroll for a certificate if it has necessary rights to do so, else Enterprise CA denied all enrollment request.
### Enrollment Rights
AD CS defines enrollment rights - which principals/users can request a certificate – using two security descriptors: one on the certificate template AD object and another on the Enterprise CA itself.
#### Certificate template
For certificate templates, the following Access Control Entries in a template’s DACL can result in a principal/user having enrollment rights:

1. **Explicit "Certificate-Enrollment" Extended Right**
    - **ACE Type**: `RIGHT_DS_CONTROL_ACCESS`
    - **Effect**: Grants the principal **manual enrollment** rights (e.g., via `certmgr.msc` or `certreq.exe`).
2. **Explicit "Certificate-AutoEnrollment" Extended Right**
    - **ACE Type**: `RIGHT_DS_CONTROL_ACCESS`
    - **Effect**: Grants the principal **auto-enrollment** rights (certificates are issued automatically via Group Policy).
3. **All Extended Rights (Wildcard Permissions)**
    - **ACE Type**: `RIGHT_DS_CONTROL_ACCESS`
    - **Effect**: Grants **all extended rights**, including both manual and auto-enrollment.
4. **FullControl / GenericAll (Total Permissions)**
    - **ACE Type**: `GENERIC_ALL` or `FULL_CONTROL`
    - **Effect**: Grants **full control** over the template, including enrollment rights (and modification/deletion rights).

![](../assets/Pasted%20image%2020250723001456.png)

####  Enterprise CA

The **Enterprise CA's security descriptor** acts as a **global gatekeeper**, overriding any enrollment rights defined in **certificate templates**. Even if a user has enrollment rights on a template, they **cannot** request a certificate if the CA's security descriptor **denies** them access.

**Where is the Enterprise CA Security Descriptor Configured?**
- **GUI Method**: Open **`certsrv.msc`** → Right-click the CA → **Properties** → **Security** tab.

![](../assets/Pasted%20image%2020250723000706.png)

- **Registry Storage**: Located at: `HKLM\SYSTEM\CurrentControlSet\Services\CertSvc\Configuration\<CA_NAME>\Security` contains a **binary blob** representing the security descriptor.

![](../assets/Pasted%20image%2020250723000844.png)

If both the Enterprise CA’s and the certificate template’s security descriptors grant the client certificate enrollment privileges, the client can then request a certificate.

So, what is happening behind the scenes when a user enrolls in a certificate? 
- In a basic scenario, a client first generates a public key and associated private key. The client creates a Certificate Signing Request (CSR) in which it specifies the public key and the name of the certificate template.
- The client then signs the CSR with the private key and sends the CSR to the Enterprise CA.
- The Enterprise CA then checks if the client has enrollment privileges at the CA level. If so, the CA looks at the certificate template specified in the CSR and verifies that the client can enroll in the given template by examining the certificate template AD object’s DACL.
- If the DACL grants the user the enrollment privileges, the user can enroll. 
- The CA will create and sign a certificate based on the certificate template’s settings and return the signed certificate to the user.

### Issuance Requirements

In addition to the certificate template and Enterprise CA access control restrictions, there are two certificate template settings we have seen used to control certificate enrollment. These are known as issuance requirements:

The first restriction is “**CA certificate manager approval**”, which puts all certificate requests based on the template into the pending state, which requires a CA certificate manager to approve or deny the request before the certificate is issued

The second set of restrictions shown in the issuance requirements are the settings “**This number of authorized signatures**” and the “**Application policy**”. 

- "This number of authorized signatures" controls the number of signatures required in the CSR for the CA to accept it. 
- "Application policy" defines the EKU OIDs that the CSR signing certificate must have. 

![](../assets/Pasted%20image%2020250727231925.png)

A Common use for these both settings is for enrollment agents. 

- Enrollment Agents are privileged entities users or computers (Think of them as Helpdesk users) authorized to **request certificates on behalf of other users**. 
- To do so, the CA must issue the enrollment agent user/computer account a certificate containing at least the Certificate Request Agent EKU (OID 1.3.6.1.4.1.311.20.2.1). 
- Once issued, the enrollment agent can then sign CSRs on behalf of other user and request certificates for other users. 

### Subject Alternative Names and Authentication

 **Subject Alternative Names (SANs) in Certificates:** SAN is an extension in X.509 certificates that allows additional identities (like DNS names, email addresses, IPs, or User Principal Names - UPNs) to be associated with the certificate.
 
 - In HTTPS Certificate, SANs enable a single certificate to secure multiple domains (e.g., `example.com`, `www.example.com`).
 - This is all well and good for HTTPS certificates, but when combined with certificates that allow for domain authentication, a dangerous scenario can arise. 
 - By default, during certificate-based authentication, one way AD maps certificates to user accounts based on a UPN specified in the SAN
 - If an attacker can specify an arbitrary SAN when requesting a certificate that has an EKU enabling client authentication, and the CA creates and signs a certificate using the attacker supplied SAN, the attacker can become any user in the domain. 
 - For example, if an attacker can request a client authentication certificate that has a domain administrator SAN field, and the CA issues the certificate without validating SAN, the attacker can authenticate as that domain admin.

Active Directory (AD) supports certificate-based authentication primarily through two protocols: **Kerberos (PKINIT)** and **Secure Channel (Schannel)**.

### Kerberos Authentication 

PKINIT is the Kerberos extension that enables certificate-based pre-authentication. The process works as follows:
- A user signs a Kerberos Ticket Granting Ticket (TGT) request with their **X.509 certificate** using their **private key** (associated with their certificate).
- The request is sent to a Domain Controller (DC).
- The DC performs several checks
- Validates the **certificate’s time validity** and issuing CA also validates the certificate includes the **Client Authentication** EKU or not.
- Confirms the **digital signature** using the user’s public key.        
- If all checks pass, the DC issues a TGT, allowing the user to access resources.

### Secure Channel (Schannel) Authentication

Schannel is Microsoft's implementation of the TLS/SSL protocols, Windows uses it when establishing TLS/SSL connections. Schannel also supports client authentication, enabling a remote server to verify the identity of the connecting user. It accomplishes this using PKI, with certificates being the primary credential. 

- During the TLS handshake, the server requests a certificate from the client for authentication. 
- The client, having previously been issued a client authentication certificate from a CA the server trusts, sends its certificate to the server. 
- The server then validates the certificate is correct and grants the user access assuming everything is okay. 
- By default, not many protocols in AD environments support AD authentication via Schannel out of the box. RDP and IIS support client authentication using Schannel, but it requires additional configuration.
- One protocol that does commonly work – assuming AD CS has been setup - is LDAPS (a.k.a., LDAP over SSL/TLS).



