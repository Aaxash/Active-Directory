
# Targeted Kerberoasting

Targeted Kerberoasting is an attack technique that leverages an attacker’s existing access to an account with specific permissions to make other accounts vulnerable to traditional Kerberoasting attacks.

- The key to this attack is exploiting the rights of an account that can modify Service Principal Names (SPNs) of other accounts in an Active Directory (AD) environment.
- In a targeted Kerberoasting attack, an attacker first gains control over an account with either  GenericAll, GenericWrite, WriteProperty or Validated-SPN permissions. 
- These permissions allow the attacker to modify the attributes of other accounts, including the addition or modification of SPNs. Once the attacker has control over such an account, they can add or modify an SPN on a different, typically high-value, account, effectively setting it up for a Kerberoasting attack.
- After adding or modifying the SPN, the attacker can then request a Service Ticket (ST) for the targeted account’s SPN. This ST is encrypted using the targeted account’s password hash, making it susceptible to offline brute-force attacks.

The exploitation process in Targeted Kerberoasting is as follows:

1. Gain control over an account with GenericAll, GenericWrite, WriteProperty or Validated-SPN permissions.
2. Use the compromised account to add or modify an SPN on a high-value target account.
3. Request a Service Ticket (ST) for the targeted account’s SPN.
4. Attempt to crack the encrypted ST offline using brute-force methods.



## Exploitation

- SQLSRV has generic write access on jack.

![image info](../assets/Pasted%20image%2020250721122403.png)

![image info](../assets/Pasted%20image%2020250721122420.png)

- SQLSRV can add an SPN on jack account, request a Service Ticket for jack account and then remove the SPN from jack account.

![image info](../assets/Pasted%20image%2020250721122618.png)

- And try cracking the Service Ticket to recover the jack password.

![image info](../assets/Pasted%20image%2020250721122956.png)

A tool to automate this is targetedKerberoast.py https://github.com/ShutdownRepo/targetedKerberoast

