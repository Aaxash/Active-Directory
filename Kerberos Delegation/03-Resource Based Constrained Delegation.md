
# Resource Based Constrained Delegation

**Resource Based Constrained Delegation** (RBCD), Introduced with Windows Server 2012, this solution allows to overcome some problems related to the **Constrained Delegation**

- In RBCD the delegation responsibility is moved. Whereas in **Constrained Delegation**, it’s the relaying server that holds the list of allowed target services, 

- In the case of **Resource Based Constrained Delegation**, it’s the resources (or services) that have a list of accounts they trust for delegation.

In RBCD , the trust is configured on the service that receives delegated credentials 

This time, the Domain Controller will look at the attribute of Service2

It will check that the account of Service 1 is present in the `msDS-AllowedToActOnBehalfOfOtherIdentity` attribute of the account of Service 2

![image info](../assets/Pasted%20image%2020250715135135.png)

1. Client uses ST to access SERVER1.
2. SERVER1 uses S4U2Proxy to request a ST for SERVER2 on behalf of the client.
3. DC checks weather SERVER2 Trusts SERVER1 for delegation or not by reading the `msDS-AllowedToActOnBehalfOfOtherIdentity` attribute of SERVER2
4. if not the request is denied
5. if SERVER2 Trusts SERVER1, DC responds with a client's ST for SERVER2.
6. Now SERVER1 uses this ST to access SERVER2 on behalf of the client.

## Abuse

##### **You Have `GenericWrite` (or Equivalent) Over a Service/Computer**

1. **Create or control a computer account** (via `ms-DS-MachineAccountQuota` or compromise).
2. **Modify the target service's/computer's `msDS-AllowedToActOnBehalfOfOtherIdentity`** to allow your computer account to delegate to it.
3. **Perform S4U2Self + S4U2Proxy** to impersonate an admin to the target service.


#### Exploitaion

1. User jack has GenericWrite access over the Domain Controller Computer Object.(DC01)
![image info](../assets/Pasted%20image%2020250721163827.png)
2. Creating a Computer account FAKE$ to add this FAKE$ Computer into  `msDS-AllowedToActOnBehalfOfOtherIdentity` attribute of Domain Controller Object.(DC01)

![image info](../assets/Pasted%20image%2020250721164234.png)
3. Reading the  `msDS-AllowedToActOnBehalfOfOtherIdentity` attribute of DC (DC01)

![image info](../assets/Pasted%20image%2020250721164313.png)
4. Adding FAKE$ Computer into `msDS-AllowedToActOnBehalfOfOtherIdentity` attribute of DC and verifying it.

![image info](../assets/Pasted%20image%2020250721164342.png)
5. Using impacket to impersonate administrator on DC01 by invoking S4U2Self and S4U2Proxy.

![image info](../assets/Pasted%20image%2020250721164812.png)
6. Importing the administrator ticket and getting shell as administrator on DC01

![image info](../assets/Pasted%20image%2020250721165612.png)

