# Understand Crypto Keys in Wallet

We will explore how a crypto wallet works in general, and find out if our money are truly safe. Did you hear about mnemonic, derivation paths, private and public Key? 

# Crypto wallet components

![Screen Shot 2022-10-10 at 5.37.39 PM-min.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1665398276611/kbkCeYfiB.png align="left")

Metamask example:

![Screen Shot 2022-10-10 at 5.38.32 PM-min.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1665398332757/eo_XMGa6y.png align="left")

# Key managements

![Screen Shot 2022-10-10 at 5.39.51 PM-min.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1665398423101/b9uLVsoUM.png align="left")

+ Mnemonic ([BIP39](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki)): a group of words, easy to remember, usually 12 or more that are generated when a new crypto wallet is created. It can be used to restore accounts.
+ Passphrase ([BIP39](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki)): an optional feature that can be used in addition to your mnemonic to restrict access to your crypto wallet. It offer extra security in the event that your mnemonic is exposed. It's like two-factor authentication.
+ Seed: series of bytes that generated after combining the mnemonic and passphrase.
+ Derivation path ([BIP32](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki)): A derivation path is a piece of data which tells a Hierarchical Deterministic (HD) wallet how to derive a specific key within a tree of keys. Example: `m/44'/60'/0'/0'/0` for private key 1 and `m/44'/60'/0'/0'/1` for for private key 2.
+ Private keys: a secret number similar to a password. They are used to sign transactions and prove ownership of a blockchain address.
+ Encryption scheme: is used to define the algorithm when encrypting transactions or messages.
+ Public keys: aka addresses. Each blockchain has a different format.

Derivation path is usually managed by wallets, and encryption scheme is specified by blockchains. As a normal user, we should pay attention to passphrase, mnemonic and private keys.

# Passphrase

![Screen Shot 2022-10-11 at 11.41.34 AM.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1665463324792/NfW3Vu2id.png align="left")

Your passphrase should be short enough to remember it in your mind only. If hackers have mnemonic, they cannot generate the real accounts without the right passphrase.

# Mnemonic and Private Keys

![Screen Shot 2022-10-11 at 11.41.43 AM.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1665463345564/Tm8XOR6EO.png align="left")

A wallet can import both mnemonic and private keys. After importing:

+ Software wallet will store them in your device memory, not in any third party servers.
+ Hardware wallet will store in chip. It is safer and cannot be hacked without physical access to the device.

That's all. I hope this post gives you an overview. Thank you for reading my blog.


