We talked about Unconstrained Delegation Previously, which allows a computer or user to impersonate as any user on any service within the domain without restrictions. 

Today, we will discuss Constrained Delegation, which was introduced with Windows Server 2003 to address the issues of unconstrained delegation by providing administrators with more control over defining trust boundaries.

In constrained delegation, the impersonation of services is restricted to a specific list of services or service principal names (SPNs).

In essence, constrained delegation is a way to limit exactly what services a particular machine/account can access while impersonating other users.

also, Constrained delegation doesn't use TGT for delegation
 
So how did Microsoft implemented this delegation? Using service-for-user (S4U) Kerberos extensions.

The S4U (Service for User) Kerberos extensions enables a service to obtain a service ticket on behalf of a user, primarily for delegation purposes. 

There are two main types: 

S4U2Proxy (Service for User to Proxy) Kerberos Constrained Delegation extension
S4U2Self (Service for User to Self) Kerberos Protocol Transition extension

- **S4U2Proxy (Service for User to Proxy):** This enables a service to obtain a  service ticket to another service on behalf of a client user, all it needs is a evidence that the client is authenticated i.e. The Service Ticket of User.

- **S4U2Self (Service for User to Self):** This extension is meant for use in cases where a user authenticates to a service in a way not using Kerberos i.e. NTLM. This allows a service to obtain a forwardable service ticket to itself on behalf of a client user, without needing the user's credentials directly and use this forwardable service ticket as an evidence that client has authenticated to request a service ticket for another service using **S4U2Proxy.**

So there are two ways to configure constrained delegation: 

1. Kerberos only: When Client uses Kerberos (uses S4U2Proxy)
2. Protocol Transition: When Client uses non Kerberos (uses S4U2Self and S4U2Proxy)

### Without protocol transition (Kerberos Only) (S4U2Proxy)

When this type is configured on a Service, the `msDS-AllowedToDelegateTo` attribute is used to specify which services/SPNs the Service is allowed to Delegate to,  This attribute specifies the list of authorized SPN for the delegation.

![[Pasted image 20250721140809.png]]

![[Pasted image 20250721140534.png]]

![[Pasted image 20250714230059.png]]

1. Client authenticates with SERVER1 with its service ticket.
2. SERVER1 uses client's ST as an evidence with S4U2Proxy to request a ST on behalf of the client for SERVER2.
3. DC checks SERVER1's `msDS-AllowedToDelegateTo` attribute for SERVER2

	- if SERVER2 is in the  `msDS-AllowedToDelegateTo` attribute of SERVER1, it responds with the ST of client for SERVER2 
	- if not then the request is denied.
4. Now SERVER1 can use this ST of client to access SERVER2 on behalf of the client.


### With protocol transition (Any Auth Protocol) (S4U2Self and S4U2Proxy)

Now what if a user uses any non kerberos authentication mechanism i.e. NTLM or HTTP Auth.

Here also the `msDS-AllowedToDelegateTo` attribute is used to specify which services/SPNs the Service is allowed to Delegate to, This attribute specifies the list of authorized SPN for the delegation.

But S4U2Proxy requires an evidence service ticket to request a service ticket on behalf of a user for the SPNs listed in `msDS-AllowedToDelegateTo`

So Here comes S4U2Self extension to the rescue.

When we are configuring constrained delegation for any authentication protocol, a **TRUSTED_TO_AUTHENTICATE_FOR_DELEGATION** UserAccountControl flag is set on the service account.

If this flag is set, then the service with this flag can request a service ticket for itself on behalf of any user by invoking the S4U2Self extension.

Now we have an evidence service ticket for user eventhough the client authenticated using NTLM or HTTP Auth.

Now the service can use this evidence service ticket to get a service ticket for `cifs/WEB-SERVER-02` as any user.

![[Pasted image 20250721140417.png]]
![[Pasted image 20250721140534.png]]

![[Pasted image 20250714232907.png]]


1. Client authenticates using HTTP Auth with SERVER1
2. SERVER1 invokes S4U2Self to request a service ticket for itself for the client
3. DC checks SERVER1 for **TRUSTED_TO_AUTHENTICATE_FOR_DELEGATION** flag 
	- if this flag is set, DC responds with a forwardable ST for SERVER1 for the client
	- if not, DC responds with a non forwardable ST for SERVER1 for the client.

4. SERVER1 uses this forwardable ST as an evidence with S4U2Proxy to request a ST on behalf of the client for SERVER2.
5. DC checks SERVER1's `msDS-AllowedToDelegateTo` attribute for SERVER2

	- if found and the ST is Forwardable then it responds with the ST of client for SERVER2 
	- if not then the request is denied.

6. Now SERVER1 can use this ST of client to access SERVER2 on behalf of the client.
### Abuse
Compromise an account configured with constrained delegation 

- With protocol transition: if an account is configured with constrained delegation with protocol transition, we can invoke S4U2Self to get a forwardable Service ticket for any user and use it with the S4U2Proxy to get a Service Ticket for any service listed in `msDS-AllowedToDelegateTo`

- Without Protocol Transition:  if account is configured with constrained delegation with kerberos only (Without Protocol Transition), we can try tricking any high Privilege user to authenticate with our service, and when user authenticates, we can use his/her service ticket with S4U2Proxy to get a Service Ticket for any service listed in `msDS-AllowedToDelegateTo`

#### Exploiting KCD with Protocol Transition

1. Enumerating using impacket: sqlsrv can delegate on cifs/DC01

```
findDelegation.py infra.local/sqlsrv:'d0il0ve$0me0ne'
```
![[Pasted image 20250721134249.png]]

2. Exploiting using impacket: using impacket to impersonate administrator on cifs/DC01 by invoking S4U2Self and S4U2Proxy and geting a Service Ticket for administrator for cifs/DC01.

![[Pasted image 20250721134215.png]]

3. Importing the administrator ticket and getting shell as admin.
![[Pasted image 20250721165941.png]]

