# TLS Certificates

Certificates gives the trust between the sender and receiver while transactions.

Whitout TLS

User were to access his online banking application, the cred types in would be sent in plain text. The hacker sniffing network traffic could easily retrieve the creds and use it to hack the bank account.

So we must need to implement encrypt the data being transferred using encryption keys. 

We encrypt the data using a key, and the same key is used by the server to decrypt the data. When the same key is used for both encryption and decryption, it is called symmetric encryption.

The problem is that we also need to send the key to the server over the network. If someone intercepts the key, they can decrypt the data, so this method is not completely secure for sending the key over an untrusted network.

So for that reason asymmatric encryption comes in the picture.
It uses the pair of keys - private key and public key
for study i am going to use private key and public lock
key is private like its only with me and lock is public as any one can access it.



