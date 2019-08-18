# qTrade Original Development Guidelines

## Introduction

An exchange has several specific needs that are different than that of a regular end user or businesses looking to accept crypto as payment. We'll attempt to clearly outline our system, technical requirements/desires, and offer guidance on how to make integration with our exchange easier.

## Our System (in brief)

qTrade has 4 primary components:

 - Website: `frontend`
 - Trade engine & DB: `backend`
 - Currency network monitor: `watcher`
 - Isolated currency signing system: `signer`

It is important to understand that the `signer`, which hold private keys, is heavily isolated. It has no open ports, has minimal third party software, and only communicates with the backend. Holding secret keys in an isolated system provides many security benefits, which we won't go into here.

A typical withdraw flow looks something like this:

 1. `watcher` has (only) public keys and is tracking available (UTXOs or Account) balances 
 2. `frontend` user requests withdraw
 3. `backend` creates a withdraw request object
 4. `watcher` grabs the withdraw request and assembles the information required to create and/or sign a transaction and pushes it back to the `backend`
 5. `signer` grabs the withdraw info, signs it, and pushes it back to the `backend`
 6. `watcher` grabs the signed transaction and broadcasts it to the network
 7. `watcher` confirms the transaction once the network considers it irreversible

### Takeaways

 - Node software isn't run in the `signer`, so the method for signing data (transactions or other) should not require a node. Ideally signing operations could be handled in a standalone script or library, preferably with GoLang or Python examples. Many teams build this into their `wallet` software. Clearly documented and readable source code goes a long way here.
 - The `watcher` needs to not have private keys, but does need to track network state (and balances). Typically this means using the Node/API to track pubkeys. See the `Network monitoring` section below

 More info about our `signer` and `watcher` can be found here: [Systems Overview](https://github.com/qtrade-exchange/integration-tools/blob/master/qTradeSystemsOverview.md)

## Operational Considerations

### Key generation and handling

Recall that one of our system goals is to run as little 3rd party software as possible in the `signer`, and only the `signer` has access to secret keys. This means a straightforward way to generate new key pairs is very valuable, ideally a well documented method that doesn't require a running `node` or `wallet`.

### Network monitoring

Our `watcher` node needs to be able to identify incoming deposits, assess their finality, assemble information to create transactions. One way of accomplishing these tasks is `wallet` software which tracks transactions involving specific pubkeys. Bitcoin accomplishes this with `importaddress`. Our `watcher` can then use that RPC call to load the node software to handle deposit detection and alerting. 

It isn't necessary that your currency follow this paradigm, but there should be API endpoints we can use to figure out the required information.

Note: If you have HD key support, RPC functionality to handle address and subaddress generation can be very useful - even if just as a guide to write our own functions. If using ECDSA, ideally your import function would accept raw compressed ECDSA keys, but other formats can be made to work.

### Deposit scanning

Ideally we would be able to lookup all transactions concerning a certain account or address, preferably in an incremental or progressive fashion. Think something like Bitcoin's `importaddress` functionality and `listsinceblock`. Just getting the current balance for a given address is insufficient, because we must be able to discover/detect individual transactions. Thus, some sort of unique ID should be linked to a transaction.

Transactions need to be queryable in the future, and the node should be able to tell us whether they are irrevocable on the network ('confirmed' or not). Similarly, we need to be able to query back an arbitrary amount of time/blocks. If the node goes offline for several hours we must be able to locate deposits that came in during that time.

It should be possible to lookup information for addresses for which we do not have the private key, since our `watcher` software does not have access to the private keys, but does need the balance information.

In the worst case we can scan the blockchain ourselves, in which case we need an easy way to get a decoded, easily parse-able list of all transactions for a given block height or hash, and then we can do the transaction indexing ourselves by scanning the blockchain.

### Deposit correlation

Payment processors, exchanges, and other third parties need a way to associate which transactions are associated with a specific user. The classic system for this is tracking incoming deposits by assigning each user a unique public key.

We'd recommend the more modern version: Allow including some small amount of data in a transaction, which can be used as a user correlative.

There are other methods, but whatever method is used it should ensure that only a few root/seed keys need to be secured. Encrypted paper backups and [Shamir's Secret Sharing](https://en.wikipedia.org/wiki/Shamir%27s_Secret_Sharing) makes backing up hundreds of keys impractical.

In account based systems that do not support arbitrary identifiers, like Ethereum and Nano, we perform an additional "sweep" step to move all deposits to a central wallet account. This process adds operational and integration complexity, and ideally would be avoided.

### Transaction Lookup

Provided the TXID we need to be able to lookup if that transaction is now safe from a double spend. For Bitcoin-like blockchains this means confirmations, but any suitable status information can be used. We should have strong guarantees that the transaction is irreversible, and guidance on when it can be considered as such.

### Network Health

Ideally we would have easy access to statistics that give us insight into our nodes health, and the health of the p2p network. For example for Bitcoin we monitor block height, current difficulty, and connected peer count.

### Persistence

Blockchain data's storage location should be configurable, such as with the `-datadir` argument on Bitcoin. If not configurable, it should at least be self-contained in a single folder. Having 6 different folders that get created in the same folder as the binary presents a big headache for putting data directories on separate disks from the executables, which is frequently desirable in cloud environments.

### Push notifications

Push is optional, although nice to have. Anything that allows us send an HTTP request of some kind from the node to our software, similar to `walletnotify` + `curl` in Bitcoin core, should work well.

## Code Quality and Documentation

As a general rule we like to see the following in currencies we're considering:

* Clear, reproducible, up to date build instructions for most major platforms (Linux, OSX, Windows). We will be deploying on Linux.
* Consistent, established, and preferably codified coding style/conventions across the code base. Logical variable naming, easy to read flow, good commenting to explain complex parts.
* A well documented API
* Examples of multi-language integration, or use of standard protocols for easy (non-library) integration. Using Protobuf instead of custom wire protocol. JSON over HTTP is always good.

## Digital Signature Algorithm (DSA)

We'd recommend supporting Ed25519 or ECDSA signatures on the secp256k1 curve. Since this is what is widely available, easy to use, and battle tested.

If you're making your own cryptocurrency you probably have your own ideas about what DSA you'd like to use, and generally speaking we can work with anything that we're confident is cryptographically secure. If you want to use a DSA other than the above recommended ones, we encourage you to select a well established and widely trusted algorithm. Broad language support, especially Python or GoLang is a plus because it makes our (and other 3rd party's) integration more secure and straightforward. 

The DSA you choose is important for supporting hardware wallets and cold storage as well, since using YubiKey or Trezor hardware devices becomes easier when using widely adopted standards. 

If one of your features is quantum resistance and requires a custom or little used DSA please be aware that this is one of the areas that can take considerable integration time and expense. It is therefore much more important that excellent docs are provided for generating addresses, address handling, and signing operations. Ideally with scripts or examples in GoLang or Python. 

## Containerization

As anyone who has run more than a half dozen cryptocurrencies at once can tell you, it quickly becomes an ongoing maintenance and security challenge. One of the ways we handle this is by containerizing the node, wallet, and RPC software. This provides us a lot of tools for managing deploying and scaling this said software, as well as an isolated environment from which that software has limited access to other systems.

We have a specific method we follow for building our container images, and if you're able to provide us an image that meets our needs that can save us a lot of time, and is likely useful to many others who want to run the software.

An outline of our image requirements can be found here: [Containerization Instructions](https://github.com/qtrade-exchange/integration-tools/blob/master/docker/Instructions.md)