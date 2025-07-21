
### **Double Hop Problem**

The "double hop problem" refers to a specific issue that arises in networked environments, particularly those using Windows authentication protocols like Kerberos or NTLM. It occurs when a user's credentials need to be passed from the client to a first server (hop 1) and then from that first server to a second server (hop 2), but the credentials cannot be delegated beyond the first server.

Imagine a web application where:

- A user (bob) logs in to a web portal (Server A).
- The web portal tries to fetch data from a SQL database (Server B) on the user's behalf.

Server A can't authenticate to Server B as the user (bob), resulting in the double hop problem.

### **Kerberos Delegation**

Delegation is the act of giving someone authority or responsibility to do something on behalf of someone else. A similar concept is applied in the Active Directory environment; delegation allows an account with the delegate property to impersonate another account to access resources within the network.

Kerberos delegation is a feature in **Active Directory (AD)** that allows a service to impersonate a user's identity to access other network resources on their behalf. (e.g., web servers accessing databases) where the initial service must act as the user when interacting with a backend service.
#### **Types of Kerberos Delegation

1. Unconstrained Delegation
2. Constrained Delegation
3. Resource-Based Constrained Delegation

### **UnConstrained Delegation**

- UnConstrained kerberos delegation is a privilege that can be assigned to a domain computer or a user

![[Pasted image 20250714170309.png]]

- When Kerberos Unconstrained Delegation is enabled on a server, the Domain Controller places a copy of the user’s TGT into the service ticket of that service. 

- When the user’s service ticket (TGS) is provided to the server for service access, the server opens the TGS and places the user’s TGT into LSASS for later use. 

- The reason TGTs get cached in memory is so the service computer (with delegation rights) can impersonate the authenticated user as and when required for accessing any other services on that user's behalf.

![[Pasted image 20250714170217.png]]


![[Pasted image 20250709090243.png]]


1. User Send AS REQ to get a TGT from DC.
2. User Gets the TGT from DC.
3. User Present TGT to get a ST from DC.
4. DC put the user's TGT in ST and send it to the user (Because the Application Server is Configured with UnConstrained Delegation, DC checks for TRUSTED_FOR_DELEGATION in UAC attribute)
5. User Uses the ST to access Application Server
6. Application Server Extracts the TGT of user from ST and uses the TGT to get a ST on behalf of the user for any other service (lets say database service)
7. DC verifies the TGT of user and send a ST for database service back to the Application server
8. Now Application Server can use this ST to access user's data from database Server.
#### Identifying Targets for Abuse

Identifying systems that are configured for Unconstrained Delegation is easy with PowerView.

![[Pasted image 20250721182126.png]]

- Any computer/service account that contains the TRUSTED_FOR_DELEGATION value true in its UserAccountControl (UAC) attribute is a viable target.
#### Exploitation 

1. Compromise the Server via an local admin, service account or computer account.
2. Trick a Domain Admin to connect to any service on the server with unconstrained delegation.
3. Then you can extract the domain admin TGT from LSASS.


