#Encryption

##Outline

At the beginning of each connection, both nodes will establish a shared secret for encrypting the rest of the communications.

To leak no information about the nodes, we will use only ephermal keys to establish the shared secret.

##Specifics

###Handshake

After the connection was established, both nodes generate a secp256k1 keypair. They send the compressed 33byte public key to the other party. Both parties then calculate a common shared secret masterKey using the ECDH technique.

Using the shared secret, they further derive further keys:

`encryptionKey = SHA256(masterKey || 0)`

`hmacKey = SHA256(masterKey || 1)`

`iv1 = SHA256(masterKey || pubkey1)`

`iv2 = SHA256(masterKey || pubkey2)`

With index 1 being our pubkey and communications in our directions, and index 2 being the pubkey of the other party and communications in their direction respectively.

###Encryption

We use AES-128-CTR to encrypt all following communications. The encrypted data is then authenticated using a HMAC technique with the derived key.

The total packet will then be

`HMAC || N || PAYLOAD`

with

`N` being the total data sent so far, including this packet.
